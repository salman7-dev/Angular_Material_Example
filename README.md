package com.clientledger.core.service.order

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.*
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.lock.OwnerOperationLockRepository
import com.clientledger.core.repository.order.OrderRepository
import com.clientledger.core.repository.summary.*
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

            val deletePlan =
                if (
                    oldLocation != null &&
                    oldStatus != newStatus
                ) {

                    summaryIndexRepository.planDeleteInTransaction(
                        transaction = transaction,
                        location = oldLocation,
                        ownerId = client.ownerId,
                        yearMonth = yearMonth.toString(),
                        status = oldStatus!!
                    )

                } else {
                    null
                }

            val newIndex =
                SummaryClientIndex(
                    clientId = client.id,
                    amount = when (newStatus) {
                        ClientType.RECEIVABLE ->
                            updatedHistory.receivable

                        ClientType.ADVANCE ->
                            updatedHistory.advance

                        ClientType.SETTLED ->
                            0
                    },
                    status = newStatus
                )

            val upsertPlan =
                if (
                    oldLocation == null ||
                    oldStatus != newStatus
                ) {

                    summaryIndexRepository.planUpsertInTransaction(
                        transaction = transaction,
                        ownerId = client.ownerId,
                        yearMonth = yearMonth.toString(),
                        bucketId = client.bucketId,
                        index = newIndex,
                        capacity = capacity
                    )

                } else {

                    summaryIndexRepository.planUpsertInTransaction(
                        transaction = transaction,
                        ownerId = client.ownerId,
                        yearMonth = yearMonth.toString(),
                        bucketId = client.bucketId,
                        index = newIndex,
                        capacity = capacity
                    )
                }

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
}

