package com.clientledger.core.service.client

import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.GlobalSummary
import com.clientledger.core.repository.client.ClientDeletionJobRepository
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.lock.OwnerOperationLockRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.service.lock.OwnerOperationLockService
import com.clientledger.core.transaction.FirestoreTransactionExecutor
import com.google.cloud.firestore.Firestore
import org.springframework.stereotype.Service
import java.time.Clock
import java.time.YearMonth

@Service
class ClientDeletionPhaseOneService(
    private val firestore: Firestore,
    private val transactionExecutor: FirestoreTransactionExecutor,
    private val clientRepository: ClientRepository,
    private val historyRepository: ClientHistoryRepository,
    private val summaryIndexRepository: SummaryClientIndexRepository,
    private val globalSummaryRepository: GlobalSummaryRepository,
    private val deletionJobRepository: ClientDeletionJobRepository,
    private val ownerOperationLockRepository: OwnerOperationLockRepository,
    private val ownerOperationLockService: OwnerOperationLockService,
    private val clock: Clock
) {

    /**
     * Phase 1:
     *
     * 1. Read the client master and monthly client histories.
     * 2. Reverse the client's position separately for each month.
     * 3. Reverse monthly financial activity using ClientHistory.
     * 4. Remove client history and summary-index entries.
     * 5. Decrement the history bucket sizes.
     * 6. Create the deletion job.
     * 7. Delete the client master.
     *
     * Order, Payment and Invoice documents are NOT read for financial
     * calculations and are NOT deleted here.
     *
     * Phase 2 is responsible for physically deleting child documents.
     */
    fun execute(
        ownerId: String,
        clientId: String
    ): com.clientledger.core.domain.ClientDeletionJob {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        return ownerOperationLockService.executeBusiness(ownerId) { context, fence ->

            transactionExecutor.execute(context) { transaction, _ ->

                check(
                    ownerOperationLockRepository.isFenceValidInTransaction(
                        transaction,
                        ownerId,
                        fence
                    )
                ) {
                    "Owner operation lock is no longer valid"
                }

                /*
                 * Step 1: Check whether Phase 1 has already completed.
                 *
                 * This makes repeated requests idempotent.
                 */
                val existingJob = deletionJobRepository.findInTransaction(
                    transaction,
                    ownerId,
                    clientId
                )

                if (existingJob != null) {
                    check(existingJob.phase1Completed) {
                        "A deletion job exists but Phase 1 is incomplete"
                    }

                    return@execute existingJob
                }

                /*
                 * Step 2: Read the client master.
                 */
                val client = clientRepository.findByIdInTransaction(
                    transaction,
                    ownerId,
                    clientId
                ) ?: throw IllegalArgumentException(
                    "Client not found: $clientId"
                )

                check(client.ownerId == ownerId) {
                    "Client does not belong to the supplied owner"
                }

                check(client.bucketId.isNotBlank()) {
                    "Client bucketId is missing"
                }

                /*
                 * Step 3: Determine the client's history window.
                 *
                 * Each month is handled independently. A later month's
                 * position must never be reversed using an earlier month's
                 * receivable or advance balance.
                 */
                val months = ClientDeletionWindow.months(
                    client.createdAt,
                    clock
                )

                /*
                 * Step 4: Read the monthly client histories.
                 *
                 * These records are the source for both:
                 * - client position reversal
                 * - monthly activity reversal
                 *
                 * We deliberately do not query Order or Payment documents.
                 */
                val histories = historyRepository.findAllByMonthsInTransaction(
                    transaction,
                    ownerId,
                    months,
                    client.bucketId,
                    clientId
                )

                val existingHistories = histories
                    .mapNotNull { (month, history) ->
                        history?.let { month to it }
                    }
                    .toMap()

                /*
                 * Step 5: Read history bucket sizes before performing writes.
                 */
                val historyBucketSizes = existingHistories.keys.associateWith { month ->

                    val bucketReference = firestore
                        .collection("owners")
                        .document(ownerId)
                        .collection("master_history_client_data")
                        .document(month.toString())
                        .collection("buckets")
                        .document(client.bucketId)

                    val bucketSnapshot = transaction
                        .get(bucketReference)
                        .get()

                    check(bucketSnapshot.exists()) {
                        "History bucket is missing for $month"
                    }

                    val size = requireNotNull(
                        bucketSnapshot.getLong("size")
                    ) {
                        "History bucket size is missing for $month"
                    }

                    check(size > 0) {
                        "History bucket size is invalid for $month"
                    }

                    bucketReference to size
                }

                /*
                 * Step 6: Plan summary-index deletions.
                 *
                 * The index repository is responsible for deleting the
                 * client's index entry and decrementing its bucket size.
                 */
                val indexDeletePlans = months.mapNotNull { month ->

                    val location = summaryIndexRepository.findInTransaction(
                        transaction,
                        ownerId,
                        month.toString(),
                        client.bucketId,
                        clientId
                    ) ?: return@mapNotNull null

                    summaryIndexRepository.planDeleteInTransaction(
                        transaction,
                        location,
                        ownerId,
                        month.toString()
                    )
                }

                /*
                 * Step 7: Calculate GlobalSummary reversals.
                 *
                 * There is one accumulated delta per month.
                 *
                 * Position fields:
                 *   - receivableClientCount
                 *   - advanceClientCount
                 *   - settledClientCount
                 *   - totalReceivableAmount
                 *   - totalAdvanceAmount
                 *
                 * Activity fields:
                 *   - invoice totals and count
                 *   - GST
                 *   - payment totals and count
                 *   - expenses
                 *   - cash flow
                 *   - net profit
                 */
                val deltasByMonth = mutableMapOf<YearMonth, GlobalSummary>()

                existingHistories.forEach { (month, history) ->

                    /*
                     * Reverse this month's client position.
                     *
                     * Example:
                     * July:      receivable = 2,240
                     * September: advance = 7,760
                     *
                     * These must be reversed independently.
                     */
                    addDelta(
                        deltas = deltasByMonth,
                        month = month,
                        addition = reverseClientPosition(
                            ownerId = ownerId,
                            month = month.toString(),
                            status = history.status,
                            receivable = history.receivable,
                            advance = history.advance
                        )
                    )

                    /*
                     * Reverse only this month's activity.
                     *
                     * Original invoice values are not used here.
                     * The calculated monthly values are used instead.
                     */
                    addDelta(
                        deltas = deltasByMonth,
                        month = month,
                        addition = reverseMonthlyActivity(
                            ownerId = ownerId,
                            month = month.toString(),
                            totalInvoiceAmount = history.totalInvoiceAmount,
                            totalInvoiceAmountWithGst =
                                history.totalInvoiceAmountWithGst,
                            invoiceCount = history.invoiceCount,
                            totalGstAmount = history.totalGstAmount,
                            totalPayments = history.totalPayments,
                            paymentCount = history.paymentCount,
                            totalExpenses = history.totalExpenses
                        )
                    )
                }

                /*
                 * Step 8: Read and plan all GlobalSummary updates.
                 *
                 * Do this before any writes so all transaction reads
                 * complete before the transaction begins writing.
                 */
                val summaryPlans = deltasByMonth.map { (month, delta) ->

                    val plan = globalSummaryRepository.planDeltaInTransaction(
                        transaction = transaction,
                        ownerId = ownerId,
                        yearMonth = month.toString(),
                        delta = delta
                    )

                    check(plan.currentSummary != null) {
                        "GlobalSummary is missing for $month; " +
                            "client deletion cannot safely reverse its totals"
                    }

                    plan
                }

                /*
                 * Step 9: All required reads are complete.
                 * Apply the writes.
                 */

                /*
                 * 9a. Delete monthly client history and decrement its bucket.
                 */
                existingHistories.forEach { (month, _) ->

                    val historyReference = firestore
                        .collection("owners")
                        .document(ownerId)
                        .collection("master_history_client_data")
                        .document(month.toString())
                        .collection("buckets")
                        .document(client.bucketId)
                        .collection("client_histories")
                        .document(clientId)

                    transaction.delete(historyReference)

                    val (bucketReference, currentSize) =
                        historyBucketSizes.getValue(month)

                    transaction.update(
                        bucketReference,
                        "size",
                        currentSize - 1
                    )
                }

                /*
                 * 9b. Delete summary-index entries and decrement their
                 *     corresponding bucket sizes.
                 */
                indexDeletePlans.forEach { plan ->

                    summaryIndexRepository.applyDeletePlanInTransaction(
                        transaction,
                        plan
                    )
                }

                /*
                 * 9c. Apply the negative GlobalSummary deltas.
                 */
                summaryPlans.forEach { plan ->

                    globalSummaryRepository.applyDeltaPlanInTransaction(
                        transaction,
                        plan
                    )
                }

                /*
                 * 9d. Save the Phase 2 job.
                 *
                 * Child deletion counters start at zero because this
                 * phase does not delete child documents.
                 */
                val now = clock.instant()

                val job = com.clientledger.core.domain.ClientDeletionJob(
                    ownerId = ownerId,
                    clientId = clientId,
                    clientName = client.clientName,
                    clientCreatedAt = client.createdAt,
                    clientBucketId = client.bucketId,
                    historyMonths = months.map(YearMonth::toString),
                    phase1Completed = true,
                    phase1CompletedAt = now,
                    phase2Completed = false,
                    ordersDeleted = 0,
                    paymentsDeleted = 0,
                    invoicesDeleted = 0,
                    status = "PHASE_1_COMPLETED",
                    createdAt = now,
                    updatedAt = now,
                    lastError = null
                )

                deletionJobRepository.saveInTransaction(
                    transaction,
                    job
                )

                /*
                 * 9e. Delete the client master.
                 *
                 * The existing repository implementation must continue
                 * to maintain the client master bucket count correctly.
                 */
                clientRepository.deleteInTransaction(
                    transaction,
                    ownerId,
                    clientId
                )

                job
            }
        }
    }

    /**
     * Removes the client's contribution to its position in one month.
     *
     * This method does not reverse invoice/payment activity.
     */
    private fun reverseClientPosition(
        ownerId: String,
        month: String,
        status: ClientType,
        receivable: java.math.BigDecimal,
        advance: java.math.BigDecimal
    ): GlobalSummary {

        return GlobalSummary(
            ownerId = ownerId,
            yearMonth = month,

            receivableClientCount =
                if (status == ClientType.RECEIVABLE) -1 else 0,

            advanceClientCount =
                if (status == ClientType.ADVANCE) -1 else 0,

            settledClientCount =
                if (status == ClientType.SETTLED) -1 else 0,

            totalReceivableAmount = receivable.negate(),
            totalAdvanceAmount = advance.negate()
        )
    }

    /**
     * Reverses monthly activity from the saved ClientHistory.
     *
     * This uses calculated history values rather than reading Order or
     * Payment documents and independently aggregating their values.
     *
     * Cash-flow convention from the supplied examples:
     *   cashFlow = payments - expenses
     *
     * Profit convention from the supplied examples:
     *   netProfit = invoice amount - expenses
     *
     * The returned values are the negatives of those contributions.
     */
    private fun reverseMonthlyActivity(
        ownerId: String,
        month: String,
        totalInvoiceAmount: java.math.BigDecimal,
        totalInvoiceAmountWithGst: java.math.BigDecimal,
        invoiceCount: Int,
        totalGstAmount: java.math.BigDecimal,
        totalPayments: java.math.BigDecimal,
        paymentCount: Int,
        totalExpenses: java.math.BigDecimal
    ): GlobalSummary {

        return GlobalSummary(
            ownerId = ownerId,
            yearMonth = month,

            totalInvoiceAmount =
                totalInvoiceAmount.negate(),

            totalInvoiceAmountWithGst =
                totalInvoiceAmountWithGst.negate(),

            totalInvoiceCount = -invoiceCount,

            totalGstAmount =
                totalGstAmount.negate(),

            totalPaymentAmount =
                totalPayments.negate(),

            totalPaymentCount = -paymentCount,

            totalExpenseAmount =
                totalExpenses.negate(),

            cashFlow =
                totalExpenses.subtract(totalPayments),

            netProfit =
                totalExpenses.subtract(totalInvoiceAmount)
        )
    }

    /**
     * Adds a delta to the existing monthly delta.
     *
     * Only one GlobalSummary update plan is created per month.
     */
    private fun addDelta(
        deltas: MutableMap<YearMonth, GlobalSummary>,
        month: YearMonth,
        addition: GlobalSummary
    ) {

        val existing = deltas[month]

        deltas[month] = if (existing == null) {
            addition
        } else {
            existing.copy(
                receivableClientCount =
                    existing.receivableClientCount +
                        addition.receivableClientCount,

                advanceClientCount =
                    existing.advanceClientCount +
                        addition.advanceClientCount,

                settledClientCount =
                    existing.settledClientCount +
                        addition.settledClientCount,

                totalReceivableAmount =
                    existing.totalReceivableAmount +
                        addition.totalReceivableAmount,

                totalAdvanceAmount =
                    existing.totalAdvanceAmount +
                        addition.totalAdvanceAmount,

                cashFlow =
                    existing.cashFlow + addition.cashFlow,

                netProfit =
                    existing.netProfit + addition.netProfit,

                totalInvoiceAmount =
                    existing.totalInvoiceAmount +
                        addition.totalInvoiceAmount,

                totalInvoiceAmountWithGst =
                    existing.totalInvoiceAmountWithGst +
                        addition.totalInvoiceAmountWithGst,

                totalInvoiceCount =
                    existing.totalInvoiceCount +
                        addition.totalInvoiceCount,

                totalGstAmount =
                    existing.totalGstAmount +
                        addition.totalGstAmount,

                totalExpenseAmount =
                    existing.totalExpenseAmount +
                        addition.totalExpenseAmount,

                totalPaymentAmount =
                    existing.totalPaymentAmount +
                        addition.totalPaymentAmount,

                totalPaymentCount =
                    existing.totalPaymentCount +
                        addition.totalPaymentCount
            )
        }
    }
}
