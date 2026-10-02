package com.clientledger.core.service.client

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.*
import com.clientledger.core.pagination.PageResult
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.lock.OwnerOperationLockRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.service.lock.OwnerOperationLockService
import com.clientledger.core.transaction.FirestoreTransactionExecutor
import com.clientledger.core.utils.IdGenerator
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Service
import java.time.Clock
import java.time.YearMonth
import kotlin.math.abs

@Service
class ClientService(
    private val firestore: Firestore,
    private val transactionExecutor: FirestoreTransactionExecutor,
    private val clientRepository: ClientRepository,
    private val historyBucketRepository: ClientHistoryBucketRepository,
    private val historyRepository: ClientHistoryRepository,
    private val globalSummaryRepository: GlobalSummaryRepository,
    private val summaryIndexBucketRepository: SummaryClientIndexBucketRepository,
    private val summaryIndexRepository: SummaryClientIndexRepository,
    private val properties: ClientLedgerProperties,
    private val clock: Clock,
    private val ownerOperationLockService: OwnerOperationLockService,
    private val ownerOperationLockRepository: OwnerOperationLockRepository,
) {

    fun create(client: Client): Client {

        require(client.ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(client.name.isNotBlank()) {
            "client name must not be blank"
        }

        require(client.phone.isNotBlank()) {
            "client phone must not be blank"
        }

        val clientId = if (client.id.isBlank()) {
            IdGenerator.generateClientId()
        } else {
            client.id
        }

        val currentYearMonth = YearMonth.now(clock)

        val clientWithId = client.copy(
            id = clientId,
            latestAmount = client.initialOpeningBalance
        )

        return ownerOperationLockService.executeBusiness(
            ownerId = clientWithId.ownerId,
        ) { context, fence ->

            transactionExecutor.execute(context) { transaction, operationContext ->

                require(
                    ownerOperationLockRepository.isFenceValidInTransaction(
                        transaction = transaction,
                        ownerId = clientWithId.ownerId,
                        expectedFence = fence
                    )
                ) {
                    "Owner operation lock fence changed: ${clientWithId.ownerId}"
                }

                operationContext.checkDeadline()

                require(
                    !clientRepository.existsInTransaction(
                        transaction = transaction,
                        ownerId = clientWithId.ownerId,
                        clientId = clientId
                    )
                ) {
                    "Client already exists: $clientId"
                }

                val materializedMonths =
                    materializedMonths(currentYearMonth)

                val summaryStatus =
                    statusFromOpeningBalance(
                        clientWithId.initialOpeningBalance
                    )

                /*
             * READ / PLAN PHASE
             *
             * All transaction reads happen before any writes.
             */

                val historyBucketPlan =
                    historyBucketRepository.planBucketAllocationInTransaction(
                        transaction = transaction,
                        ownerId = clientWithId.ownerId,
                        yearMonth = currentYearMonth.toString()
                    )

                val globalSummaryPlans =
                    materializedMonths.map { yearMonth ->

                        globalSummaryRepository.planDeltaInTransaction(
                            transaction = transaction,
                            ownerId = clientWithId.ownerId,
                            yearMonth = yearMonth.toString(),
                            delta = globalSummaryDelta(
                                client = clientWithId,
                                yearMonth = yearMonth.toString()
                            )
                        )
                    }

                /*
             * Summary index bucket planning is done for every
             * materialized month.
             */
                val summaryIndexPlans =
                    materializedMonths.map { yearMonth ->

                        summaryIndexBucketRepository
                            .planBucketAllocationInTransaction(
                                transaction = transaction,
                                ownerId = clientWithId.ownerId,
                                yearMonth = yearMonth.toString(),
                                status = summaryStatus
                            )
                    }

                /*
             * WRITE / APPLY PHASE
             */

                historyBucketRepository.applyBucketAllocationInTransaction(
                    transaction = transaction,
                    plan = historyBucketPlan
                )

                createMonthlyHistories(
                    transaction = transaction,
                    client = clientWithId,
                    materializedMonths = materializedMonths,
                    bucketId = historyBucketPlan.bucketId
                )

                globalSummaryPlans.forEach { plan ->

                    globalSummaryRepository.applyDeltaPlanInTransaction(
                        transaction = transaction,
                        plan = plan
                    )
                }

                summaryIndexPlans.forEachIndexed { index, plan ->

                    val yearMonth = materializedMonths[index]

                    summaryIndexBucketRepository
                        .applyBucketAllocationInTransaction(
                            transaction = transaction,
                            plan = plan
                        )

                    summaryIndexRepository.insertInTransaction(
                        transaction = transaction,
                        ownerId = clientWithId.ownerId,
                        yearMonth = yearMonth.toString(),
                        bucketId = plan.bucketId,
                        index = SummaryClientIndex(
                            clientId = clientWithId.id,
                            amount = abs(
                                clientWithId.initialOpeningBalance
                            ),
                            status = summaryStatus
                        )
                    )
                }

                val finalClient =
                    clientWithId.copy(
                        bucketId = historyBucketPlan.bucketId,
                        type = summaryStatus
                    )

                operationContext.checkDeadline()

                clientRepository.createInTransaction(
                    transaction = transaction,
                    client = finalClient
                )
            }
        }
    }

    private fun materializedMonths(
        currentMonth: YearMonth
    ): List<YearMonth> {

        val editableMonths =
            properties.history.editableMonths

        require(editableMonths > 0) {
            "history.editable-months must be greater than zero"
        }

        return (editableMonths - 1 downTo 0)
            .map { offset ->
                currentMonth.minusMonths(offset.toLong())
            }
    }

    private fun createMonthlyHistories(
        transaction: Transaction,
        client: Client,
        materializedMonths: List<YearMonth>,
        bucketId: String
    ) {

        materializedMonths.forEach { yearMonth ->

            historyRepository.saveInTransaction(
                transaction = transaction,
                history = ClientMonthlyHistory(
                    clientId = client.id,
                    ownerId = client.ownerId,
                    yearMonth = yearMonth.toString(),
                    openingBalance =
                    client.initialOpeningBalance,
                    closingBalance =
                    client.initialOpeningBalance,
                    receivable =
                    if (client.initialOpeningBalance > 0) {
                        client.initialOpeningBalance
                    } else {
                        0
                    },
                    advance =
                    if (client.initialOpeningBalance < 0) {
                        abs(client.initialOpeningBalance)
                    } else {
                        0
                    },
                    status =
                    statusFromOpeningBalance(
                        client.initialOpeningBalance
                    )
                ),
                bucketId = bucketId
            )
        }
    }

    private fun globalSummaryDelta(
        client: Client,
        yearMonth: String
    ): GlobalSummary {

        val openingBalance =
            client.initialOpeningBalance

        return GlobalSummary(
            ownerId = client.ownerId,
            yearMonth = yearMonth,

            receivableClientCount =
            if (openingBalance > 0) 1 else 0,

            advanceClientCount =
            if (openingBalance < 0) 1 else 0,

            settledClientCount =
            if (openingBalance == 0L) 1 else 0,

            totalReceivableAmount =
            if (openingBalance > 0) {
                openingBalance
            } else {
                0
            },

            totalAdvanceAmount =
            if (openingBalance < 0) {
                abs(openingBalance)
            } else {
                0
            }
        )
    }

    private fun statusFromOpeningBalance(
        openingBalance: Long
    ): ClientType {

        return when {

            openingBalance > 0 ->
                ClientType.RECEIVABLE

            openingBalance < 0 ->
                ClientType.ADVANCE

            else ->
                ClientType.SETTLED
        }
    }

    fun findById(
        ownerId: String,
        clientId: String
    ): Client? {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        return clientRepository.findById(
            ownerId = ownerId,
            clientId = clientId
        )
    }

    fun findPage(
        ownerId: String,
        size: Int,
        cursor: String?
    ): PageResult<Client> {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        return clientRepository.findPage(
            ownerId = ownerId,
            size = size,
            cursor = cursor
        )
    }
}package com.clientledger.core.service.order

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.*
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.lock.OwnerOperationLockRepository
import com.clientledger.core.repository.order.OrderRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexDeletePlan
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.repository.summary.SummaryClientIndexUpsertPlan
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
    private val summaryIndexBucketRepository: com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository,
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
                 * READ / PLAN
                 *
                 * Build SummaryClientIndex changes for every
                 * affected month.
                 *
                 * IMPORTANT:
                 * No writes happen inside this method.
                 */
                val summaryIndexPlans =
                    planSummaryIndexChanges(
                        transaction = transaction,
                        client = client,
                        propagationPlan = propagationPlan
                    )

                operationContext.checkDeadline()

                /*
                 * WRITE / APPLY
                 *
                 * All transaction reads are now complete.
                 */

                /*
                 * 1. Apply updated monthly histories.
                 */
                propagationPlan.updatedHistories.values.forEach { history ->

                    historyRepository.saveInTransaction(
                        transaction = transaction,
                        history = history,
                        bucketId = client.bucketId
                    )
                }

                /*
                 * 2. Apply GlobalSummary changes.
                 */
                globalSummaryPlans.forEach { plan ->

                    globalSummaryRepository.applyDeltaPlanInTransaction(
                        transaction = transaction,
                        plan = plan
                    )
                }

                /*
                 * 3. Apply SummaryClientIndex changes.
                 *
                 * Same client.bucketId is used.
                 */
                summaryIndexPlans.forEach { plan ->

                    plan.deletePlan?.let { deletePlan ->

                        summaryIndexRepository.applyDeletePlanInTransaction(
                            transaction = transaction,
                            plan = deletePlan
                        )
                    }

                    plan.upsertPlan?.let { upsertPlan ->

                        summaryIndexRepository.applyUpsertPlanInTransaction(
                            transaction = transaction,
                            plan = upsertPlan
                        )
                    }
                }

                operationContext.checkDeadline()

                /*
                 * 4. Finally persist the source Order.
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

    private fun planSummaryIndexChanges(
        transaction: Transaction,
        client: Client,
        propagationPlan: HistoryPropagationPlan
    ): List<SummaryIndexChangePlan> {

        val capacity =
            properties.summary.indexBucketCapacity.toLong()

        require(capacity > 0) {
            "summary.indexBucketCapacity must be greater than zero"
        }

        return propagationPlan.months.map { yearMonth ->

            val updatedHistory =
                propagationPlan.updatedHistories[yearMonth]
                    ?: throw IllegalStateException(
                        "Updated history not found for $yearMonth"
                    )

            val previousHistory =
                propagationPlan.previousHistories[yearMonth]

            val oldStatus =
                previousHistory?.status

            val newStatus =
                updatedHistory.status

            /*
             * Find the existing index only when an old history exists.
             *
             * We use client.bucketId directly.
             * No bucket scan is performed.
             */
            val oldLocation =
                if (oldStatus != null) {

                    summaryIndexRepository.findInTransaction(
                        transaction = transaction,
                        ownerId = client.ownerId,
                        yearMonth = yearMonth.toString(),
                        status = oldStatus,
                        bucketId = client.bucketId,
                        clientId = client.id
                    )

                } else {
                    null
                }

            /*
             * If an existing history has an index, that index must exist.
             *
             * The index is a materialized representation of the history.
             */
            if (oldStatus != null && oldLocation == null) {

                throw IllegalStateException(
                    "Summary client index not found for " +
                            "client=${client.id}, " +
                            "yearMonth=$yearMonth, " +
                            "status=$oldStatus, " +
                            "bucketId=${client.bucketId}"
                )
            }

            val newIndex =
                SummaryClientIndex(
                    clientId = client.id,
                    amount =
                    when (newStatus) {

                        ClientType.RECEIVABLE ->
                            updatedHistory.receivable

                        ClientType.ADVANCE ->
                            updatedHistory.advance

                        ClientType.SETTLED ->
                            0
                    },
                    status = newStatus
                )

            /*
             * Same status:
             *
             * Existing index is updated in-place.
             *
             * No bucket size change.
             */
            if (
                oldLocation != null &&
                oldStatus == newStatus
            ) {

                val updatePlan =
                    summaryIndexRepository.planUpsertInTransaction(
                        transaction = transaction,
                        ownerId = client.ownerId,
                        yearMonth = yearMonth.toString(),
                        bucketId = client.bucketId,
                        index = newIndex,
                        capacity = capacity
                    )

                return@map SummaryIndexChangePlan(
                    deletePlan = null,
                    upsertPlan = updatePlan
                )
            }

            /*
             * Status changed:
             *
             * 1. Remove old index from old status bucket.
             * 2. Insert new index into new status bucket.
             *
             * The bucketId remains the same.
             */
            val deletePlan =
                oldLocation?.let { location ->

                    summaryIndexRepository.planDeleteInTransaction(
                        transaction = transaction,
                        location = location,
                        ownerId = client.ownerId,
                        yearMonth = yearMonth.toString(),
                        status = oldStatus
                            ?: throw IllegalStateException(
                                "Old status missing for existing index"
                            )
                    )
                }

            /*
             * No previous index means this is a newly materialized
             * month. We insert the new index.
             *
             * If status changed, we also insert into the target status.
             */
            val upsertPlan =
                summaryIndexRepository.planUpsertInTransaction(
                    transaction = transaction,
                    ownerId = client.ownerId,
                    yearMonth = yearMonth.toString(),
                    bucketId = client.bucketId,
                    index = newIndex,
                    capacity = capacity
                )

            SummaryIndexChangePlan(
                deletePlan = deletePlan,
                upsertPlan = upsertPlan
            )
        }
    }

    private data class SummaryIndexChangePlan(
        val deletePlan: SummaryClientIndexDeletePlan?,
        val upsertPlan: SummaryClientIndexUpsertPlan?
    )
}package com.clientledger.core.service.monthlyrollover

