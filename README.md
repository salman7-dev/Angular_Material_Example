package com.clientledger.core.service.order

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.Order
import com.clientledger.core.domain.OrderItem
import com.clientledger.core.domain.OrderStatus
import com.clientledger.core.domain.OrderUnit
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.lock.OwnerOperationLockRepository
import com.clientledger.core.repository.maintenance.MaintenanceRepository
import com.clientledger.core.repository.order.OrderRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.service.client.ClientEditWindowService
import com.clientledger.core.service.client.ClientService
import com.clientledger.core.service.lock.OwnerOperationLockService
import com.clientledger.core.transaction.FirestoreTransactionExecutor
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles
import java.time.Clock
import java.time.Instant
import java.time.ZoneId

@SpringBootTest
@ActiveProfiles("test")
class OrderServiceTest {

    @Autowired
    private lateinit var properties: ClientLedgerProperties

    @Autowired
    private lateinit var ownerOperationLockService: OwnerOperationLockService

    companion object {

        private lateinit var firestore: Firestore

        private val testClock = Clock.fixed(
            Instant.parse("2026-09-24T10:00:00Z"),
            ZoneId.of("UTC")
        )

        @JvmStatic
        @BeforeAll
        fun setup() {

            firestore =
                FirestoreOptions.newBuilder()
                    .setProjectId("client-ledger-dashboard")
                    .setHost("127.0.0.1:8080")
                    .setEmulatorHost("127.0.0.1:8080")
                    .setCredentials(NoCredentials.getInstance())
                    .build()
                    .service
        }

        @JvmStatic
        @AfterAll
        fun cleanup() {

            if (::firestore.isInitialized) {
                firestore.close()
            }
        }
    }

    @Test
    fun createOrderPersistsCalculatedFinancialValues() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Order Client",
                    phone = "9000000001",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val order =
            Order(
                ownerId = ownerId,
                clientId = client.id,
                orderDate = "2026-08-20",
                items = listOf(
                    OrderItem(
                        name = "Product A",
                        quantity = 1,
                        unit = OrderUnit.SINGLE,
                        amount = 5000,
                        gstRate = 18,
                        gstAmount = 900,
                        expense = 3000
                    )
                )
            )

        val orderService =
            createOrderService()

        val createdOrder =
            orderService.create(order)

        assertNotNull(createdOrder.id)

        assertEquals(
            true,
            createdOrder.id.startsWith("ORD-")
        )

        assertEquals(
            5000,
            createdOrder.totalInvoiceAmount
        )

        assertEquals(
            5900,
            createdOrder.totalInvoiceAmountWithGst
        )

        assertEquals(
            900,
            createdOrder.totalGstAmount
        )

        assertEquals(
            3000,
            createdOrder.totalExpense
        )

        assertEquals(
            2000,
            createdOrder.profitAmount
        )

        assertEquals(
            OrderStatus.CREATED,
            createdOrder.status
        )

        val orderRepository =
            OrderRepository(firestore)

        val storedOrder =
            orderRepository.findById(
                ownerId = ownerId,
                orderDate = "2026-08-20",
                orderId = createdOrder.id
            )

        assertNotNull(storedOrder)

        assertEquals(
            createdOrder.id,
            storedOrder!!.id
        )

        assertEquals(
            ownerId,
            storedOrder.ownerId
        )

        assertEquals(
            client.id,
            storedOrder.clientId
        )

        assertEquals(
            "2026-08-20",
            storedOrder.orderDate
        )

        assertEquals(
            5000,
            storedOrder.totalInvoiceAmount
        )

        assertEquals(
            5900,
            storedOrder.totalInvoiceAmountWithGst
        )

        assertEquals(
            900,
            storedOrder.totalGstAmount
        )

        assertEquals(
            3000,
            storedOrder.totalExpense
        )

