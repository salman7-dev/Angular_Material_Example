package com.clientledger.core.service.order

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientMonthlyHistory
import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.GlobalSummary
import com.clientledger.core.domain.Order
import com.clientledger.core.domain.OrderStatus
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.lock.OwnerOperationLockRepository
import com.clientledger.core.repository.order.OrderRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.service.client.ClientEditWindowService
import com.clientledger.core.service.lock.OwnerOperationLockService
import com.clientledger.core.transaction.FirestoreTransactionExecutor
import com.clientledger.core.utils.IdGenerator
import com.google.cloud.firestore.Transaction
import java.time.Clock
import java.time.LocalDate
import java.time.YearMonth
import kotlin.math.abs

class OrderService(
    private val transactionExecutor: FirestoreTransactionExecutor,
    private val orderRepository: OrderRepository,
    private val clientRepository: ClientRepository,
    private val historyBucketRepository: ClientHistoryBucketRepository,
    private val historyRepository: ClientHistoryRepository,
    private val globalSummaryRepository: GlobalSummaryRepository,
    private val summaryIndexBucketRepository: SummaryClientIndexBucketRepository,
    private val summaryIndexRepository: SummaryClientIndexRepository,
    private val ownerOperationLockRepository: OwnerOperationLockRepository,
    private val properties: ClientLedgerProperties,
    private val clock: Clock,
    private val ownerOperationLockService: OwnerOperationLockService,
    private val clientEditWindowService: ClientEditWindowService
) {

    fun create(order: Order): Order {

        require(order.ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(order.clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        require(order.items.isNotEmpty()) {
            "order items must not be empty"
        }

        val orderDate =
            try {
                LocalDate.parse(order.orderDate)
            } catch (exception: Exception) {
                throw IllegalArgumentException(
                    "orderDate must be in yyyy-MM-dd format",
                    exception
                )
            }

        return ownerOperationLockService.executeBusiness(
            ownerId = order.ownerId
        ) { context, fence ->

            transactionExecutor.execute(context) { transaction, operationContext ->

                require(
                    ownerOperationLockRepository.isFenceValidInTransaction(
                        transaction = transaction,
                        ownerId = order.ownerId,
                        expectedFence = fence
                    )
                ) {
                    "Owner operation lock fence changed: ${order.ownerId}"
                }

                operationContext.checkDeadline()

                /*
                 * READ
                 *
                 * Read the client before doing any writes.
                 */
                val client =
                    clientRepository.findByIdInTransaction(
                        transaction = transaction,
                        ownerId = order.ownerId,
                        clientId = order.clientId
                    )
                        ?: throw IllegalArgumentException(
                            "Client not found: ${order.clientId}"
                        )

                clientEditWindowService.validateEditableDate(
                    client = client,
                    targetDate = orderDate
                )

                val totalInvoiceAmount =
                    order.items.sumOf { it.amount }

                val totalGstAmount =
                    order.items.sumOf { it.gstAmount }

                val totalExpense =
                    order.items.sumOf { it.expense }

                val profitAmount =
                    totalInvoiceAmount - totalExpense

                val orderWithCalculatedValues =
                    order.copy(
                        id =
                        if (order.id.isBlank()) {
                            IdGenerator.generateOrderId()
                        } else {
                            order.id
                        },
                        totalInvoiceAmount = totalInvoiceAmount,
                        totalGstAmount = totalGstAmount,
                        totalExpense = totalExpense,
                        profitAmount = profitAmount,
                        status =
                        if (order.status == OrderStatus.CREATED) {
                            OrderStatus.CREATED
                        } else {
                            order.status
                        }
                    )

                operationContext.checkDeadline()

                /*
                 * READ / PLAN
                 *
                 * History propagation calculates everything in memory.
                 * No history write happens here.
                 */
                val propagationPlan =
                    planHistoryPropagation(
                        transaction = transaction,
                        client = client,
                        order = orderWithCalculatedValues,
                        orderDate = orderDate
                    )

                operationContext.checkDeadline()

                /*
                 * READ / PLAN
                 *
                 * Build GlobalSummary deltas for every affected month.
                 *
                 * This happens BEFORE any history or summary write.
                 */
                val globalSummaryPlans =
                    propagationPlan.months.map { yearMonth ->

                        val delta =
                            globalSummaryDelta(
                                order = orderWithCalculatedValues,
                                orderMonth = propagationPlan.orderMonth,
                                yearMonth = yearMonth,
                                previousHistory =
                                propagationPlan.previousHistories[
                                    yearMonth
                                ],
                                updatedHistory =
                                propagationPlan.updatedHistories[
                                    yearMonth
                                ]
                                    ?: throw IllegalStateException(
                                        "Updated history not found for $yearMonth"
                                    )
                            )

                        globalSummaryRepository.planDeltaInTransaction(
                            transaction = transaction,
                            ownerId = client.ownerId,
                            yearMonth = yearMonth.toString(),
                            delta = delta
                        )
                    }

                operationContext.checkDeadline()

                /*
                 * WRITE / APPLY
                 *
                 * All transaction reads are now complete.
                 */

                propagationPlan.updatedHistories.values.forEach { history ->

                    historyRepository.saveInTransaction(
                        transaction = transaction,
                        history = history,
                        bucketId = client.bucketId
                    )
                }

                globalSummaryPlans.forEach { plan ->

                    globalSummaryRepository.applyDeltaPlanInTransaction(
                        transaction = transaction,
                        plan = plan
                    )
                }

                operationContext.checkDeadline()

                /*
                 * Finally persist the source Order.
                 */
                orderRepository.createInTransaction(
                    transaction = transaction,
                    order = orderWithCalculatedValues
                )
            }
        }
    }

    private data class HistoryPropagationPlan(
        val orderMonth: YearMonth,
        val months: List<YearMonth>,
        val previousHistories: Map<YearMonth, ClientMonthlyHistory>,
        val updatedHistories: Map<YearMonth, ClientMonthlyHistory>
    )

    private fun planHistoryPropagation(
        transaction: Transaction,
        client: Client,
        order: Order,
        orderDate: LocalDate
    ): HistoryPropagationPlan {

        val orderMonth =
            YearMonth.from(orderDate)

        val currentMonth =
            YearMonth.now(clock)

        require(!orderMonth.isAfter(currentMonth)) {
            "Order date cannot be in the future."
        }

        require(client.bucketId.isNotBlank()) {
            "Client history bucketId must not be blank"
        }

        val months =
            generateSequence(orderMonth) { previous ->
                previous
                    .plusMonths(1)
                    .takeUnless { it.isAfter(currentMonth) }
            }.toList()

        /*
         * READ ALL HISTORIES FIRST.
         */
        val existingHistories =
            months.associateWith { yearMonth ->

                historyRepository.findInTransaction(
                    transaction = transaction,
                    ownerId = client.ownerId,
                    yearMonth = yearMonth.toString(),
                    bucketId = client.bucketId,
                    clientId = client.id
                )
            }

        /*
         * CALCULATE ALL UPDATED HISTORIES IN MEMORY.
         */
        val updatedHistories =
            mutableMapOf<YearMonth, ClientMonthlyHistory>()

        var previousHistory: ClientMonthlyHistory? = null

        months.forEach { yearMonth ->

            val existingHistory =
                existingHistories[yearMonth]

            val updatedHistory =
                when {

                    yearMonth == orderMonth -> {

                        val baseHistory =
                            existingHistory
                                ?: throw IllegalStateException(
                                    "Client history not found for " +
                                            "${client.id} in $yearMonth"
                                )

                        applyOrderToHistory(
                            history = baseHistory,
                            order = order
                        )
                    }

                    existingHistory == null -> {

                        val previous =
                            previousHistory
                                ?: throw IllegalStateException(
                                    "Previous history not found before $yearMonth"
                                )

                        copyHistoryToNewMonth(
                            previousHistory = previous,
                            newYearMonth = yearMonth.toString()
                        )
                    }

                    else -> {

                        val previous =
                            previousHistory
                                ?: throw IllegalStateException(
                                    "Previous history not found before $yearMonth"
                                )

                        recalculateOpeningFromPreviousMonth(
                            history = existingHistory,
                            previousHistory = previous
                        )
                    }
                }

            updatedHistories[yearMonth] =
                updatedHistory

            previousHistory =
                updatedHistory
        }

        return HistoryPropagationPlan(
            orderMonth = orderMonth,
            months = months,
            previousHistories =
            existingHistories
                .mapNotNull { (month, history) ->
                    history?.let { month to it }
                }
                .toMap(),
            updatedHistories = updatedHistories
        )
    }

    private fun globalSummaryDelta(
        order: Order,
        orderMonth: YearMonth,
        yearMonth: YearMonth,
        previousHistory: ClientMonthlyHistory?,
        updatedHistory: ClientMonthlyHistory
    ): GlobalSummary {

        val positionDelta =
            calculatePositionDelta(
                previousHistory = previousHistory,
                updatedHistory = updatedHistory
            )

        /*
         * Order activity belongs ONLY to the order month.
         *
         * It is NOT propagated to later months.
         */
        val activityDelta =
            if (yearMonth == orderMonth) {

                GlobalSummary(
                    totalInvoiceAmount =
                    order.totalInvoiceAmount,

                    totalInvoiceAmountWithGst =
                    order.totalInvoiceAmount +
                            order.totalGstAmount,

                    totalGstAmount =
                    order.totalGstAmount,

                    totalExpenseAmount =
                    order.totalExpense,

                    totalPaymentAmount = 0,

                    /*
                     * Monthly cash flow:
                     *
                     * payments received - expenses
                     *
                     * Order itself is not cash received.
                     */
                    cashFlow =
                    -order.totalExpense,

                    netProfit =
                    order.profitAmount
                )

            } else {
                GlobalSummary()
            }

        return GlobalSummary(
            receivableClientCount =
            positionDelta.receivableClientCount,

            advanceClientCount =
            positionDelta.advanceClientCount,

            settledClientCount =
            positionDelta.settledClientCount,

            totalReceivableAmount =
            positionDelta.totalReceivableAmount,

            totalAdvanceAmount =
            positionDelta.totalAdvanceAmount,

            cashFlow =
            activityDelta.cashFlow,

            netProfit =
            activityDelta.netProfit,

            totalInvoiceAmount =
            activityDelta.totalInvoiceAmount,

            totalInvoiceAmountWithGst =
            activityDelta.totalInvoiceAmountWithGst,

            totalGstAmount =
            activityDelta.totalGstAmount,

            totalExpenseAmount =
            activityDelta.totalExpenseAmount,

            totalPaymentAmount =
            activityDelta.totalPaymentAmount
        )
    }

    private fun calculatePositionDelta(
        previousHistory: ClientMonthlyHistory?,
        updatedHistory: ClientMonthlyHistory
    ): GlobalSummary {

        if (previousHistory == null) {

            return positionFromHistory(
                updatedHistory
            )
        }

        val oldPosition =
            positionFromHistory(
                previousHistory
            )

        val newPosition =
            positionFromHistory(
                updatedHistory
            )

        return GlobalSummary(
            receivableClientCount =
            newPosition.receivableClientCount -
                    oldPosition.receivableClientCount,

            advanceClientCount =
            newPosition.advanceClientCount -
                    oldPosition.advanceClientCount,

            settledClientCount =
            newPosition.settledClientCount -
                    oldPosition.settledClientCount,

            totalReceivableAmount =
            newPosition.totalReceivableAmount -
                    oldPosition.totalReceivableAmount,

            totalAdvanceAmount =
            newPosition.totalAdvanceAmount -
                    oldPosition.totalAdvanceAmount
        )
    }

    private fun positionFromHistory(
        history: ClientMonthlyHistory
    ): GlobalSummary {

        return GlobalSummary(
            receivableClientCount =
            if (history.status == ClientType.RECEIVABLE) {
                1
            } else {
                0
            },

            advanceClientCount =
            if (history.status == ClientType.ADVANCE) {
                1
            } else {
                0
            },

            settledClientCount =
            if (history.status == ClientType.SETTLED) {
                1
            } else {
                0
            },

            totalReceivableAmount =
            history.receivable,

            totalAdvanceAmount =
            history.advance
        )
    }

    private fun copyHistoryToNewMonth(
        previousHistory: ClientMonthlyHistory,
        newYearMonth: String
    ): ClientMonthlyHistory {

        val openingBalance =
            previousHistory.closingBalance

        return previousHistory.copy(
            yearMonth = newYearMonth,
            openingBalance = openingBalance,
            totalInvoiceAmount = 0,
            totalPayments = 0,
            totalDiscount = 0,
            totalExpenses = 0,
            totalGstAmount = 0,
            closingBalance = openingBalance,
            receivable =
            if (openingBalance > 0) {
                openingBalance
            } else {
                0
            },
            advance =
            if (openingBalance < 0) {
                abs(openingBalance)
            } else {
                0
            },
            status =
            statusFromClosingBalance(
                openingBalance
            )
        )
    }

    private fun applyOrderToHistory(
        history: ClientMonthlyHistory,
        order: Order
    ): ClientMonthlyHistory {

        val newTotalInvoiceAmount =
            history.totalInvoiceAmount +
                    order.totalInvoiceAmount

        val newTotalGstAmount =
            history.totalGstAmount +
                    order.totalGstAmount

        val newTotalExpenses =
            history.totalExpenses +
                    order.totalExpense

        val newClosingBalance =
            history.openingBalance +
                    newTotalInvoiceAmount -
                    history.totalPayments -
                    history.totalDiscount

        val status =
            statusFromClosingBalance(
                newClosingBalance
            )

        return history.copy(
            totalInvoiceAmount =
            newTotalInvoiceAmount,

            totalGstAmount =
            newTotalGstAmount,

            totalExpenses =
            newTotalExpenses,

            closingBalance =
            newClosingBalance,

            receivable =
            if (newClosingBalance > 0) {
                newClosingBalance
            } else {
                0
            },

            advance =
            if (newClosingBalance < 0) {
                abs(newClosingBalance)
            } else {
                0
            },

            status = status
        )
    }

    private fun recalculateOpeningFromPreviousMonth(
        history: ClientMonthlyHistory,
        previousHistory: ClientMonthlyHistory
    ): ClientMonthlyHistory {

        val newOpeningBalance =
            previousHistory.closingBalance

        val newClosingBalance =
            newOpeningBalance +
                    history.totalInvoiceAmount -
                    history.totalPayments -
                    history.totalDiscount

        val status =
            statusFromClosingBalance(
                newClosingBalance
            )

        return history.copy(
            openingBalance =
            newOpeningBalance,

            closingBalance =
            newClosingBalance,

            receivable =
            if (newClosingBalance > 0) {
                newClosingBalance
            } else {
                0
            },

            advance =
            if (newClosingBalance < 0) {
                abs(newClosingBalance)
            } else {
                0
            },

            status = status
        )
    }

    private fun statusFromClosingBalance(
        closingBalance: Long
    ): ClientType {

        return when {
            closingBalance > 0 ->
                ClientType.RECEIVABLE

            closingBalance < 0 ->
                ClientType.ADVANCE

            else ->
                ClientType.SETTLED
        }
    }
}package com.clientledger.core.repository.order

import com.clientledger.core.domain.Order
import com.clientledger.core.domain.OrderItem
import com.clientledger.core.domain.OrderStatus
import com.clientledger.core.domain.OrderUnit
import com.google.cloud.firestore.CollectionReference
import com.google.cloud.firestore.DocumentSnapshot
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Repository
import java.time.Instant
import java.time.LocalDate
import java.time.format.DateTimeFormatter
import java.time.format.DateTimeParseException
import java.util.Date

@Repository
class OrderRepository(
    private val firestore: Firestore
) {

    private val orderDateFormatter =
        DateTimeFormatter.ISO_LOCAL_DATE

    private fun transactionCollection(
        ownerId: String,
        yearMonth: String
    ): CollectionReference {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }

        return firestore
            .collection("owners")
            .document(ownerId)
            .collection("transactions")
            .document(yearMonth)
            .collection("items")
    }

    fun create(
        order: Order
    ): Order {

        validateOrder(order)

        transactionCollection(
            order.ownerId,
            yearMonthFromOrderDate(order.orderDate)
        )
            .document(order.id)
            .set(toFirestoreMap(order))
            .get()

        return order
    }

    fun findById(
        ownerId: String,
        orderDate: String,
        orderId: String
    ): Order? {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(orderId.isNotBlank()) {
            "order id must not be blank"
        }

        val yearMonth =
            yearMonthFromOrderDate(orderDate)

        val snapshot =
            transactionCollection(
                ownerId,
                yearMonth
            )
                .document(orderId)
                .get()
                .get()

        if (!snapshot.exists()) {
            return null
        }

        return toOrder(snapshot)
    }

    fun exists(
        ownerId: String,
        orderDate: String,
        orderId: String
    ): Boolean {

        return findById(
            ownerId,
            orderDate,
            orderId
        ) != null
    }

    fun createInTransaction(
        transaction: Transaction,
        order: Order
    ): Order {

        validateOrder(order)

        val reference =
            transactionCollection(
                order.ownerId,
                yearMonthFromOrderDate(order.orderDate)
            )
                .document(order.id)

        transaction.set(
            reference,
            toFirestoreMap(order)
        )

        return order
    }

    private fun validateOrder(
        order: Order
    ) {

        require(order.ownerId.isNotBlank()) {
            "order ownerId must not be blank"
        }

        require(order.id.isNotBlank()) {
            "order id must not be blank"
        }

        require(order.clientId.isNotBlank()) {
            "order clientId must not be blank"
        }

        validateOrderDate(order.orderDate)

        order.items.forEach { item ->
            require(item.name.isNotBlank()) {
                "order item name must not be blank"
            }

            require(item.quantity > 0) {
                "order item quantity must be greater than zero"
            }

            require(item.amount >= 0) {
                "order item amount must not be negative"
            }

            require(item.gstRate >= 0) {
                "order item gstRate must not be negative"
            }

            require(item.gstAmount >= 0) {
                "order item gstAmount must not be negative"
            }

            require(item.expense >= 0) {
                "order item expense must not be negative"
            }
        }
    }

    private fun validateOrderDate(
        orderDate: String
    ) {

        try {
            LocalDate.parse(
                orderDate,
                orderDateFormatter
            )
        } catch (exception: DateTimeParseException) {

            throw IllegalArgumentException(
                "orderDate must be in yyyy-MM-dd format",
                exception
            )
        }
    }

    private fun yearMonthFromOrderDate(
        orderDate: String
    ): String {

        val date =
            try {
                LocalDate.parse(
                    orderDate,
                    orderDateFormatter
                )
            } catch (exception: DateTimeParseException) {

                throw IllegalArgumentException(
                    "orderDate must be in yyyy-MM-dd format",
                    exception
                )
            }

        return "%04d-%02d".format(
            date.year,
            date.monthValue
        )
    }

    private fun toFirestoreMap(
        order: Order
    ): Map<String, Any?> {

        return mapOf(
            "id" to order.id,
            "ownerId" to order.ownerId,
            "clientId" to order.clientId,
            "orderDate" to order.orderDate,
            "deliveryDate" to order.deliveryDate,
            "items" to order.items.map { item ->
                mapOf(
                    "name" to item.name,
                    "quantity" to item.quantity,
                    "unit" to item.unit.name,
                    "amount" to item.amount,
                    "gstRate" to item.gstRate,
                    "gstAmount" to item.gstAmount,
                    "expense" to item.expense
                )
            },
            "totalInvoiceAmount" to order.totalInvoiceAmount,
            "totalGstAmount" to order.totalGstAmount,
            "totalExpense" to order.totalExpense,
            "profitAmount" to order.profitAmount,
            "status" to order.status.name,
            "createdAt" to order.createdAt,
            "updatedAt" to order.updatedAt
        )
    }

    private fun toOrder(
        snapshot: DocumentSnapshot
    ): Order {

        val itemMaps =
            snapshot.get("items")
                    as? List<*>
                ?: emptyList<Any?>()

        val items =
            itemMaps.mapNotNull { value ->

                val data =
                    value as? Map<*, *>
                        ?: return@mapNotNull null

                OrderItem(
                    name =
                    data["name"] as? String
                        ?: "",

                    quantity =
                    (data["quantity"] as? Number)
                        ?.toInt()
                        ?: 0,

                    unit =
                    (data["unit"] as? String)
                        ?.let {
                            OrderUnit.valueOf(it)
                        }
                        ?: OrderUnit.SINGLE,

                    amount =
                    (data["amount"] as? Number)
                        ?.toLong()
                        ?: 0,

                    gstRate =
                    (data["gstRate"] as? Number)
                        ?.toInt()
                        ?: 0,

                    gstAmount =
                    (data["gstAmount"] as? Number)
                        ?.toLong()
                        ?: 0,

                    expense =
                    (data["expense"] as? Number)
                        ?.toLong()
                        ?: 0
                )
            }

        return Order(
            id =
            snapshot.getString("id")
                ?: snapshot.id,

            ownerId =
            snapshot.getString("ownerId")
                ?: "",

            clientId =
            snapshot.getString("clientId")
                ?: "",

            orderDate =
            snapshot.getString("orderDate")
                ?: "",

            deliveryDate =
            snapshot.getString("deliveryDate"),

            items = items,

            totalInvoiceAmount =
            snapshot.getLong("totalInvoiceAmount")
                ?: 0,

            totalGstAmount =
            snapshot.getLong("totalGstAmount")
                ?: 0,

            totalExpense =
            snapshot.getLong("totalExpense")
                ?: 0,

            profitAmount =
            snapshot.getLong("profitAmount")
                ?: 0,

            status =
            snapshot.getString("status")
                ?.let {
                    OrderStatus.valueOf(it)
                }
                ?: OrderStatus.CREATED,

            createdAt =
            toInstant(
                snapshot.get("createdAt")
            ),

            updatedAt =
            toInstant(
                snapshot.get("updatedAt")
            )
        )
    }

    private fun toInstant(
        value: Any?
    ): Instant {

        return when (value) {

            is Instant ->
                value

            is Date ->
                value.toInstant()

            is com.google.cloud.Timestamp ->
                value.toDate().toInstant()

            is Map<*, *> -> {

                val seconds =
                    (value["seconds"] as? Number)
                        ?.toLong()
                        ?: return Instant.EPOCH

                val nanos =
                    (value["nanos"] as? Number)
                        ?.toInt()
                        ?: 0

                Instant.ofEpochSecond(
                    seconds,
                    nanos.toLong()
                )
            }

            else ->
                Instant.EPOCH
        }
    }
}package com.clientledger.core.service.order

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
}package com.clientledger.core.repository.summary