import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.MonthlyRolloverState
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.lock.OwnerOperationLockRepository
import com.clientledger.core.repository.monthlyrollover.MonthlyRolloverRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.service.lock.OwnerOperationLockService
import com.clientledger.core.transaction.FirestoreTransactionExecutor
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service
import java.time.Clock
import java.time.YearMonth

@Service
class PerOwnerMonthlyRolloverService(
    private val clientRepository: ClientRepository,
    private val clientHistoryRepository: ClientHistoryRepository,
    private val clientHistoryBucketRepository: ClientHistoryBucketRepository,
    private val globalSummaryRepository: GlobalSummaryRepository,
    private val summaryClientIndexRepository: SummaryClientIndexRepository,
    private val summaryClientIndexBucketRepository:
    SummaryClientIndexBucketRepository,
    private val monthlyRolloverRepository: MonthlyRolloverRepository,
    private val ownerOperationLockRepository:
    OwnerOperationLockRepository,
    private val ownerOperationLockService:
    OwnerOperationLockService,
    private val transactionExecutor:
    FirestoreTransactionExecutor,
    private val firestore: Firestore,
    private val clock: Clock
) {

    private val logger =
        LoggerFactory.getLogger(
            PerOwnerMonthlyRolloverService::class.java
        )

    companion object {
        private const val OWNER_ROLLOVER_RETRY_COUNT = 3
        private const val HISTORY_CLIENT_PAGE_SIZE = 100
    }

    fun rollover(
        ownerId: String,
        previousYearMonth: YearMonth,
        newYearMonth: YearMonth
    ) {
        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(
            newYearMonth == previousYearMonth.plusMonths(1)
        ) {
            "newYearMonth must immediately follow previousYearMonth"
        }

        val previousMonth =
            previousYearMonth.toString()

        val currentMonth =
            newYearMonth.toString()

        ownerOperationLockService.executeLongRunning(ownerId) { fence ->

            ensurePending(
                ownerId = ownerId,
                yearMonth = currentMonth
            )

            val existing =
                monthlyRolloverRepository.find(
                    ownerId = ownerId,
                    yearMonth = currentMonth
                )

            if (
                existing?.status ==
                MonthlyRolloverState.Status.COMPLETED
            ) {
                logger.info(
                    "Monthly rollover already completed: ownerId={}, yearMonth={}",
                    ownerId,
                    currentMonth
                )

                return@executeLongRunning
            }

            var lastException: Exception? = null

            /*
             * Initial execution + 3 retries.
             *
             * Therefore the maximum number of complete
             * owner rollover executions is 4.
             */
            for (attempt in 1..(OWNER_ROLLOVER_RETRY_COUNT + 1)) {

                try {

                    logger.info(
                        "[MONTHLY-ROLLOVER] Owner rollover attempt | ownerId={} | yearMonth={} | attempt={}/{}",
                        ownerId,
                        currentMonth,
                        attempt,
                        OWNER_ROLLOVER_RETRY_COUNT + 1
                    )

                    monthlyRolloverRepository.markRunning(
                        ownerId = ownerId,
                        yearMonth = currentMonth,
                        startedAt = clock.instant()
                    )

                    copyHistory(
                        ownerId = ownerId,
                        previousMonth = previousMonth,
                        currentMonth = currentMonth,
                        fence = fence
                    )

                    copyGlobalSummary(
                        ownerId = ownerId,
                        previousMonth = previousMonth,
                        currentMonth = currentMonth,
                        fence = fence
                    )

                    copySummaryIndexes(
                        ownerId = ownerId,
                        previousMonth = previousMonth,
                        currentMonth = currentMonth,
                        fence = fence
                    )

                    monthlyRolloverRepository.markCompleted(
                        ownerId = ownerId,
                        yearMonth = currentMonth,
                        completedAt = clock.instant()
                    )

                    /*
                     * markCompleted() already removes the owner
                     * from the failed-owner retry index atomically.
                     */
                    logger.info(
                        "[MONTHLY-ROLLOVER] Owner rollover completed | ownerId={} | yearMonth={} | attempt={}",
                        ownerId,
                        currentMonth,
                        attempt
                    )

                    return@executeLongRunning

                } catch (exception: Exception) {

                    lastException = exception

                    logger.error(
                        "[MONTHLY-ROLLOVER] Owner rollover attempt failed | ownerId={} | yearMonth={} | attempt={}/{}",
                        ownerId,
                        currentMonth,
                        attempt,
                        OWNER_ROLLOVER_RETRY_COUNT + 1,
                        exception
                    )

                    /*
                     * No explicit continue is required.
                     * The for-loop automatically performs
                     * the next retry.
                     */
                }
            }

            /*
             * All 4 executions failed.
             *
             * markFailed() atomically:
             *
             * 1. Marks monthly rollover as FAILED.
             * 2. Creates the failed-owner retry record.
             *
             * Therefore the owner is now discoverable by
             * FailedOwnerRecoveryService.
             */
            try {

                monthlyRolloverRepository.markFailed(
                    ownerId = ownerId,
                    yearMonth = currentMonth,
                    failedAt = clock.instant(),
                    error = lastException?.message
                )

            } catch (stateException: Exception) {

                logger.error(
                    "Failed to mark monthly rollover as FAILED: ownerId={}, yearMonth={}",
                    ownerId,
                    currentMonth,
                    stateException
                )
            }

            /*
             * The original rollover exception is propagated
             * so the global orchestrator can record this owner
             * as failed without stopping other owners.
             */
            throw lastException
                ?: IllegalStateException(
                    "Monthly rollover failed: ownerId=$ownerId, yearMonth=$currentMonth"
                )
        }
    }

    private fun ensurePending(
        ownerId: String,
        yearMonth: String
    ) {
        if (
            monthlyRolloverRepository.find(
                ownerId,
                yearMonth
            ) == null
        ) {
            monthlyRolloverRepository.createPending(
                ownerId = ownerId,
                yearMonth = yearMonth
            )
        }
    }

    private fun copyHistory(
        ownerId: String,
        previousMonth: String,
        currentMonth: String,
        fence: Long
    ) {
        var cursor: String? = null

        while (true) {

            val page =
                clientRepository.findPage(
                    ownerId = ownerId,
                    size = HISTORY_CLIENT_PAGE_SIZE,
                    cursor = cursor
                )

            if (page.content.isEmpty()) {
                break
            }

            page.content.forEach { client ->

                val previousHistory =
                    clientHistoryRepository.find(
                        ownerId = ownerId,
                        yearMonth = previousMonth,
                        bucketId = client.bucketId,
                        clientId = client.id
                    ) ?: return@forEach

                transactionExecutor.execute { transaction ->

                    requireFence(
                        transaction = transaction,
                        ownerId = ownerId,
                        fence = fence
                    )

                    clientHistoryRepository.copyToNewMonthInTransaction(
                        transaction = transaction,
                        previousHistory = previousHistory,
                        bucketId = client.bucketId,
                        newYearMonth = currentMonth
                    )
                }
            }

            cursor = page.nextCursor

            if (cursor == null) {
                break
            }
        }

        /*
         * Bucket metadata must be copied only once.
         *
         * Client pagination and history-bucket pagination
         * are independent.
         */
        copyHistoryBuckets(
            ownerId = ownerId,
            previousMonth = previousMonth,
            currentMonth = currentMonth,
            fence = fence
        )
    }

    private fun copyHistoryBuckets(
        ownerId: String,
        previousMonth: String,
        currentMonth: String,
        fence: Long
    ) {
        val bucketIds =
            clientHistoryBucketRepository.findBucketIds(
                ownerId = ownerId,
                yearMonth = previousMonth
            )

        bucketIds.forEach { bucketId ->

            transactionExecutor.execute { transaction ->

                requireFence(
                    transaction = transaction,
                    ownerId = ownerId,
                    fence = fence
                )

                clientHistoryBucketRepository.copyBucketInTransaction(
                    transaction = transaction,
                    ownerId = ownerId,
                    previousYearMonth = previousMonth,
                    newYearMonth = currentMonth,
                    bucketId = bucketId
                )
            }
        }
    }

    private fun copyGlobalSummary(
        ownerId: String,
        previousMonth: String,
        currentMonth: String,
        fence: Long
    ) {
        val previousSummary =
            globalSummaryRepository.find(
                ownerId = ownerId,
                yearMonth = previousMonth
            ) ?: return

        transactionExecutor.execute { transaction ->

            requireFence(
                transaction = transaction,
                ownerId = ownerId,
                fence = fence
            )

            globalSummaryRepository.copyToNewMonthInTransaction(
                transaction = transaction,
                previousSummary = previousSummary,
                newYearMonth = currentMonth
            )
        }
    }

    private fun copySummaryIndexes(
        ownerId: String,
        previousMonth: String,
        currentMonth: String,
        fence: Long
    ) {
        ClientType.entries.forEach { clientType ->

            val bucketIds =
                summaryClientIndexBucketRepository.findBucketIds(
                    ownerId = ownerId,
                    yearMonth = previousMonth,
                    status = clientType
                )

            bucketIds.forEach { bucketId ->

                val indexes =
                    summaryClientIndexRepository.findAllInBucket(
                        ownerId = ownerId,
                        yearMonth = previousMonth,
                        status = clientType,
                        bucketId = bucketId
                    )

                transactionExecutor.execute { transaction ->

                    requireFence(
                        transaction = transaction,
                        ownerId = ownerId,
                        fence = fence
                    )

                    summaryClientIndexBucketRepository
                        .copyBucketInTransaction(
                            transaction = transaction,
                            ownerId = ownerId,
                            previousYearMonth = previousMonth,
                            newYearMonth = currentMonth,
                            status = clientType,
                            bucketId = bucketId
                        )

                    indexes.forEach { index ->

                        summaryClientIndexRepository
                            .copyInTransaction(
                                transaction = transaction,
                                ownerId = ownerId,
                                newYearMonth = currentMonth,
                                status = clientType,
                                bucketId = bucketId,
                                index = index
                            )
                    }
                }
            }
        }
    }

    private fun requireFence(
        transaction: Transaction,
        ownerId: String,
        fence: Long
    ) {
        require(
            ownerOperationLockRepository
                .isFenceValidInTransaction(
                    transaction = transaction,
                    ownerId = ownerId,
                    expectedFence = fence
                )
        ) {
            "Owner operation lock fence changed: $ownerId"
        }
    }
}
package com.clientledger.core.service.summary

