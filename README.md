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
            900,
            augustHistory.totalGstAmount
        )

        assertEquals(
            3000,
            augustHistory.totalExpenses
        )

        assertEquals(
            10000,
            augustHistory.closingBalance
        )

        assertEquals(
            10000,
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
            10000,
            septemberHistory!!.openingBalance
        )

        assertEquals(
            0,
            septemberHistory.totalInvoiceAmount
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
            10000,
            septemberHistory.closingBalance
        )

        assertEquals(
            10000,
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

        val clientEditWindowService =
            ClientEditWindowService(
                properties = properties,
                clock = testClock
            )

        return OrderService(
            transactionExecutor = transactionExecutor,
            orderRepository = orderRepository,
            clientRepository = clientRepository,
            historyBucketRepository = historyBucketRepository,
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
                status = ClientType.RECEIVABLE,
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(augustIndex)

        assertEquals(
            client.id,
            augustIndex!!.clientId
        )

        assertEquals(
            10000,
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
                status = ClientType.RECEIVABLE,
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(septemberIndex)

        assertEquals(
            10000,
            septemberIndex!!.amount
        )

        assertEquals(
            ClientType.RECEIVABLE,
            septemberIndex.status
        )
    }
}