import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.SummaryClientIndex
import com.clientledger.core.pagination.PageCursor
import com.clientledger.core.pagination.PageResult
import com.google.cloud.firestore.*
import org.springframework.stereotype.Repository
import java.util.Base64

@Repository
class SummaryClientIndexRepository(
    private val firestore: Firestore
) {

    private fun statusReference(
        ownerId: String,
        yearMonth: String,
        status: ClientType
    ): DocumentReference {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }

        return firestore
            .collection("owners")
            .document(ownerId)
            .collection("summary_client_index")
            .document(yearMonth)
            .collection("status")
            .document(status.name.lowercase())
    }

    private fun bucketCollection(
        ownerId: String,
        yearMonth: String,
        status: ClientType
    ): CollectionReference {

        return statusReference(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status
        )
            .collection("buckets")
    }

    private fun clientReference(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String,
        clientId: String
    ): DocumentReference {

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        require(clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status
        )
            .document(bucketId)
            .collection("clients")
            .document(clientId)
    }

    fun insert(
        ownerId: String,
        yearMonth: String,
        bucketId: String,
        index: SummaryClientIndex
    ): SummaryClientIndex {

        val reference = clientReference(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = index.status,
            bucketId = bucketId,
            clientId = index.clientId
        )

        reference
            .set(toFirestoreMap(index))
            .get()

        return index
    }

    fun insertInTransaction(
        transaction: Transaction,
        ownerId: String,
        yearMonth: String,
        bucketId: String,
        index: SummaryClientIndex
    ): SummaryClientIndex {

        val reference = clientReference(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = index.status,
            bucketId = bucketId,
            clientId = index.clientId
        )

        transaction.set(
            reference,
            toFirestoreMap(index)
        )

        return index
    }

    fun find(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String,
        clientId: String
    ): SummaryClientIndex? {

        val reference = clientReference(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status,
            bucketId = bucketId,
            clientId = clientId
        )

        val snapshot = reference
            .get()
            .get()

        if (!snapshot.exists()) {
            return null
        }

        return toIndex(snapshot)
    }

    fun findPage(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        size: Int,
        cursor: String?
    ): PageResult<SummaryClientIndex> {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }

        require(size > 0) {
            "size must be greater than zero"
        }

        val pageCursor = decodeCursor(cursor)

        var currentBucketId = pageCursor?.bucketId
        var currentClientId = pageCursor?.clientId

        val result = mutableListOf<SummaryClientIndex>()

        while (result.size < size) {

            val bucketId =
                currentBucketId
                    ?: findFirstBucketId(
                        ownerId = ownerId,
                        yearMonth = yearMonth,
                        status = status
                    )

            if (bucketId == null) {
                break
            }

            val remaining = size - result.size

            val clientCollection =
                bucketCollection(
                    ownerId = ownerId,
                    yearMonth = yearMonth,
                    status = status
                )
                    .document(bucketId)
                    .collection("clients")

            val clientQuery =
                clientCollection
                    .orderBy("__name__")
                    .let { query ->

                        if (
                            currentBucketId == bucketId &&
                            currentClientId != null
                        ) {
                            query.startAfter(currentClientId)
                        } else {
                            query
                        }
                    }
                    .limit(remaining + 1)

            val clientSnapshots =
                clientQuery
                    .get()
                    .get()
                    .documents

            val hasMoreInCurrentBucket =
                clientSnapshots.size > remaining

            val documentsToReturn =
                if (hasMoreInCurrentBucket) {
                    clientSnapshots.take(remaining)
                } else {
                    clientSnapshots
                }

            result.addAll(
                documentsToReturn.map(::toIndex)
            )

            if (hasMoreInCurrentBucket) {

                val lastDocument =
                    documentsToReturn.last()

                return PageResult(
                    content = result,
                    hasNext = true,
                    nextCursor = encodeCursor(
                        PageCursor(
                            bucketId = bucketId,
                            clientId = lastDocument.id
                        )
                    )
                )
            }

            val nextBucketId =
                findNextBucketId(
                    ownerId = ownerId,
                    yearMonth = yearMonth,
                    status = status,
                    bucketId = bucketId
                )

            if (result.size == size) {

                val lastDocument =
                    documentsToReturn.lastOrNull()

                if (nextBucketId != null && lastDocument != null) {
                    return PageResult(
                        content = result,
                        hasNext = true,
                        nextCursor = encodeCursor(
                            PageCursor(
                                bucketId = bucketId,
                                clientId = lastDocument.id
                            )
                        )
                    )
                }

                return PageResult(
                    content = result,
                    hasNext = false,
                    nextCursor = null
                )
            }

            if (nextBucketId == null) {
                break
            }

            currentBucketId = nextBucketId
            currentClientId = null
        }

        return PageResult(
            content = result,
            hasNext = false,
            nextCursor = null
        )
    }

    private fun findFirstBucketId(
        ownerId: String,
        yearMonth: String,
        status: ClientType
    ): String? {

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status
        )
            .orderBy("__name__")
            .limit(1)
            .get()
            .get()
            .documents
            .firstOrNull()
            ?.id
    }

    private fun findNextBucketId(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String
    ): String? {

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status
        )
            .orderBy("__name__")
            .startAfter(bucketId)
            .limit(1)
            .get()
            .get()
            .documents
            .firstOrNull()
            ?.id
    }

    private fun encodeCursor(
        cursor: PageCursor
    ): String {

        val raw =
            "${cursor.bucketId}|${cursor.clientId}"

        return Base64
            .getUrlEncoder()
            .withoutPadding()
            .encodeToString(
                raw.toByteArray()
            )
    }

    private fun decodeCursor(
        cursor: String?
    ): PageCursor? {

        if (cursor.isNullOrBlank()) {
            return null
        }

        return try {

            val decoded =
                String(
                    Base64
                        .getUrlDecoder()
                        .decode(cursor)
                )

            val parts = decoded.split("|")

            require(parts.size == 2)
            require(parts[0].isNotBlank())
            require(parts[1].isNotBlank())

            PageCursor(
                bucketId = parts[0],
                clientId = parts[1]
            )

        } catch (exception: Exception) {

            throw IllegalArgumentException(
                "Invalid cursor",
                exception
            )
        }
    }

    private fun toFirestoreMap(
        index: SummaryClientIndex
    ): Map<String, Any> {

        return mapOf(
            "clientId" to index.clientId,
            "amount" to index.amount,
            "status" to index.status.name
        )
    }

    private fun toIndex(
        snapshot: DocumentSnapshot
    ): SummaryClientIndex {

        return SummaryClientIndex(
            clientId =
            snapshot.getString("clientId")
                ?: snapshot.id,

            amount =
            snapshot.getLong("amount")
                ?: 0,

            status =
            snapshot.getString("status")
                ?.let { ClientType.valueOf(it) }
                ?: ClientType.SETTLED
        )
    }

    fun findAllInBucket(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String
    ): List<SummaryClientIndex> {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        val clientCollection =
            bucketCollection(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status
            )
                .document(bucketId)
                .collection("clients")

        return clientCollection
            .orderBy("__name__")
            .get()
            .get()
            .documents
            .map(::toIndex)
    }

    fun copyInTransaction(
        transaction: Transaction,
        ownerId: String,
        newYearMonth: String,
        status: ClientType,
        bucketId: String,
        index: SummaryClientIndex
    ): SummaryClientIndex {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(newYearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "newYearMonth must be in yyyy-MM format"
        }

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        require(index.clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        require(index.status == status) {
            "Index status does not match bucket status"
        }

        val targetReference =
            bucketCollection(
                ownerId = ownerId,
                yearMonth = newYearMonth,
                status = status
            )
                .document(bucketId)
                .collection("clients")
                .document(index.clientId)

        transaction.set(
            targetReference,
            mapOf(
                "clientId" to index.clientId,
                "amount" to index.amount,
                "status" to index.status.name
            )
        )

        return index
    }
}
package com.clientledger.core.repository.summary

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.SummaryClientIndex
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.assertThrows
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test