import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.GlobalSummary
import com.clientledger.core.domain.SummaryClientIndex
import com.clientledger.core.pagination.PageResult
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import org.springframework.stereotype.Service

@Service
class SummaryService(
    private val globalSummaryRepository: GlobalSummaryRepository,
    private val summaryClientIndexRepository: SummaryClientIndexRepository
) {

    fun getMonthSummary(
        ownerId: String,
        yearMonth: String
    ): GlobalSummary? {

        return globalSummaryRepository.find(
            ownerId = ownerId,
            yearMonth = yearMonth
        )
    }

    fun getYearSummary(
        ownerId: String,
        year: Int
    ): GlobalSummary {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(year in 1..9999) {
            "year must be between 1 and 9999"
        }

        var lastAvailableSummary: GlobalSummary? = null

        var totalCashFlow = 0L
        var totalNetProfit = 0L
        var totalInvoiceAmount = 0L
        var totalInvoiceAmountWithGst = 0L
        var totalGstAmount = 0L
        var totalExpenseAmount = 0L
        var totalPaymentAmount = 0L

        for (month in 1..12) {

            val yearMonth =
                "%04d-%02d".format(year, month)

            val summary =
                globalSummaryRepository.find(
                    ownerId = ownerId,
                    yearMonth = yearMonth
                )
                    ?: continue

            lastAvailableSummary = summary

            totalCashFlow += summary.cashFlow
            totalNetProfit += summary.netProfit
            totalInvoiceAmount += summary.totalInvoiceAmount
            totalInvoiceAmountWithGst += summary.totalInvoiceAmountWithGst
            totalGstAmount += summary.totalGstAmount
            totalExpenseAmount += summary.totalExpenseAmount
            totalPaymentAmount += summary.totalPaymentAmount
        }

        val lastSummary = lastAvailableSummary
            ?: return GlobalSummary(
                ownerId = ownerId,
                yearMonth = year.toString()
            )

        return GlobalSummary(
            ownerId = ownerId,
            yearMonth = year.toString(),

            receivableClientCount =
            lastSummary.receivableClientCount,

            advanceClientCount =
            lastSummary.advanceClientCount,

            settledClientCount =
            lastSummary.settledClientCount,

            totalReceivableAmount =
            lastSummary.totalReceivableAmount,

            totalAdvanceAmount =
            lastSummary.totalAdvanceAmount,

            cashFlow = totalCashFlow,
            netProfit = totalNetProfit,

            totalInvoiceAmount =
            totalInvoiceAmount,

            totalInvoiceAmountWithGst =
            totalInvoiceAmountWithGst,

            totalGstAmount =
            totalGstAmount,

            totalExpenseAmount =
            totalExpenseAmount,

            totalPaymentAmount =
            totalPaymentAmount
        )
    }

    fun getClients(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        size: Int,
        cursor: String?
    ): PageResult<SummaryClientIndex> {

        require(
            status == ClientType.RECEIVABLE ||
                    status == ClientType.ADVANCE
        ) {
            "status must be RECEIVABLE or ADVANCE"
        }

        return summaryClientIndexRepository.findPage(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status,
            size = size,
            cursor = cursor
        )
    }

}
