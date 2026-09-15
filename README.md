override fun getAllSnapshotsForMonth(
    transaction: Transaction,
    yearMonth: YearMonth
): List<ClientBalanceSnapshot> {

    val query =
        db.collectionGroup("months")
            .whereEqualTo("yearMonth", yearMonth.toString())

    return transaction.get(query)
        .get()
        .documents
        .map { document ->

            ClientBalanceSnapshot(
                clientId =
                    document.getString("clientId")
                        ?: "",

                yearMonth =
                    YearMonth.parse(
                        document.getString("yearMonth")
                            ?: document.id
                    ),

                openingBalance =
                    document.getLong("openingBalance") ?: 0L,

                totalOrdersCount =
                    document.getLong("totalOrdersCount")
                        ?.toInt() ?: 0,

                totalInvoiceAmount =
                    document.getLong("totalInvoiceAmount") ?: 0L,

                totalInvoiceAmountWithGst =
                    document.getLong("totalInvoiceAmountWithGst")
                        ?: 0L,

                totalPayments =
                    document.getLong("totalPayments") ?: 0L,

                totalExpenses =
                    document.getLong("totalExpenses") ?: 0L,

                closingBalance =
                    document.getLong("closingBalance") ?: 0L,

                totalGstAmounts =
                    document.getLong("totalGstAmounts") ?: 0L,

                totalDiscount =
                    document.getLong("totalDiscount") ?: 0L,

                updatedAt =
                    document.getDate("updatedAt")
                        ?.toInstant()
                        ?: Instant.now()
            )
        }
        .sortedBy { it.yearMonth }
}
