val uniqueGlobalMonths =
    allCalculatedSnapshots
        .map { it.yearMonth }
        .toSet()

val existingGlobalSummaries =
    uniqueGlobalMonths.associateWith { yearMonth ->
        getGlobalSummaryData(
            transaction = transaction,
            yearMonth = yearMonth
        )
    }

val updatedGlobalSummaries =
    mutableMapOf<YearMonth, GlobalSummary>()

affectedClientIds.forEach { clientId ->
    val clientSnapshots =
        allCalculatedSnapshots.filter {
            it.clientId == clientId
        }

    clientSnapshots.forEach { newSnapshot ->
        val yearMonth =
            newSnapshot.yearMonth

        val existingGlobalData =
            updatedGlobalSummaries[
                yearMonth
            ]?.let { summary ->
                globalSummaryToMap(summary)
            } ?: existingGlobalSummaries[
                yearMonth
            ]

        val updatedSummary =
            if (existingGlobalData == null) {
                globalSummaryService.rebuildGlobalSummaryForMonth(
                    transaction = transaction,
                    yearMonth = yearMonth,
                    replacementSnapshots = allCalculatedSnapshots
                )
            } else {
                val oldSnapshot =
                    oldSnapshotForGlobalByClientMonth[
                        clientId to yearMonth
                    ] ?: getEffectiveSnapshot(
                        clientId = clientId,
                        yearMonth = yearMonth,
                        snapshots =
                        snapshotsByClient[
                            clientId
                        ].orEmpty()
                    )

                globalSummaryService.calculateGlobalSummaryDelta(
                    yearMonth = yearMonth,
                    existingData = existingGlobalData,
                    oldSnapshot = oldSnapshot,
                    newSnapshot = newSnapshot
                )
            }

        updatedGlobalSummaries[
            yearMonth
        ] = updatedSummary
    }
}