class SummaryClientIndexRepositoryTest {

    companion object {

        private lateinit var firestore: Firestore

        @JvmStatic
        @BeforeAll
        fun setup() {
            firestore = FirestoreOptions.newBuilder()
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
            firestore.close()
        }
    }

    private fun repository(): SummaryClientIndexRepository {
        return SummaryClientIndexRepository(
            firestore = firestore
        )
    }

    @Test
    fun insertAndFindReceivableClient() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val index = SummaryClientIndex(
            clientId = "client-${System.nanoTime()}",
            amount = 10000,
            status = ClientType.RECEIVABLE
        )

        repository.insert(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_000",
            index = index
        )

        val result = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000",
            clientId = index.clientId
        )

        assertNotNull(result)
        assertEquals(index.clientId, result!!.clientId)
        assertEquals(10000, result.amount)
        assertEquals(ClientType.RECEIVABLE, result.status)
    }

    @Test
    fun insertAndFindAdvanceClient() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val index = SummaryClientIndex(
            clientId = "client-${System.nanoTime()}",
            amount = 5000,
            status = ClientType.ADVANCE
        )

        repository.insert(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_000",
            index = index
        )

        val result = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = ClientType.ADVANCE,
            bucketId = "bucket_000",
            clientId = index.clientId
        )

        assertNotNull(result)
        assertEquals(index.clientId, result!!.clientId)
        assertEquals(5000, result.amount)
        assertEquals(ClientType.ADVANCE, result.status)
    }

    @Test
    fun differentBucketsKeepClientsSeparate() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val firstClient = SummaryClientIndex(
            clientId = "client-1-${System.nanoTime()}",
            amount = 1000,
            status = ClientType.RECEIVABLE
        )

        val secondClient = SummaryClientIndex(
            clientId = "client-2-${System.nanoTime()}",
            amount = 2000,
            status = ClientType.RECEIVABLE
        )

        repository.insert(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_000",
            index = firstClient
        )

        repository.insert(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_001",
            index = secondClient
        )

        val firstResult = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000",
            clientId = firstClient.clientId
        )

        val secondResult = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_001",
            clientId = secondClient.clientId
        )

        assertNotNull(firstResult)
        assertNotNull(secondResult)

        assertEquals(firstClient.clientId, firstResult!!.clientId)
        assertEquals(secondClient.clientId, secondResult!!.clientId)
    }

    @Test
    fun findReturnsNullWhenClientDoesNotExist() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        val result = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000",
            clientId = "missing-client"
        )

        assertEquals(null, result)
    }

    @Test
    fun insertInTransactionAndFind() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val index = SummaryClientIndex(
            clientId = "client-${System.nanoTime()}",
            amount = 7500,
            status = ClientType.RECEIVABLE
        )

        firestore.runTransaction { transaction ->

            repository.insertInTransaction(
                transaction = transaction,
                ownerId = ownerId,
                yearMonth = yearMonth,
                bucketId = "bucket_000",
                index = index
            )

            null
        }.get()

        val result = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000",
            clientId = index.clientId
        )

        assertNotNull(result)
        assertEquals(index.clientId, result!!.clientId)
        assertEquals(7500, result.amount)
        assertEquals(ClientType.RECEIVABLE, result.status)
    }

    @Test
    fun findPageReturnsEmptyWhenNoClientsExist() {

        val repository = repository()

        val result =
            repository.findPage(
                ownerId = "owner-${System.nanoTime()}",
                yearMonth = "2026-09",
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = null
            )

        assertEquals(0, result.content.size)
        assertEquals(false, result.hasNext)
        assertEquals(null, result.nextCursor)
    }

    @Test
    fun findPageReturnsSingleClient() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 1
        )

        val result =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = null
            )

        assertEquals(1, result.content.size)
        assertEquals("client-000", result.content.first().clientId)
        assertEquals(false, result.hasNext)
        assertEquals(null, result.nextCursor)
    }

    @Test
    fun findPageReturnsExactly50Clients() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 50
        )

        val result =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = null
            )

        assertEquals(50, result.content.size)
        assertEquals("client-000", result.content.first().clientId)
        assertEquals("client-049", result.content.last().clientId)
        assertEquals(false, result.hasNext)
        assertEquals(null, result.nextCursor)
    }

    @Test
    fun findPageReturns51ClientsAcrossTwoPages() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 51
        )

        val firstPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = null
            )

        assertEquals(50, firstPage.content.size)
        assertEquals("client-000", firstPage.content.first().clientId)
        assertEquals("client-049", firstPage.content.last().clientId)
        assertEquals(true, firstPage.hasNext)
        assertNotNull(firstPage.nextCursor)

        val secondPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = firstPage.nextCursor
            )

        assertEquals(1, secondPage.content.size)
        assertEquals("client-050", secondPage.content.first().clientId)
        assertEquals(false, secondPage.hasNext)
        assertEquals(null, secondPage.nextCursor)
    }

    @Test
    fun findPageReturns100ClientsAcrossTwoPages() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 100
        )

        val firstPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = null
            )

        val secondPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = firstPage.nextCursor
            )

        assertEquals(50, firstPage.content.size)
        assertEquals("client-049", firstPage.content.last().clientId)
        assertEquals(true, firstPage.hasNext)

        assertEquals(50, secondPage.content.size)
        assertEquals("client-050", secondPage.content.first().clientId)
        assertEquals("client-099", secondPage.content.last().clientId)
        assertEquals(false, secondPage.hasNext)
        assertEquals(null, secondPage.nextCursor)
    }

    @Test
    fun findPageReturns101ClientsAcrossThreePages() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 101
        )

        val firstPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = null
            )

        val secondPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = firstPage.nextCursor
            )

        val thirdPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = secondPage.nextCursor
            )

        assertEquals(50, firstPage.content.size)
        assertEquals("client-000", firstPage.content.first().clientId)
        assertEquals("client-049", firstPage.content.last().clientId)
        assertEquals(true, firstPage.hasNext)

        assertEquals(50, secondPage.content.size)
        assertEquals("client-050", secondPage.content.first().clientId)
        assertEquals("client-099", secondPage.content.last().clientId)
        assertEquals(true, secondPage.hasNext)

        assertEquals(1, thirdPage.content.size)
        assertEquals("client-100", thirdPage.content.first().clientId)
        assertEquals(false, thirdPage.hasNext)
        assertEquals(null, thirdPage.nextCursor)
    }

    @Test
    fun findPageDoesNotSkipOrDuplicateClients() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 101
        )

        val firstPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = null
            )

        val secondPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = firstPage.nextCursor
            )

        val thirdPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 50,
                cursor = secondPage.nextCursor
            )

        val clientIds =
            firstPage.content.map { it.clientId } +
                    secondPage.content.map { it.clientId } +
                    thirdPage.content.map { it.clientId }

        assertEquals(101, clientIds.size)
        assertEquals(101, clientIds.distinct().size)

        assertEquals(
            (0..100).map { "client-%03d".format(it) },
            clientIds
        )
    }

    @Test
    fun findPageHandlesMultipleBuckets() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 301
        )

        val firstPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 100,
                cursor = null
            )

        val secondPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 100,
                cursor = firstPage.nextCursor
            )

        val thirdPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 100,
                cursor = secondPage.nextCursor
            )

        val fourthPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                size = 100,
                cursor = thirdPage.nextCursor
            )

        val clientIds =
            firstPage.content.map { it.clientId } +
                    secondPage.content.map { it.clientId } +
                    thirdPage.content.map { it.clientId } +
                    fourthPage.content.map { it.clientId }

        assertEquals(100, firstPage.content.size)
        assertEquals(100, secondPage.content.size)
        assertEquals(100, thirdPage.content.size)
        assertEquals(1, fourthPage.content.size)

        assertEquals(301, clientIds.size)
        assertEquals(301, clientIds.distinct().size)

        assertEquals("client-000", clientIds.first())
        assertEquals("client-300", clientIds.last())
    }

    @Test
    fun findPageRejectsInvalidCursor() {

        val repository = repository()

        val exception =
            assertThrows<IllegalArgumentException> {

                repository.findPage(
                    ownerId = "owner-${System.nanoTime()}",
                    yearMonth = "2026-09",
                    status = ClientType.RECEIVABLE,
                    size = 50,
                    cursor = "invalid-cursor"
                )
            }

        assertEquals(
            "Invalid cursor",
            exception.message
        )
    }


    private fun insertClients(
        repository: SummaryClientIndexRepository,
        ownerId: String,
        yearMonth: String,
        count: Int
    ) {

        val bucketIds =
            if (count <= 300) {
                listOf("bucket_000")
            } else {
                listOf("bucket_000", "bucket_001")
            }

        bucketIds.forEach { bucketId ->

            firestore
                .collection("owners")
                .document(ownerId)
                .collection("summary_client_index")
                .document(yearMonth)
                .collection("status")
                .document(ClientType.RECEIVABLE.name.lowercase())
                .collection("buckets")
                .document(bucketId)
                .set(
                    mapOf(
                        "capacity" to 300,
                        "size" to if (bucketId == "bucket_000") {
                            minOf(count, 300)
                        } else {
                            count - 300
                        }
                    )
                )
                .get()
        }

        repeat(count) { index ->

            val bucketId =
                if (index < 300) {
                    "bucket_000"
                } else {
                    "bucket_001"
                }

            repository.insert(
                ownerId = ownerId,
                yearMonth = yearMonth,
                bucketId = bucketId,
                index = SummaryClientIndex(
                    clientId = "client-%03d".format(index),
                    amount = index.toLong(),
                    status = ClientType.RECEIVABLE
                )
            )
        }
    }


    @Test
    fun `copy index preserves same bucket and client data`() {

        val ownerId = "owner-index-copy-test"
        val previousYearMonth = "2026-09"
        val newYearMonth = "2026-10"
        val status = ClientType.RECEIVABLE
        val bucketId = "bucket_000"

        val bucketRepository =
            SummaryClientIndexBucketRepository(
                firestore = firestore,
                properties = ClientLedgerProperties()
            )

        val indexRepository =
            SummaryClientIndexRepository(
                firestore = firestore
            )

        // Create source bucket.
        firestore
            .collection("owners")
            .document(ownerId)
            .collection("summary_client_index")
            .document(previousYearMonth)
            .collection("status")
            .document(status.name.lowercase())
            .collection("buckets")
            .document(bucketId)
            .set(
                mapOf(
                    "capacity" to 300L,
                    "size" to 2L
                )
            )
            .get()

        val clientA =
            SummaryClientIndex(
                clientId = "client-A",
                amount = 15_000,
                status = status
            )

        val clientB =
            SummaryClientIndex(
                clientId = "client-B",
                amount = 8_000,
                status = status
            )

        indexRepository.insert(
            ownerId = ownerId,
            yearMonth = previousYearMonth,
            bucketId = bucketId,
            index = clientA
        )

        indexRepository.insert(
            ownerId = ownerId,
            yearMonth = previousYearMonth,
            bucketId = bucketId,
            index = clientB
        )

        // Read source data.
        val sourceBucket =
            bucketRepository.find(
                ownerId = ownerId,
                yearMonth = previousYearMonth,
                status = status,
                bucketId = bucketId
            )

        val sourceClients =
            indexRepository.findAllInBucket(
                ownerId = ownerId,
                yearMonth = previousYearMonth,
                status = status,
                bucketId = bucketId
            )

        requireNotNull(sourceBucket)

        // Copy bucket + clients atomically.
        firestore
            .runTransaction { transaction ->

                bucketRepository.copyBucketInTransaction(
                    transaction = transaction,
                    ownerId = ownerId,
                    previousYearMonth = previousYearMonth,
                    newYearMonth = newYearMonth,
                    status = status,
                    bucketId = bucketId
                )

                sourceClients.forEach { index ->

                    indexRepository.copyInTransaction(
                        transaction = transaction,
                        ownerId = ownerId,
                        newYearMonth = newYearMonth,
                        status = status,
                        bucketId = bucketId,
                        index = index
                    )
                }

                null
            }
            .get()

        // Verify bucket ID was preserved.
        val newBucketIds =
            bucketRepository.findBucketIds(
                ownerId = ownerId,
                yearMonth = newYearMonth,
                status = status
            )

        assertEquals(
            listOf(bucketId),
            newBucketIds
        )

        // Verify bucket metadata was preserved.
        val newBucket =
            bucketRepository.find(
                ownerId = ownerId,
                yearMonth = newYearMonth,
                status = status,
                bucketId = bucketId
            )

        assertEquals(
            sourceBucket,
            newBucket
        )

        // Verify client index data was preserved.
        val newClients =
            indexRepository.findAllInBucket(
                ownerId = ownerId,
                yearMonth = newYearMonth,
                status = status,
                bucketId = bucketId
            )

        assertEquals(
            sourceClients,
            newClients
        )
    }
}
