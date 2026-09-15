fun rebuildGlobalSummaryForMonth(
    transaction: Transaction,
    yearMonth: YearMonth
): GlobalSummary {

    val snapshots =
        snapshotRepository.getAllSnapshotsForMonth(
            transaction = transaction,
            yearMonth = yearMonth
        )

    var totalOrdersCount = 0L
    var totalInvoiceAmount = 0L
    var totalGstAmount = 0L
    var totalInvoiceAmountWithGst = 0L
    var totalExpenses = 0L
    var totalPayments = 0L
    var totalReceivable = 0L
    var totalAdvance = 0L
    var totalDiscount = 0L

    snapshots.forEach { snapshot ->

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

    return GlobalSummary(
        yearMonth = yearMonth.toString(),
        totalOrdersCount = totalOrdersCount.toInt(),
        totalInvoiceAmount = totalInvoiceAmount,
        totalGstAmount = totalGstAmount,
        totalInvoiceAmountWithGst =
            totalInvoiceAmountWithGst,
        totalExpenses = totalExpenses,
        totalPayments = totalPayments,
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