        assertEquals(
            2000,
            storedOrder.profitAmount
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val augustHistory =
            historyRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-08",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(augustHistory)

        assertEquals(
            5000,
            augustHistory!!.openingBalance
        )

        assertEquals(
            5000,
            augustHistory.totalInvoiceAmount
        )

        assertEquals(
            5900,
            augustHistory.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            augustHistory.invoiceCount
        )

        assertEquals(
            900,
            augustHistory.totalGstAmount
        )

        assertEquals(
            3000,
            augustHistory.totalExpenses
        )

        assertEquals(
            10900,
            augustHistory.closingBalance
        )

        assertEquals(
            10900,
            augustHistory.receivable
        )

        assertEquals(
            0,
            augustHistory.advance
        )

        assertEquals(
            ClientType.RECEIVABLE,
            augustHistory.status
        )

        val septemberHistory =
            historyRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-09",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(septemberHistory)

        assertEquals(
            10900,
            septemberHistory!!.openingBalance
        )

        assertEquals(
            0,
            septemberHistory.totalInvoiceAmount
        )

        assertEquals(
            0,
            septemberHistory.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            septemberHistory.invoiceCount
        )

        assertEquals(
            0,
            septemberHistory.totalGstAmount
        )

        assertEquals(
            0,
            septemberHistory.totalExpenses
        )

        assertEquals(
            10900,
            septemberHistory.closingBalance
        )

        assertEquals(
            10900,
            septemberHistory.receivable
        )

        assertEquals(
            0,
            septemberHistory.advance
        )

        assertEquals(
            ClientType.RECEIVABLE,
            septemberHistory.status
        )
    }

    @Test
    fun createOrderUpdatesSummaryClientIndexUsingClientBucket() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Summary Index Client",
                    phone = "9000000002",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val order =
            Order(
                ownerId = ownerId,
                clientId = client.id,
                orderDate = "2026-08-20",
                items = listOf(
                    OrderItem(
                        name = "Product A",
                        quantity = 1,
                        unit = OrderUnit.SINGLE,
                        amount = 5000,
                        gstRate = 18,
                        gstAmount = 900,
                        expense = 3000
                    )
                )
            )

        val orderService =
            createOrderService()

        orderService.create(order)

        val summaryIndexRepository =
            SummaryClientIndexRepository(firestore)

        val augustIndex =
            summaryIndexRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-08",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(augustIndex)

        assertEquals(
            client.id,
            augustIndex!!.clientId
        )

        assertEquals(
            10900,
            augustIndex.amount
        )

        assertEquals(
            ClientType.RECEIVABLE,
            augustIndex.status
        )

        val septemberIndex =
            summaryIndexRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-09",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(septemberIndex)

        assertEquals(
            10900,
            septemberIndex!!.amount
        )

