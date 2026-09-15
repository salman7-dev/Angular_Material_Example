override fun getOpeningBalanceSeed(
    transaction: Transaction,
    clientId: String,
    startMonth: YearMonth,
    fallbackOpeningBalance: Long
): Long {

    // Exact month snapshot
    getSnapshot(transaction, clientId, startMonth)
        ?.let { return it.openingBalance }

    val currentMonth = YearMonth.now()
    val futureMonths = ChronoUnit.MONTHS.between(startMonth, currentMonth)
    val pastMonths = 12L - futureMonths

    val windowStart = startMonth.minusMonths(pastMonths)
    val windowEnd = startMonth.plusMonths(futureMonths)

    val snapshots = transaction.get(
        db.collectionGroup("months")
            .whereEqualTo("clientId", clientId)
            .whereGreaterThanOrEqualTo("yearMonth", windowStart.toString())
            .whereLessThanOrEqualTo("yearMonth", windowEnd.toString())
    ).get().documents

    if (snapshots.isEmpty()) {
        return fallbackOpeningBalance
    }

    val nearest = snapshots
        .mapNotNull { doc ->
            doc.getString("yearMonth")?.let { ym ->
                Triple(
                    doc,
                    YearMonth.parse(ym),
                    kotlin.math.abs(
                        ChronoUnit.MONTHS.between(startMonth, YearMonth.parse(ym))
                    )
                )
            }
        }
        .minByOrNull { it.third }
        ?: return fallbackOpeningBalance

    val (document, snapshotMonth) = nearest

    return if (snapshotMonth.isBefore(startMonth)) {
        document.getLong("closingBalance")
    } else {
        document.getLong("openingBalance")
    } ?: fallbackOpeningBalance
}




 private fun getGlobalSummaryData(
        transaction: Transaction,
        yearMonth: YearMonth
    ): Map<String, Any?> {

        val docRef =
            db.collection("global_summaries")
                .document(yearMonth.toString())

        val document = transaction.get(docRef).get()

        if (!document.exists()) {
            return emptyMap()
        }
        return document.data ?: emptyMap()
    }


private fun getOrRecalculateMonthSnapshot(
        transaction: Transaction,
        clientId: String,
        yearMonth: YearMonth,
        openingBalance: Long,
        client: Client?
    ): ClientBalanceSnapshot {
        println("DEBUG: getOrRecalculateMonthSnapshot - Checking for existing snapshot for $clientId in $yearMonth")
        val existingSnapshot = getMonthSnapshot(
            transaction = transaction,
            clientId = clientId,
            yearMonth = yearMonth
        )

        // Snapshot exists → cheap propagation.
        if (existingSnapshot != null) {
            return ledgerCalculationEngine.propagateBalance(
                existingSnapshot = existingSnapshot,
                openingBalance = openingBalance
            )
        }

        // Snapshot is missing → recovery case.
        // Read only this month's actual transactions.
        val orders = orderRepository.findAllForClientAndMonth(
            transaction,
            clientId,
            yearMonth
        )
        val payments = paymentRepository.findAllForClientAndMonth(
            transaction,
            clientId,
            yearMonth
        )
        val orderIds = orders.map { it.id }.toSet()
        val invoices = invoiceRepository.findAllForClientAndOrderIds(
            transaction,
            clientId,
            orderIds
        )
        val discountByMonth = invoices.sumOf { invoice -> invoice.discount }
        val totalOrdersCount = orders.size
        val totalInvoiceAmountWithGst = orders.sumOf { it.totalInvoiceAmount }
        val totalExpenses = orders.sumOf { it.totalExpense }
        val totalGstAmounts = orders.sumOf { it.totalGstAmount }
        val totalPayments = payments.sumOf { it.amount }
        val totalInvoiceAmount = totalInvoiceAmountWithGst - totalGstAmounts
        val closingBalance = openingBalance +
                    totalInvoiceAmountWithGst -
                    totalPayments -
                    discountByMonth

        println("DEBUG: getOrRecalculateMonthSnapshot - Recalculated snapshot for $clientId in $yearMonth: openingBalance=$openingBalance, totalOrdersCount=$totalOrdersCount, totalInvoiceAmount=$totalInvoiceAmount, totalInvoiceAmountWithGst=$totalInvoiceAmountWithGst, totalPayments=$totalPayments, totalExpenses=$totalExpenses, closingBalance=$closingBalance, totalGstAmounts=$totalGstAmounts, totalDiscount=$discountByMonth")
        return ClientBalanceSnapshot(
            clientId = clientId,
            yearMonth = yearMonth,
            openingBalance = openingBalance,
            totalOrdersCount = totalOrdersCount,
            totalInvoiceAmount = totalInvoiceAmount,
            totalInvoiceAmountWithGst = totalInvoiceAmountWithGst,
            totalPayments = totalPayments,
            totalExpenses = totalExpenses,
            closingBalance = closingBalance,
            totalGstAmounts = totalGstAmounts,
            totalDiscount = discountByMonth,
            updatedAt = Instant.now()
        )
    }