//package com.clientledger.core.repository.summary
//
//import com.clientledger.core.domain.ClientType
//import com.clientledger.core.domain.SummaryClientIndex
//import com.clientledger.core.pagination.PageCursor
//import com.clientledger.core.pagination.PageResult
//import com.google.cloud.firestore.*
//import org.springframework.stereotype.Repository
//import java.util.Base64
//
//@Repository
//class SummaryClientIndexRepository(
//    private val firestore: Firestore
//) {
//
//    private fun statusReference(
//        ownerId: String,
//        yearMonth: String,
//        status: ClientType
//    ): DocumentReference {
//
//        require(ownerId.isNotBlank()) {
//            "ownerId must not be blank"
//        }
//
//        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
//            "yearMonth must be in yyyy-MM format"
//        }
//
//        return firestore
//            .collection("owners")
//            .document(ownerId)
//            .collection("summary_client_index")
//            .document(yearMonth)
//            .collection("status")
//            .document(status.name.lowercase())
//    }
//
//    private fun bucketCollection(
//        ownerId: String,
//        yearMonth: String,
//        status: ClientType
//    ): CollectionReference {
//
//        return statusReference(
//            ownerId = ownerId,
//            yearMonth = yearMonth,
//            status = status
//        )
//            .collection("buckets")
//    }
//
//    private fun clientReference(
//        ownerId: String,
//        yearMonth: String,
//        status: ClientType,
//        bucketId: String,
//        clientId: String
//    ): DocumentReference {
//
//        require(bucketId.isNotBlank()) {
//            "bucketId must not be blank"
//        }
//
//        require(clientId.isNotBlank()) {
//            "clientId must not be blank"
//        }
//
//        return bucketCollection(
//            ownerId = ownerId,
//            yearMonth = yearMonth,
//            status = status
//        )
//            .document(bucketId)
//            .collection("clients")
//            .document(clientId)
//    }
//
//    fun insert(
//        ownerId: String,
//        yearMonth: String,
//        bucketId: String,
//        index: SummaryClientIndex
//    ): SummaryClientIndex {
//
//        val reference = clientReference(
//            ownerId = ownerId,
//            yearMonth = yearMonth,
//            status = index.status,
//            bucketId = bucketId,
//            clientId = index.clientId
//        )
//
//        reference
//            .set(toFirestoreMap(index))
//            .get()
//
//        return index
//    }
//
//    fun insertInTransaction(
//        transaction: Transaction,
//        ownerId: String,
//        yearMonth: String,
//        bucketId: String,
//        index: SummaryClientIndex
//    ): SummaryClientIndex {
//
//        val reference = clientReference(
//            ownerId = ownerId,
//            yearMonth = yearMonth,
//            status = index.status,
//            bucketId = bucketId,
//            clientId = index.clientId
//        )
//
//        transaction.set(
//            reference,
//            toFirestoreMap(index)
//        )
//
//        return index
//    }
//
//    fun find(
//        ownerId: String,
//        yearMonth: String,
//        status: ClientType,
//        bucketId: String,
//        clientId: String
//    ): SummaryClientIndex? {
//
//        val reference = clientReference(
//            ownerId = ownerId,
//            yearMonth = yearMonth,
//            status = status,
//            bucketId = bucketId,
//            clientId = clientId
//        )
//
//        val snapshot = reference
//            .get()
//            .get()
//
//        if (!snapshot.exists()) {
//            return null
//        }
//
//        return toIndex(snapshot)
//    }
//
//    fun findPage(
//        ownerId: String,
//        yearMonth: String,
//        status: ClientType,
//        size: Int,
//        cursor: String?
//    ): PageResult<SummaryClientIndex> {
//
//        require(ownerId.isNotBlank()) {
//            "ownerId must not be blank"
//        }
//
//        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
//            "yearMonth must be in yyyy-MM format"
//        }
//
//        require(size > 0) {
//            "size must be greater than zero"
//        }
//
//        val pageCursor = decodeCursor(cursor)
//
//        var currentBucketId = pageCursor?.bucketId
//        var currentClientId = pageCursor?.clientId
//
//        val result = mutableListOf<SummaryClientIndex>()
//
//        while (result.size < size) {
//
//            val bucketId =
//                currentBucketId
//                    ?: findFirstBucketId(
//                        ownerId = ownerId,
//                        yearMonth = yearMonth,
//                        status = status
//                    )
//
//            if (bucketId == null) {
//                break
//            }
//
//            val remaining = size - result.size
//
//            val clientCollection =
//                bucketCollection(
//                    ownerId = ownerId,
//                    yearMonth = yearMonth,
//                    status = status
//                )
//                    .document(bucketId)
//                    .collection("clients")
//
//            val clientQuery =
//                clientCollection
//                    .orderBy("__name__")
//                    .let { query ->
//
//                        if (
//                            currentBucketId == bucketId &&
//                            currentClientId != null
//                        ) {
//                            query.startAfter(currentClientId)
//                        } else {
//                            query
//                        }
//                    }
//                    .limit(remaining + 1)
//
//            val clientSnapshots =
//                clientQuery
//                    .get()
//                    .get()
//                    .documents
//
//            val hasMoreInCurrentBucket =
//                clientSnapshots.size > remaining
//
//            val documentsToReturn =
//                if (hasMoreInCurrentBucket) {
//                    clientSnapshots.take(remaining)
//                } else {
//                    clientSnapshots
//                }
//
//            result.addAll(
//                documentsToReturn.map(::toIndex)
//            )
//
//            if (hasMoreInCurrentBucket) {
//
//                val lastDocument =
//                    documentsToReturn.last()
//
//                return PageResult(
//                    content = result,
//                    hasNext = true,
//                    nextCursor = encodeCursor(
//                        PageCursor(
//                            bucketId = bucketId,
//                            clientId = lastDocument.id
//                        )
//                    )
//                )
//            }
//
//            val nextBucketId =
//                findNextBucketId(
//                    ownerId = ownerId,
//                    yearMonth = yearMonth,
//                    status = status,
//                    bucketId = bucketId
//                )
//
//            if (result.size == size) {
//
//                val lastDocument =
//                    documentsToReturn.lastOrNull()
//
//                if (nextBucketId != null && lastDocument != null) {
//                    return PageResult(
//                        content = result,
//                        hasNext = true,
//                        nextCursor = encodeCursor(
//                            PageCursor(
//                                bucketId = bucketId,
//                                clientId = lastDocument.id
//                            )
//                        )
//                    )
//                }
//
//                return PageResult(
//                    content = result,
//                    hasNext = false,
//                    nextCursor = null
//                )
//            }
//
//            if (nextBucketId == null) {
//                break
//            }
//
//            currentBucketId = nextBucketId
//            currentClientId = null
//        }
//
//        return PageResult(
//            content = result,
//            hasNext = false,
//            nextCursor = null
//        )
//    }
//
//    private fun findFirstBucketId(
//        ownerId: String,
//        yearMonth: String,
//        status: ClientType
//    ): String? {
//
//        return bucketCollection(
//            ownerId = ownerId,
//            yearMonth = yearMonth,
//            status = status
//        )
//            .orderBy("__name__")
//            .limit(1)
//            .get()
//            .get()
//            .documents
//            .firstOrNull()
//            ?.id
//    }
//
//    private fun findNextBucketId(
//        ownerId: String,
//        yearMonth: String,
//        status: ClientType,
//        bucketId: String
//    ): String? {
//
//        return bucketCollection(
//            ownerId = ownerId,
//            yearMonth = yearMonth,
//            status = status
//        )
//            .orderBy("__name__")
//            .startAfter(bucketId)
//            .limit(1)
//            .get()
//            .get()
//            .documents
//            .firstOrNull()
//            ?.id
//    }
//
//    private fun encodeCursor(
//        cursor: PageCursor
//    ): String {
//
//        val raw =
//            "${cursor.bucketId}|${cursor.clientId}"
//
//        return Base64
//            .getUrlEncoder()
//            .withoutPadding()
//            .encodeToString(
//                raw.toByteArray()
//            )
//    }
//
//    private fun decodeCursor(
//        cursor: String?
//    ): PageCursor? {
//
//        if (cursor.isNullOrBlank()) {
//            return null
//        }
//
//        return try {
//
//            val decoded =
//                String(
//                    Base64
//                        .getUrlDecoder()
//                        .decode(cursor)
//                )
//
//            val parts = decoded.split("|")
//
//            require(parts.size == 2)
//            require(parts[0].isNotBlank())
//            require(parts[1].isNotBlank())
//
//            PageCursor(
//                bucketId = parts[0],
//                clientId = parts[1]
//            )
//
//        } catch (exception: Exception) {
//
//            throw IllegalArgumentException(
//                "Invalid cursor",
//                exception
//            )
//        }
//    }
//
//    private fun toFirestoreMap(
//        index: SummaryClientIndex
//    ): Map<String, Any> {
//
//        return mapOf(
//            "clientId" to index.clientId,
//            "amount" to index.amount,
//            "status" to index.status.name
//        )
//    }
//
//    private fun toIndex(
//        snapshot: DocumentSnapshot
//    ): SummaryClientIndex {
//
//        return SummaryClientIndex(
//            clientId =
//            snapshot.getString("clientId")
//                ?: snapshot.id,
//
//            amount =
//            snapshot.getLong("amount")
//                ?: 0,
//
//            status =
//            snapshot.getString("status")
//                ?.let { ClientType.valueOf(it) }
//                ?: ClientType.SETTLED
//        )
//    }
//
//    fun findAllInBucket(
//        ownerId: String,
//        yearMonth: String,
//        status: ClientType,
//        bucketId: String
//    ): List<SummaryClientIndex> {
//
//        require(ownerId.isNotBlank()) {
//            "ownerId must not be blank"
//        }
//
//        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
//            "yearMonth must be in yyyy-MM format"
//        }
//
//        require(bucketId.isNotBlank()) {
//            "bucketId must not be blank"
//        }
//
//        val clientCollection =
//            bucketCollection(
//                ownerId = ownerId,
//                yearMonth = yearMonth,
//                status = status
//            )
//                .document(bucketId)
//                .collection("clients")
//
//        return clientCollection
//            .orderBy("__name__")
//            .get()
//            .get()
//            .documents
//            .map(::toIndex)
//    }
//
//    fun copyInTransaction(
//        transaction: Transaction,
//        ownerId: String,
//        newYearMonth: String,
//        status: ClientType,
//        bucketId: String,
//        index: SummaryClientIndex
//    ): SummaryClientIndex {
//
//        require(ownerId.isNotBlank()) {
//            "ownerId must not be blank"
//        }
//
//        require(newYearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
//            "newYearMonth must be in yyyy-MM format"
//        }
//
//        require(bucketId.isNotBlank()) {
//            "bucketId must not be blank"
//        }
//
//        require(index.clientId.isNotBlank()) {
//            "clientId must not be blank"
//        }
//
//        require(index.status == status) {
//            "Index status does not match bucket status"
//        }
//
//        val targetReference =
//            bucketCollection(
//                ownerId = ownerId,
//                yearMonth = newYearMonth,
//                status = status
//            )
//                .document(bucketId)
//                .collection("clients")
//                .document(index.clientId)
//
//        transaction.set(
//            targetReference,
//            mapOf(
//                "clientId" to index.clientId,
//                "amount" to index.amount,
//                "status" to index.status.name
//            )
//        )
//
//        return index
//    }
//}

