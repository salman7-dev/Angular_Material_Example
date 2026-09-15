package com.clientledger.core.service.summary

import com.clientledger.core.domain.snapshot.ClientBalanceSnapshot
import com.clientledger.core.domain.summary.*
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.order.FirestoreOrderRepository
import com.clientledger.core.repository.payment.FirestorePaymentRepository
import com.clientledger.core.repository.snapshot.SnapshotRepository
import com.google.cloud.firestore.Firestore
import org.springframework.stereotype.Service
import java.time.Instant
import java.time.YearMonth
import kotlin.math.abs

@Service
class GlobalSummaryService(
    private val clientRepository: ClientRepository,
    private val snapshotRepository: SnapshotRepository,
    private val db: Firestore,
    private val orderRepository: FirestoreOrderRepository,
    private val paymentRepository: FirestorePaymentRepository
) {

    /**
     * Recalculate and store the transaction-based global summary.
     *
     * Source of truth:
     * Client Balance Snapshots.
     *
     * IMPORTANT:
     * initialOpeningBalance is NOT added here.
     *
     * Snapshot closingBalance already contains the
     * effective transaction-based balance for the month.
     */
    fun recomputeGlobalSummaryForMonth(
        yearMonth: YearMonth
    ) {

        val clients =
            clientRepository.findAll()

        var totalOrdersCount = 0L
        var totalInvoiceAmount = 0L
        var totalGstAmount = 0L
        var totalInvoiceAmountWithGst = 0L
        var totalExpenses = 0L
        var totalPayments = 0L
        var totalReceivable = 0L
        var totalAdvance = 0L
        var totalDiscount = 0L

        clients.forEach { client ->

            val snapshot =
                getEffectiveSnapshotForMonth(
                    clientId = client.id,
                    yearMonth = yearMonth
                )

            totalOrdersCount +=
                snapshot.totalOrdersCount.toLong()

            totalInvoiceAmount +=
                snapshot.totalInvoiceAmount

            totalGstAmount +=
                snapshot.totalGstAmounts

            totalInvoiceAmountWithGst +=
                snapshot.totalInvoiceAmountWithGst

            totalExpenses +=
                snapshot.totalExpenses

            totalPayments +=
                snapshot.totalPayments

            totalDiscount +=
                snapshot.totalDiscount

            when {
                snapshot.closingBalance > 0L -> {
                    totalReceivable +=
                        snapshot.closingBalance
                }

                snapshot.closingBalance < 0L -> {
                    totalAdvance +=
                        abs(snapshot.closingBalance)
                }
            }
        }

        val netProfit =
            totalInvoiceAmount - totalExpenses

        val cashFlow =
            totalPayments - totalExpenses

        val globalSummary =
            GlobalSummary(
                yearMonth = yearMonth.toString(),

                totalOrdersCount =
                totalOrdersCount.toInt(),

                totalInvoiceAmount =
                totalInvoiceAmount,

                totalGstAmount =
                totalGstAmount,

                totalInvoiceAmountWithGst =
                totalInvoiceAmountWithGst,

                totalExpenses =
                totalExpenses,

                totalPayments =
                totalPayments,

                totalReceivableAmount =
                totalReceivable,

                totalAdvanceAmount =
                totalAdvance,

                netProfit =
                netProfit,

                cashFlow =
                cashFlow,

                totalDiscount =
                totalDiscount,

                updatedAt =
                Instant.now()
            )

        val docRef =
            db.collection("global_summaries")
                .document(yearMonth.toString())

        db.runTransaction { transaction ->
            transaction.set(
                docRef,
                globalSummary
            )
            null
        }.get()
    }

    /**
     * Returns the effective snapshot for a particular client/month.
     *
     * If an exact snapshot exists, it is returned.
     *
     * If the exact month does not exist, the latest previous
     * snapshot is used and its closing balance is carried forward.
     *
     * IMPORTANT:
     * initialOpeningBalance is NOT added here.
     */
    private fun getEffectiveSnapshotForMonth(
        clientId: String,
        yearMonth: YearMonth
    ): ClientBalanceSnapshot {

        /*
         * 1. Exact month snapshot.
         */
        val exactSnapshot =
            snapshotRepository.getSnapshot(
                clientId = clientId,
                yearMonth = yearMonth
            )

        if (exactSnapshot != null) {
            return exactSnapshot
        }

        /*
         * 2. Find latest snapshot before requested month.
         */
        val previousSnapshot =
            snapshotRepository.findLatestSnapshotBefore(
                clientId = clientId,
                yearMonth = yearMonth
            )

        val balance =
            previousSnapshot?.closingBalance ?: 0L

        /*
         * 3. Create a virtual snapshot.
         *
         * This is only used for calculation.
         * It is NOT written to Firestore.
         */
        return ClientBalanceSnapshot(
            clientId = clientId,
            yearMonth = yearMonth,
            openingBalance = balance,
            totalOrdersCount = 0,
            totalInvoiceAmount = 0L,
            totalInvoiceAmountWithGst = 0L,
            totalPayments = 0L,
            totalExpenses = 0L,
            closingBalance = balance,
            totalGstAmounts = 0L,
            totalDiscount = 0L,
            updatedAt = Instant.now()
        )
    }

    /**
     * Returns the stored global summary for a month.
     *
     * IMPORTANT:
     * This method reads ONLY from global_summaries.
     *
     * ClientMonthlyReporting is NOT used here.
     */
    fun getGlobalSummaryForMonth(
        yearMonth: YearMonth
    ): Map<String, Any?> {

        val docRef =
            db.collection("global_summaries")
                .document(yearMonth.toString())

        val snapshot =
            docRef.get().get()

        if (!snapshot.exists()) {
            return emptyMap()
        }

        return snapshot.data ?: emptyMap()
    }

    /**
     * Get complete yearly global summary.
     *
     * Monthly transaction metrics are summed.
     *
     * Receivable/Advance is taken only from the
     * final month of the requested year.
     */
    fun getGlobalSummaryForYear(
        year: Int
    ): GlobalSummary {

        val currentYear =
            YearMonth.now().year

        val currentMonthValue =
            YearMonth.now().monthValue

        val lastMonth =
            if (year == currentYear) {
                currentMonthValue
            } else {
                12
            }

        var totalOrdersCount = 0L
        var totalInvoiceAmount = 0L
        var totalGstAmount = 0L
        var totalInvoiceAmountWithGst = 0L
        var totalExpenses = 0L
        var totalPayments = 0L
        var totalDiscount = 0L
        var netProfit = 0L
        var cashFlow = 0L

        for (month in 1..lastMonth) {

            val yearMonth =
                YearMonth.of(year, month)

            val summary =
                getGlobalSummaryForMonth(yearMonth)

            totalOrdersCount +=
                (summary["totalOrdersCount"] as? Number)
                    ?.toLong() ?: 0L

            totalInvoiceAmount +=
                (summary["totalInvoiceAmount"] as? Number)
                    ?.toLong() ?: 0L

            totalGstAmount +=
                (summary["totalGstAmount"] as? Number)
                    ?.toLong() ?: 0L

            totalInvoiceAmountWithGst +=
                (summary["totalInvoiceAmountWithGst"] as? Number)
                    ?.toLong() ?: 0L

            totalExpenses +=
                (summary["totalExpenses"] as? Number)
                    ?.toLong() ?: 0L

            totalPayments +=
                (summary["totalPayments"] as? Number)
                    ?.toLong() ?: 0L

            totalDiscount +=
                (summary["totalDiscount"] as? Number)
                    ?.toLong() ?: 0L

            netProfit +=
                (summary["netProfit"] as? Number)
                    ?.toLong() ?: 0L

            cashFlow +=
                (summary["cashFlow"] as? Number)
                    ?.toLong() ?: 0L
        }

        /*
         * Receivable/Advance is a point-in-time balance.
         *
         * Therefore do NOT sum it across months.
         */
        val finalMonth =
            YearMonth.of(year, lastMonth)

        val finalSummary =
            getGlobalSummaryForMonth(finalMonth)

        val totalReceivableAmount =
            (finalSummary["totalReceivableAmount"] as? Number)
                ?.toLong() ?: 0L

        val totalAdvanceAmount =
            (finalSummary["totalAdvanceAmount"] as? Number)
                ?.toLong() ?: 0L

        return GlobalSummary(
            yearMonth = year.toString(),

            totalOrdersCount =
            totalOrdersCount.toInt(),

            totalInvoiceAmount =
            totalInvoiceAmount,

            totalGstAmount =
            totalGstAmount,

            totalInvoiceAmountWithGst =
            totalInvoiceAmountWithGst,

            totalExpenses =
            totalExpenses,

            totalPayments =
            totalPayments,

            totalReceivableAmount =
            totalReceivableAmount,

            totalAdvanceAmount =
            totalAdvanceAmount,

            totalDiscount =
            totalDiscount,

            netProfit =
            netProfit,

            cashFlow =
            cashFlow
        )
    }

    /**
     * Invoice drilldown.
     */
    fun getInvoiceDrillDown(
        year: Int,
        month: Int?
    ): InvoiceDrillDownResponse {

        val allClients =
            clientRepository
                .findAll()
                .associateBy { it.id }

        val orders =
            if (month != null) {
                orderRepository.findAllForMonth(year, month)
            } else {
                orderRepository.findAllForYear(year)
            }

        val items =
            orders.map { order ->

                val client =
                    allClients[order.clientId]

                val totalWithGst =
                    order.totalInvoiceAmount

                val gst =
                    order.totalGstAmount

                val withoutGst =
                    totalWithGst - gst

                InvoiceDrillDownDto(
                    orderId = order.id,
                    clientId = order.clientId,
                    clientName = client?.name ?: "Unknown Client",
                    orderDate = order.orderDate,
                    invoiceAmountWithoutGst = withoutGst,
                    gstAmount = gst,
                    totalAmountWithGst = totalWithGst
                )
            }

        return InvoiceDrillDownResponse(
            totalCount = items.size,
            totalInvoiceAmountWithOutGst =
            items.sumOf { it.invoiceAmountWithoutGst },
            totalInvoiceAmountWithGst =
            items.sumOf { it.totalAmountWithGst },
            totalGstAmount =
            items.sumOf { it.gstAmount },
            items = items
        )
    }

    /**
     * Expense drilldown.
     */
    fun getExpenseDrillDown(
        year: Int,
        month: Int?
    ): BalanceDrillDownResponse<ExpenseDrillDownDto> {

        val allClients =
            clientRepository
                .findAll()
                .associateBy { it.id }

        val orders =
            if (month != null) {
                orderRepository.findAllForMonth(year, month)
            } else {
                orderRepository.findAllForYear(year)
            }

        val expenseOrders =
            orders.filter { order ->
                order.totalExpense > 0
            }

        val items =
            expenseOrders.map { order ->

                val client =
                    allClients[order.clientId]

                ExpenseDrillDownDto(
                    orderId = order.id,
                    clientId = order.clientId,
                    clientName = client?.name ?: "Unknown Client",
                    orderDate = order.orderDate,
                    totalExpense = order.totalExpense
                )
            }

        return BalanceDrillDownResponse(
            totalCount = items.size,
            totalAmount =
            items.sumOf { it.totalExpense },
            items = items
        )
    }

    /**
     * Payment drilldown.
     */
    fun getPaymentDrillDown(
        year: Int,
        month: Int?
    ): BalanceDrillDownResponse<PaymentDrillDownDto> {

        val allClients =
            clientRepository
                .findAll()
                .associateBy { it.id }

        val payments =
            if (month != null) {
                paymentRepository.findAllForMonth(year, month)
            } else {
                paymentRepository.findAllForYear(year)
            }

        val items =
            payments.map { payment ->

                val client =
                    allClients[payment.clientId]

                PaymentDrillDownDto(
                    paymentId = payment.id,
                    clientId = payment.clientId,
                    clientName = client?.name ?: "Unknown Client",
                    paymentDate = payment.paymentDate,
                    amount = payment.amount,
                    paymentType = payment.paymentMode
                )
            }

        return BalanceDrillDownResponse(
            totalCount = items.size,
            totalAmount =
            items.sumOf { it.amount },
            items = items
        )
    }

    /**
     * Receivable drilldown.
     *
     * Uses Client Balance Snapshots only.
     *
     * month != null:
     * requested month
     *
     * month == null:
     * December
     */
    fun getReceivableDrillDown(
        year: Int,
        month: Int?
    ): BalanceDrillDownResponse<BalanceDrillDownDto> {

        val targetYearMonth =
            if (month != null) {
                YearMonth.of(year, month)
            } else {
                YearMonth.of(year, 12)
            }

        val clients =
            clientRepository.findAll()

        val items =
            clients.mapNotNull { client ->

                val snapshot =
                    getEffectiveSnapshotForMonth(
                        clientId = client.id,
                        yearMonth = targetYearMonth
                    )

                if (snapshot.closingBalance > 0L) {
                    BalanceDrillDownDto(
                        clientId = client.id,
                        clientName = client.name,
                        balanceAmount = snapshot.closingBalance
                    )
                } else {
                    null
                }
            }

        return BalanceDrillDownResponse(
            totalCount = items.size,
            totalAmount =
            items.sumOf { it.balanceAmount },
            items = items
        )
    }

    /**
     * Advance drilldown.
     *
     * Uses Client Balance Snapshots only.
     */
    fun getAdvanceDrillDown(
        year: Int,
        month: Int?
    ): BalanceDrillDownResponse<BalanceDrillDownDto> {

        val targetYearMonth =
            if (month != null) {
                YearMonth.of(year, month)
            } else {
                YearMonth.of(year, 12)
            }

        val clients =
            clientRepository.findAll()

        val items =
            clients.mapNotNull { client ->

                val snapshot =
                    getEffectiveSnapshotForMonth(
                        clientId = client.id,
                        yearMonth = targetYearMonth
                    )

                if (snapshot.closingBalance < 0L) {
                    BalanceDrillDownDto(
                        clientId = client.id,
                        clientName = client.name,
                        balanceAmount =
                        abs(snapshot.closingBalance)
                    )
                } else {
                    null
                }
            }

        return BalanceDrillDownResponse(
            totalCount = items.size,
            totalAmount =
            items.sumOf { it.balanceAmount },
            items = items
        )
    }

    /**
     * Discount drilldown.
     *
     * Uses Client Balance Snapshots only.
     */
    fun getDiscountDrillDown(
        year: Int,
        month: Int?
    ): List<DiscountDrillDownDto> {

        val targetYearMonth =
            if (month != null) {
                YearMonth.of(year, month)
            } else {
                YearMonth.of(year, 12)
            }

        val clients =
            clientRepository.findAll()

        return clients.mapNotNull { client ->

            val snapshot =
                getEffectiveSnapshotForMonth(
                    clientId = client.id,
                    yearMonth = targetYearMonth
                )

            if (snapshot.totalDiscount != 0L) {
                DiscountDrillDownDto(
                    clientId = client.id,
                    clientName = client.name,
                    totalDiscount = snapshot.totalDiscount
                )
            } else {
                null
            }
        }
    }

    /**
     * Calculate Global Summary using old/new snapshot delta.
     *
     * This method is used by the transactional ledger update flow.
     *
     * ClientMonthlyReporting is NOT involved.
     */
    fun calculateGlobalSummaryDelta(
        yearMonth: YearMonth,
        existingData: Map<String, Any?>,
        oldSnapshot: ClientBalanceSnapshot?,
        newSnapshot: ClientBalanceSnapshot
    ): GlobalSummary {

        val oldOrdersCount =
            oldSnapshot?.totalOrdersCount ?: 0

        val oldInvoiceAmount =
            oldSnapshot?.totalInvoiceAmount ?: 0L

        val oldGstAmount =
            oldSnapshot?.totalGstAmounts ?: 0L

        val oldInvoiceAmountWithGst =
            oldSnapshot?.totalInvoiceAmountWithGst ?: 0L

        val oldExpenses =
            oldSnapshot?.totalExpenses ?: 0L

        val oldPayments =
            oldSnapshot?.totalPayments ?: 0L

        val oldDiscount =
            oldSnapshot?.totalDiscount ?: 0L

        val oldClosingBalance =
            oldSnapshot?.closingBalance ?: 0L

        val newOrdersCount =
            newSnapshot.totalOrdersCount

        val newInvoiceAmount =
            newSnapshot.totalInvoiceAmount

        val newGstAmount =
            newSnapshot.totalGstAmounts

        val newInvoiceAmountWithGst =
            newSnapshot.totalInvoiceAmountWithGst

        val newExpenses =
            newSnapshot.totalExpenses

        val newPayments =
            newSnapshot.totalPayments

        val newDiscount =
            newSnapshot.totalDiscount

        val newClosingBalance =
            newSnapshot.closingBalance

        val totalOrdersCount =
            ((existingData["totalOrdersCount"] as? Number)?.toLong() ?: 0L) -
                    oldOrdersCount +
                    newOrdersCount

        val totalInvoiceAmount =
            ((existingData["totalInvoiceAmount"] as? Number)?.toLong() ?: 0L) -
                    oldInvoiceAmount +
                    newInvoiceAmount

        val totalGstAmount =
            ((existingData["totalGstAmount"] as? Number)?.toLong() ?: 0L) -
                    oldGstAmount +
                    newGstAmount

        val totalInvoiceAmountWithGst =
            ((existingData["totalInvoiceAmountWithGst"] as? Number)?.toLong() ?: 0L) -
                    oldInvoiceAmountWithGst +
                    newInvoiceAmountWithGst

        val totalExpenses =
            ((existingData["totalExpenses"] as? Number)?.toLong() ?: 0L) -
                    oldExpenses +
                    newExpenses

        val totalPayments =
            ((existingData["totalPayments"] as? Number)?.toLong() ?: 0L) -
                    oldPayments +
                    newPayments

        val totalDiscount =
            ((existingData["totalDiscount"] as? Number)?.toLong() ?: 0L) -
                    oldDiscount +
                    newDiscount

        var totalReceivable =
            (existingData["totalReceivableAmount"] as? Number)?.toLong()
                ?: 0L

        var totalAdvance =
            (existingData["totalAdvanceAmount"] as? Number)?.toLong()
                ?: 0L

        when {
            oldClosingBalance > 0L ->
                totalReceivable -= oldClosingBalance

            oldClosingBalance < 0L ->
                totalAdvance -= abs(oldClosingBalance)
        }

        when {
            newClosingBalance > 0L ->
                totalReceivable += newClosingBalance

            newClosingBalance < 0L ->
                totalAdvance += abs(newClosingBalance)
        }

        return GlobalSummary(
            yearMonth = yearMonth.toString(),

            totalOrdersCount =
            totalOrdersCount.toInt(),

            totalInvoiceAmount =
            totalInvoiceAmount,

            totalGstAmount =
            totalGstAmount,

            totalInvoiceAmountWithGst =
            totalInvoiceAmountWithGst,

            totalExpenses =
            totalExpenses,

            totalPayments =
            totalPayments,

            totalReceivableAmount =
            totalReceivable,

            totalAdvanceAmount =
            totalAdvance,

            netProfit =
            totalInvoiceAmount - totalExpenses,

            cashFlow =
            totalPayments - totalExpenses,

            totalDiscount =
            totalDiscount,

            updatedAt =
            Instant.now()
        )
    }
}