        assertEquals(
            ClientType.RECEIVABLE,
            septemberIndex.status
        )
    }

    @Test
    fun updateOrderChangesLedgerForNormalEdit() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Normal Edit Client",
                    phone = "9000000003",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = client.id,
                    orderDate = "2026-08-20",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            unit = OrderUnit.SINGLE,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            unit = OrderUnit.SINGLE,
                            amount = 7000,
                            gstRate = 18,
                            gstAmount = 1260,
                            expense = 4000
                        )
                    )
                )
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            client.id,
            updatedOrder.clientId
        )

        assertEquals(
            "2026-08-20",
            updatedOrder.orderDate
        )

        assertEquals(
            7000,
            updatedOrder.totalInvoiceAmount
        )

        assertEquals(
            8260,
            updatedOrder.totalInvoiceAmountWithGst
        )

        assertEquals(
            1260,
            updatedOrder.totalGstAmount
        )

        assertEquals(
            4000,
            updatedOrder.totalExpense
        )

        assertEquals(
            3000,
            updatedOrder.profitAmount
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val augustHistory =
            historyRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-08",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(augustHistory)

        assertEquals(
            5000,
            augustHistory!!.openingBalance
        )

        assertEquals(
            7000,
            augustHistory.totalInvoiceAmount
        )

        assertEquals(
            8260,
            augustHistory.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            augustHistory.invoiceCount
        )

        assertEquals(
            1260,
            augustHistory.totalGstAmount
        )

        assertEquals(
            4000,
            augustHistory.totalExpenses
        )

        assertEquals(
            13260,
            augustHistory.closingBalance
        )

        assertEquals(
            13260,
            augustHistory.receivable
        )

        assertEquals(
            0,
            augustHistory.advance
        )

        assertEquals(
            ClientType.RECEIVABLE,
            augustHistory.status
        )

        val septemberHistory =
            historyRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-09",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(septemberHistory)

        assertEquals(
            13260,
            septemberHistory!!.openingBalance
        )

        assertEquals(
            0,
            septemberHistory.totalInvoiceAmount
        )

        assertEquals(
            0,
            septemberHistory.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            septemberHistory.invoiceCount
        )

        assertEquals(
            0,
            septemberHistory.totalGstAmount
        )

        assertEquals(
            0,
            septemberHistory.totalExpenses
        )

        assertEquals(
            13260,
            septemberHistory.closingBalance
        )

        assertEquals(
            13260,
            septemberHistory.receivable
        )

        assertEquals(
            0,
            septemberHistory.advance
        )

        assertEquals(
            ClientType.RECEIVABLE,
            septemberHistory.status
        )

        val summaryIndexRepository =
            SummaryClientIndexRepository(firestore)

        val augustIndex =
            summaryIndexRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-08",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(augustIndex)

        assertEquals(
            13260,
            augustIndex!!.amount
        )

        assertEquals(
            ClientType.RECEIVABLE,
            augustIndex.status
        )

        val septemberIndex =
            summaryIndexRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-09",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(septemberIndex)

        assertEquals(
            13260,
            septemberIndex!!.amount
        )

        assertEquals(
            ClientType.RECEIVABLE,
            septemberIndex.status
        )
    }

    @Test
    fun updateOrderMovesOrderFromPastMonthToLaterMonth() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Date Change Forward Client",
                    phone = "9000000004",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = client.id,
                    orderDate = "2026-08-20",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            unit = OrderUnit.SINGLE,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    orderDate = "2026-09-10"
                )
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            client.id,
            updatedOrder.clientId
        )

        assertEquals(
            "2026-09-10",
            updatedOrder.orderDate
        )

        assertEquals(
            5000,
            updatedOrder.totalInvoiceAmount
        )

        assertEquals(
            5900,
            updatedOrder.totalInvoiceAmountWithGst
        )

        assertEquals(
            900,
            updatedOrder.totalGstAmount
        )

        assertEquals(
            3000,
            updatedOrder.totalExpense
        )

        assertEquals(
            2000,
            updatedOrder.profitAmount
        )

        val orderRepository =
            OrderRepository(firestore)

        val oldOrder =
            orderRepository.findById(
                ownerId = ownerId,
                orderDate = "2026-08-20",
                orderId = originalOrder.id
            )

        assertEquals(
            null,
            oldOrder
        )

        val newOrder =
            orderRepository.findById(
                ownerId = ownerId,
                orderDate = "2026-09-10",
                orderId = originalOrder.id
            )

        assertNotNull(newOrder)

        assertEquals(
            originalOrder.id,
            newOrder!!.id
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val augustHistory =
            historyRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-08",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(augustHistory)

        assertEquals(
            5000,
            augustHistory!!.openingBalance
        )

        assertEquals(
            0,
            augustHistory.totalInvoiceAmount
        )

        assertEquals(
            0,
            augustHistory.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            augustHistory.invoiceCount
        )

        assertEquals(
            0,
            augustHistory.totalGstAmount
        )

        assertEquals(
            0,
            augustHistory.totalExpenses
        )

        assertEquals(
            5000,
            augustHistory.closingBalance
        )

        assertEquals(
            5000,
            augustHistory.receivable
        )

        assertEquals(
            0,
            augustHistory.advance
        )

        assertEquals(
            ClientType.RECEIVABLE,
            augustHistory.status
        )

        val septemberHistory =
            historyRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-09",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(septemberHistory)

        assertEquals(
            5000,
            septemberHistory!!.openingBalance
        )

        assertEquals(
            5000,
            septemberHistory.totalInvoiceAmount
        )

        assertEquals(
            5900,
            septemberHistory.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            septemberHistory.invoiceCount
        )

        assertEquals(
            900,
            septemberHistory.totalGstAmount
        )

        assertEquals(
            3000,
            septemberHistory.totalExpenses
        )

        assertEquals(
            10900,
            septemberHistory.closingBalance
        )

        assertEquals(
            10900,
            septemberHistory.receivable
        )

        assertEquals(
            0,
            septemberHistory.advance
        )

        assertEquals(
            ClientType.RECEIVABLE,
            septemberHistory.status
        )

        val summaryIndexRepository =
            SummaryClientIndexRepository(firestore)

        val augustIndex =
            summaryIndexRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-08",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(augustIndex)

        assertEquals(
            5000,
            augustIndex!!.amount
        )

        assertEquals(
            ClientType.RECEIVABLE,
            augustIndex.status
        )

        val septemberIndex =
            summaryIndexRepository.find(
                ownerId = ownerId,
                yearMonth = "2026-09",
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(septemberIndex)

        assertEquals(
            10900,
            septemberIndex!!.amount
        )

        assertEquals(
            ClientType.RECEIVABLE,
            septemberIndex.status
        )
    }

    @Test
    fun updateOrderMovesOrderFromLaterMonthToPastMonth() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Date Backward Client",
                    phone = "9000000011",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = client.id,
                    orderDate = "2026-09-10",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    orderDate = "2026-08-20"
                )
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            client.id,
            updatedOrder.clientId
        )

        assertEquals(
            "2026-08-20",
            updatedOrder.orderDate
        )

        assertEquals(
            5000,
            updatedOrder.totalInvoiceAmount
        )

        assertEquals(
            5900,
            updatedOrder.totalInvoiceAmountWithGst
        )

        val orderRepository =
            OrderRepository(firestore)

        assertEquals(
            null,
            orderRepository.findById(
                ownerId,
                "2026-09-10",
                originalOrder.id
            )
        )

        val storedOrder =
            orderRepository.findById(
                ownerId,
                "2026-08-20",
                originalOrder.id
            )

        assertNotNull(storedOrder)

        assertEquals(
            originalOrder.id,
            storedOrder!!.id
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val august =
            historyRepository.find(
                ownerId,
                "2026-08",
                client.bucketId,
                client.id
            )

        val september =
            historyRepository.find(
                ownerId,
                "2026-09",
                client.bucketId,
                client.id
            )

        assertNotNull(august)
        assertNotNull(september)

        assertEquals(
            5000,
            august!!.openingBalance
        )

        assertEquals(
            5000,
            august.totalInvoiceAmount
        )

        assertEquals(
            5900,
            august.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            august.invoiceCount
        )

        assertEquals(
            10900,
            august.closingBalance
        )

        assertEquals(
            10900,
            september!!.openingBalance
        )

        assertEquals(
            0,
            september.totalInvoiceAmount
        )

        assertEquals(
            0,
            september.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            september.invoiceCount
        )

        assertEquals(
            10900,
            september.closingBalance
        )
    }

    @Test
    fun updateOrderChangesDateWithinSameMonth() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Same Month Client",
                    phone = "9000000012",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = client.id,
                    orderDate = "2026-08-05",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    orderDate = "2026-08-25"
                )
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            "2026-08-25",
            updatedOrder.orderDate
        )

        val orderRepository =
            OrderRepository(firestore)

        val storedOrder =
            orderRepository.findById(
                ownerId = ownerId,
                orderDate = "2026-08-25",
                orderId = originalOrder.id
            )

        assertNotNull(storedOrder)

        assertEquals(
            originalOrder.id,
            storedOrder!!.id
        )

        assertEquals(
            "2026-08-25",
            storedOrder.orderDate
        )

        assertEquals(
            client.id,
            storedOrder.clientId
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val august =
            historyRepository.find(
                ownerId,
                "2026-08",
                client.bucketId,
                client.id
            )

        assertNotNull(august)

        assertEquals(
            5000,
            august!!.openingBalance
        )

        assertEquals(
            5000,
            august.totalInvoiceAmount
        )

        assertEquals(
            5900,
            august.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            august.invoiceCount
        )

        assertEquals(
            10900,
            august.closingBalance
        )
    }

    @Test
    fun updateOrderChangesDateAndFinancialValuesForSameClient() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Date Financial Client",
                    phone = "9000000013",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = client.id,
                    orderDate = "2026-08-20",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    orderDate = "2026-09-10",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 7000,
                            gstRate = 18,
                            gstAmount = 1260,
                            expense = 4000
                        )
                    )
                )
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            client.id,
            updatedOrder.clientId
        )

        assertEquals(
            "2026-09-10",
            updatedOrder.orderDate
        )

        assertEquals(
            7000,
            updatedOrder.totalInvoiceAmount
        )

        assertEquals(
            8260,
            updatedOrder.totalInvoiceAmountWithGst
        )

        assertEquals(
            1260,
            updatedOrder.totalGstAmount
        )

        assertEquals(
            4000,
            updatedOrder.totalExpense
        )

        assertEquals(
            3000,
            updatedOrder.profitAmount
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val august =
            historyRepository.find(
                ownerId,
                "2026-08",
                client.bucketId,
                client.id
            )

        val september =
            historyRepository.find(
                ownerId,
                "2026-09",
                client.bucketId,
                client.id
            )

        assertNotNull(august)
        assertNotNull(september)

        assertEquals(
            5000,
            august!!.openingBalance
        )

        assertEquals(
            0,
            august.totalInvoiceAmount
        )

        assertEquals(
            0,
            august.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            august.invoiceCount
        )

        assertEquals(
            5000,
            august.closingBalance
        )

        assertEquals(
            5000,
            september!!.openingBalance
        )

        assertEquals(
            7000,
            september.totalInvoiceAmount
        )

        assertEquals(
            8260,
            september.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            september.invoiceCount
        )

        assertEquals(
            13260,
            september.closingBalance
        )
    }

    @Test
    fun updateOrderChangesClientForSameDate() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val oldClient =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Old Client",
                    phone = "9000000014",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val newClient =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "New Client",
                    phone = "9000000015",
                    initialOpeningBalance = 2000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = oldClient.id,
                    orderDate = "2026-08-20",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    clientId = newClient.id
                )
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            newClient.id,
            updatedOrder.clientId
        )

        assertEquals(
            "2026-08-20",
            updatedOrder.orderDate
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val oldAugust =
            historyRepository.find(
                ownerId,
                "2026-08",
                oldClient.bucketId,
                oldClient.id
            )

        val newAugust =
            historyRepository.find(
                ownerId,
                "2026-08",
                newClient.bucketId,
                newClient.id
            )

        assertNotNull(oldAugust)
        assertNotNull(newAugust)

        assertEquals(
            5000,
            oldAugust!!.openingBalance
        )

        assertEquals(
            0,
            oldAugust.totalInvoiceAmount
        )

        assertEquals(
            0,
            oldAugust.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            oldAugust.invoiceCount
        )

        assertEquals(
            5000,
            oldAugust.closingBalance
        )

        assertEquals(
            2000,
            newAugust!!.openingBalance
        )

        assertEquals(
            5000,
            newAugust.totalInvoiceAmount
        )

        assertEquals(
            5900,
            newAugust.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            newAugust.invoiceCount
        )

        assertEquals(
            7900,
            newAugust.closingBalance
        )

        val orderRepository =
            OrderRepository(firestore)

        val storedOrder =
            orderRepository.findById(
                ownerId,
                "2026-08-20",
                originalOrder.id
            )

        assertNotNull(storedOrder)

        assertEquals(
            originalOrder.id,
            storedOrder!!.id
        )

        assertEquals(
            newClient.id,
            storedOrder.clientId
        )
    }

    @Test
    fun updateOrderChangesClientAndFinancialValuesForSameDate() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val oldClient =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Old Financial Client",
                    phone = "9000000016",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val newClient =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "New Financial Client",
                    phone = "9000000017",
                    initialOpeningBalance = 2000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = oldClient.id,
                    orderDate = "2026-08-20",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    clientId = newClient.id,
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 7000,
                            gstRate = 18,
                            gstAmount = 1260,
                            expense = 4000
                        )
                    )
                )
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            newClient.id,
            updatedOrder.clientId
        )

        assertEquals(
            7000,
            updatedOrder.totalInvoiceAmount
        )

        assertEquals(
            8260,
            updatedOrder.totalInvoiceAmountWithGst
        )

        assertEquals(
            1260,
            updatedOrder.totalGstAmount
        )

        assertEquals(
            4000,
            updatedOrder.totalExpense
        )

        assertEquals(
            3000,
            updatedOrder.profitAmount
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val oldAugust =
            historyRepository.find(
                ownerId,
                "2026-08",
                oldClient.bucketId,
                oldClient.id
            )

        val newAugust =
            historyRepository.find(
                ownerId,
                "2026-08",
                newClient.bucketId,
                newClient.id
            )

        assertNotNull(oldAugust)
        assertNotNull(newAugust)

        assertEquals(
            5000,
            oldAugust!!.openingBalance
        )

        assertEquals(
            0,
            oldAugust.totalInvoiceAmount
        )

        assertEquals(
            0,
            oldAugust.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            oldAugust.invoiceCount
        )

        assertEquals(
            5000,
            oldAugust.closingBalance
        )

        assertEquals(
            2000,
            newAugust!!.openingBalance
        )

        assertEquals(
            7000,
            newAugust.totalInvoiceAmount
        )

        assertEquals(
            8260,
            newAugust.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            newAugust.invoiceCount
        )

        assertEquals(
            10260,
            newAugust.closingBalance
        )
    }

    @Test
    fun updateOrderChangesClientAndDate() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val oldClient =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Old Full Change Client",
                    phone = "9000000018",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val newClient =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "New Full Change Client",
                    phone = "9000000019",
                    initialOpeningBalance = 2000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = oldClient.id,
                    orderDate = "2026-08-20",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    clientId = newClient.id,
                    orderDate = "2026-09-10",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 7000,
                            gstRate = 18,
                            gstAmount = 1260,
                            expense = 4000
                        )
                    )
                )
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            newClient.id,
            updatedOrder.clientId
        )

        assertEquals(
            "2026-09-10",
            updatedOrder.orderDate
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val oldAugust =
            historyRepository.find(
                ownerId,
                "2026-08",
                oldClient.bucketId,
                oldClient.id
            )

        val oldSeptember =
            historyRepository.find(
                ownerId,
                "2026-09",
                oldClient.bucketId,
                oldClient.id
            )

        val newSeptember =
            historyRepository.find(
                ownerId,
                "2026-09",
                newClient.bucketId,
                newClient.id
            )

        assertNotNull(oldAugust)
        assertNotNull(oldSeptember)
        assertNotNull(newSeptember)

        assertEquals(
            5000,
            oldAugust!!.openingBalance
        )

        assertEquals(
            0,
            oldAugust.totalInvoiceAmount
        )

        assertEquals(
            0,
            oldAugust.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            oldAugust.invoiceCount
        )

        assertEquals(
            5000,
            oldAugust.closingBalance
        )

        assertEquals(
            5000,
            oldSeptember!!.openingBalance
        )

        assertEquals(
            0,
            oldSeptember.totalInvoiceAmount
        )

        assertEquals(
            0,
            oldSeptember.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            oldSeptember.invoiceCount
        )

        assertEquals(
            5000,
            oldSeptember.closingBalance
        )

        assertEquals(
            2000,
            newSeptember!!.openingBalance
        )

        assertEquals(
            7000,
            newSeptember.totalInvoiceAmount
        )

        assertEquals(
            8260,
            newSeptember.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            newSeptember.invoiceCount
        )

        assertEquals(
            10260,
            newSeptember.closingBalance
        )

        val orderRepository =
            OrderRepository(firestore)

        assertEquals(
            null,
            orderRepository.findById(
                ownerId,
                "2026-08-20",
                originalOrder.id
            )
        )

        val storedOrder =
            orderRepository.findById(
                ownerId,
                "2026-09-10",
                originalOrder.id
            )

        assertNotNull(storedOrder)

        assertEquals(
            originalOrder.id,
            storedOrder!!.id
        )

        assertEquals(
            newClient.id,
            storedOrder.clientId
        )
    }

    @Test
    fun updateOrderWithNoChangesDoesNotChangeLedgerValues() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "No Change Client",
                    phone = "9000000020",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = client.id,
                    orderDate = "2026-08-20",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder = originalOrder
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            originalOrder.clientId,
            updatedOrder.clientId
        )

        assertEquals(
            originalOrder.orderDate,
            updatedOrder.orderDate
        )

        assertEquals(
            originalOrder.totalInvoiceAmount,
            updatedOrder.totalInvoiceAmount
        )

        assertEquals(
            originalOrder.totalInvoiceAmountWithGst,
            updatedOrder.totalInvoiceAmountWithGst
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val august =
            historyRepository.find(
                ownerId,
                "2026-08",
                client.bucketId,
                client.id
            )

        assertNotNull(august)

        assertEquals(
            5000,
            august!!.openingBalance
        )

        assertEquals(
            5000,
            august.totalInvoiceAmount
        )

        assertEquals(
            5900,
            august.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            august.invoiceCount
        )

        assertEquals(
            10900,
            august.closingBalance
        )

        assertEquals(
            ClientType.RECEIVABLE,
            august.status
        )
    }

    @Test
    fun updateOrderFromCurrentMonthToPastMonthRecalculatesCurrentMonth() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Current To Past Client",
                    phone = "9000000021",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = client.id,
                    orderDate = "2026-09-10",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    orderDate = "2026-08-20"
                )
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            "2026-08-20",
            updatedOrder.orderDate
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val august =
            historyRepository.find(
                ownerId,
                "2026-08",
                client.bucketId,
                client.id
            )

        val september =
            historyRepository.find(
                ownerId,
                "2026-09",
                client.bucketId,
                client.id
            )

        assertNotNull(august)
        assertNotNull(september)

        assertEquals(
            5000,
            august!!.openingBalance
        )

        assertEquals(
            5000,
            august.totalInvoiceAmount
        )

        assertEquals(
            5900,
            august.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            august.invoiceCount
        )

        assertEquals(
            10900,
            august.closingBalance
        )

        assertEquals(
            10900,
            september!!.openingBalance
        )

        assertEquals(
            0,
            september.totalInvoiceAmount
        )

        assertEquals(
            0,
            september.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            september.invoiceCount
        )

        assertEquals(
            10900,
            september.closingBalance
        )
    }

    @Test
    fun updateOrderIntoCurrentMonthRecalculatesThroughCurrentMonth() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Into Current Client",
                    phone = "9000000022",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T07:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = client.id,
                    orderDate = "2026-08-20",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    orderDate = "2026-09-10"
                )
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            "2026-09-10",
            updatedOrder.orderDate
        )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val august =
            historyRepository.find(
                ownerId,
                "2026-08",
                client.bucketId,
                client.id
            )

        val september =
            historyRepository.find(
                ownerId,
                "2026-09",
                client.bucketId,
                client.id
            )

        assertNotNull(august)
        assertNotNull(september)

        assertEquals(
            5000,
            august!!.openingBalance
        )

        assertEquals(
            0,
            august.totalInvoiceAmount
        )

        assertEquals(
            0,
            august.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            august.invoiceCount
        )

        assertEquals(
            5000,
            august.closingBalance
        )

        assertEquals(
            5000,
            september!!.openingBalance
        )

        assertEquals(
            5000,
            september.totalInvoiceAmount
        )

        assertEquals(
            5900,
            september.totalInvoiceAmountWithGst
        )

        assertEquals(
            1,
            september.invoiceCount
        )

        assertEquals(
            10900,
            september.closingBalance
        )
    }

    @Test
    fun updateOrderFromCurrentMonthToPastMonthKeepsSameOrderId() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Order ID Client",
                    phone = "9000000023",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = client.id,
                    orderDate = "2026-09-10",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        val updatedOrder =
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    orderDate = "2026-08-20"
                )
            )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )

        assertEquals(
            originalOrder.id,
            updatedOrder.id
        )
    }

    @Test
    fun updateOrderRejectsFutureDate() {

        val clientService =
            createClientService()

        val ownerId =
            "owner-${System.nanoTime()}"

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Future Date Client",
                    phone = "9000000024",
                    initialOpeningBalance = 5000,
                    createdAt = Instant.parse("2026-07-01T10:00:00Z"),
                    updatedAt = Instant.parse("2026-07-01T10:00:00Z")
                )
            )

        val orderService =
            createOrderService()

        val originalOrder =
            orderService.create(
                Order(
                    ownerId = ownerId,
                    clientId = client.id,
                    orderDate = "2026-08-20",
                    items = listOf(
                        OrderItem(
                            name = "Product A",
                            quantity = 1,
                            amount = 5000,
                            gstRate = 18,
                            gstAmount = 900,
                            expense = 3000
                        )
                    )
                )
            )

        org.junit.jupiter.api.Assertions.assertThrows(
            RuntimeException::class.java
        ) {
            orderService.update(
                ownerId = ownerId,
                orderId = originalOrder.id,
                updatedOrder =
                originalOrder.copy(
                    orderDate = "2026-09-25"
                )
            )
        }
    }

    private fun createClientService(): ClientService {

        val transactionExecutor =
            FirestoreTransactionExecutor(firestore)

        val clientRepository =
            ClientRepository(firestore)

        val historyBucketRepository =
            ClientHistoryBucketRepository(
                firestore = firestore,
                properties = properties
            )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val globalSummaryRepository =
            GlobalSummaryRepository(firestore)

        val summaryIndexBucketRepository =
            SummaryClientIndexBucketRepository(
                firestore = firestore,
                properties = properties
            )

        val summaryIndexRepository =
            SummaryClientIndexRepository(firestore)

        val maintenanceRepository =
            MaintenanceRepository(firestore)

        val ownerOperationLockRepository =
            OwnerOperationLockRepository(
                firestore = firestore,
                maintenanceRepository = maintenanceRepository
            )

        return ClientService(
            firestore = firestore,
            transactionExecutor = transactionExecutor,
            clientRepository = clientRepository,
            historyBucketRepository = historyBucketRepository,
            historyRepository = historyRepository,
            globalSummaryRepository = globalSummaryRepository,
            summaryIndexBucketRepository = summaryIndexBucketRepository,
            summaryIndexRepository = summaryIndexRepository,
            ownerOperationLockRepository = ownerOperationLockRepository,
            properties = properties,
            clock = testClock,
            ownerOperationLockService = ownerOperationLockService
        )
    }

    private fun createOrderService(): OrderService {

        val transactionExecutor =
            FirestoreTransactionExecutor(firestore)

        val orderRepository =
            OrderRepository(firestore)

        val clientRepository =
            ClientRepository(firestore)

        val historyRepository =
            ClientHistoryRepository(firestore)

        val globalSummaryRepository =
            GlobalSummaryRepository(firestore)

        val summaryIndexBucketRepository =
            SummaryClientIndexBucketRepository(
                firestore = firestore,
                properties = properties
            )

        val summaryIndexRepository =
            SummaryClientIndexRepository(firestore)

        val maintenanceRepository =
            MaintenanceRepository(firestore)

        val ownerOperationLockRepository =
            OwnerOperationLockRepository(
                firestore = firestore,
                maintenanceRepository = maintenanceRepository
            )

        val clientEditWindowService =
            ClientEditWindowService(
                properties = properties,
                clock = testClock
            )

        return OrderService(
            transactionExecutor = transactionExecutor,
            orderRepository = orderRepository,
            clientRepository = clientRepository,
            historyRepository = historyRepository,
            globalSummaryRepository = globalSummaryRepository,
            summaryIndexBucketRepository = summaryIndexBucketRepository,
            summaryIndexRepository = summaryIndexRepository,
            ownerOperationLockRepository = ownerOperationLockRepository,
            properties = properties,
            clock = testClock,
            ownerOperationLockService = ownerOperationLockService,
            clientEditWindowService = clientEditWindowService
        )
    }
}

Important: this test file assumes your current "ClientMonthlyHistory" already contains:

val totalInvoiceAmountWithGst: Long = 0
val invoiceCount: Long = 0

and that your "ClientHistoryRepository" persists both fields.

Also, this test update exposes an important consequence: your "OrderService" itself must now update "invoiceCount" and reset "totalInvoiceAmountWithGst"/counts in "copyHistoryToNewMonth()". If those service changes haven't been made yet, these tests will correctly fail.