package com.clientledger.core.repository.summary

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.SummaryClientIndex
import com.clientledger.core.pagination.PageCursor
import com.clientledger.core.pagination.PageResult
import com.google.cloud.firestore.*
import org.springframework.stereotype.Repository
import java.util.Base64

data class SummaryClientIndexLocation(
    val bucketId: String,
    val index: SummaryClientIndex,
    val bucketExists: Boolean,
    val bucketSize: Long
)

data class SummaryClientIndexUpsertPlan(
    val clientReference: DocumentReference,
    val bucketReference: DocumentReference,
    val index: SummaryClientIndex,
    val bucketExists: Boolean,
    val currentBucketSize: Long,
    val capacity: Long
)

data class SummaryClientIndexDeletePlan(
    val clientReference: DocumentReference,
    val bucketReference: DocumentReference,
    val currentBucketSize: Long
)

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

    private fun bucketReference(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String
    ): DocumentReference {

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status
        )
            .document(bucketId)
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

        return bucketReference(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status,
            bucketId = bucketId
        )
            .collection("clients")
            .document(clientId)
    }

    fun insert(
        ownerId: String,
        yearMonth: String,
        bucketId: String,
        index: SummaryClientIndex
    ): SummaryClientIndex {

        val reference =
            clientReference(
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

        val reference =
            clientReference(
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

        val reference =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status,
                bucketId = bucketId,
                clientId = clientId
            )

        val snapshot =
            reference
                .get()
                .get()

        if (!snapshot.exists()) {
            return null
        }

        return toIndex(snapshot)
    }

    /**
     * Finds the index in the exact bucket supplied by the client.
     *
     * No bucket scan is required because Client.bucketId is the
     * source of truth for the current architecture.
     */
    fun findInTransaction(
        transaction: Transaction,
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String,
        clientId: String
    ): SummaryClientIndexLocation? {

        val clientRef =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status,
                bucketId = bucketId,
                clientId = clientId
            )

        val bucketRef =
            bucketReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status,
                bucketId = bucketId
            )

        val clientSnapshot =
            transaction
                .get(clientRef)
                .get()

        if (!clientSnapshot.exists()) {
            return null
        }

        val bucketSnapshot =
            transaction
                .get(bucketRef)
                .get()

        val bucketSize =
            bucketSnapshot
                .getLong("size")
                ?: 0L

        return SummaryClientIndexLocation(
            bucketId = bucketId,
            index = toIndex(clientSnapshot),
            bucketExists = bucketSnapshot.exists(),
            bucketSize = bucketSize
        )
    }

    /**
     * Reads the target bucket and target client index.
     *
     * This method is READ / PLAN only.
     * It performs no writes.
     */
    fun planUpsertInTransaction(
        transaction: Transaction,
        ownerId: String,
        yearMonth: String,
        bucketId: String,
        index: SummaryClientIndex,
        capacity: Long
    ): SummaryClientIndexUpsertPlan {

        require(capacity > 0) {
            "capacity must be greater than zero"
        }

        val clientRef =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = index.status,
                bucketId = bucketId,
                clientId = index.clientId
            )

        val bucketRef =
            bucketReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = index.status,
                bucketId = bucketId
            )

        val clientSnapshot =
            transaction
                .get(clientRef)
                .get()

        val bucketSnapshot =
            transaction
                .get(bucketRef)
                .get()

        val currentBucketSize =
            bucketSnapshot
                .getLong("size")
                ?: 0L

        return SummaryClientIndexUpsertPlan(
            clientReference = clientRef,
            bucketReference = bucketRef,
            index = index,
            bucketExists = bucketSnapshot.exists(),
            currentBucketSize = currentBucketSize,
            capacity = capacity
        )
    }

    /**
     * WRITE / APPLY for a planned upsert.
     *
     * Existing index:
     *   update only the index document.
     *
     * Missing index:
     *   create the index and increment/create bucket metadata.
     */
    fun applyUpsertPlanInTransaction(
        transaction: Transaction,
        plan: SummaryClientIndexUpsertPlan
    ): SummaryClientIndex {

        if (!plan.bucketExists) {

            transaction.set(
                plan.bucketReference,
                mapOf(
                    "capacity" to plan.capacity,
                    "size" to 1L
                )
            )

        } else if (plan.currentBucketSize <= plan.capacity) {

            /*
             * If the index already exists, the bucket size must not
             * increase. The caller uses the existence information from
             * the READ / PLAN phase.
             *
             * This method is intended for the create/missing-index case.
             */
        }

        transaction.set(
            plan.clientReference,
            toFirestoreMap(plan.index)
        )

        return plan.index
    }

    /**
     * Creates a plan for removing an index.
     *
     * READ / PLAN only.
     */
    fun planDeleteInTransaction(
        transaction: Transaction,
        location: SummaryClientIndexLocation,
        ownerId: String,
        yearMonth: String,
        status: ClientType
    ): SummaryClientIndexDeletePlan {

        val clientRef =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status,
                bucketId = location.bucketId,
                clientId = location.index.clientId
            )

        val bucketRef =
            bucketReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status,
                bucketId = location.bucketId
            )

        return SummaryClientIndexDeletePlan(
            clientReference = clientRef,
            bucketReference = bucketRef,
            currentBucketSize = location.bucketSize
        )
    }

    /**
     * WRITE / APPLY for a planned deletion.
     *
     * Removes the client index and releases its bucket capacity.
     */
    fun applyDeletePlanInTransaction(
        transaction: Transaction,
        plan: SummaryClientIndexDeletePlan
    ) {

        transaction.delete(
            plan.clientReference
        )

        val newSize =
            (plan.currentBucketSize - 1L)
                .coerceAtLeast(0L)

        transaction.update(
            plan.bucketReference,
            "size",
            newSize
        )
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

        val pageCursor =
            decodeCursor(cursor)

        var currentBucketId =
            pageCursor?.bucketId

        var currentClientId =
            pageCursor?.clientId

        val result =
            mutableListOf<SummaryClientIndex>()

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

            val remaining =
                size - result.size

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
                    nextCursor =
                    encodeCursor(
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

                if (
                    nextBucketId != null &&
                    lastDocument != null
                ) {

                    return PageResult(
                        content = result,
                        hasNext = true,
                        nextCursor =
                        encodeCursor(
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

            currentBucketId =
                nextBucketId

            currentClientId =
                null
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

            val parts =
                decoded.split("|")

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
                ?.let {
                    ClientType.valueOf(it)
                }
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
            toFirestoreMap(index)
        )

        return index
    }
}
