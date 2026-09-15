val existingGlobalSummaries =
    calculatedSnapshots.associateWith { snapshot ->
        getGlobalSummaryData(
            transaction = transaction,
            yearMonth = snapshot.yearMonth
        )
    }

val updatedGlobalSummaries =
    calculatedSnapshots.associate { newSnapshot ->
        val yearMonth =
            newSnapshot.yearMonth

        val existingData =
            existingGlobalSummaries[
                newSnapshot
            ]

        val updatedSummary =
            if (existingData == null) {
                globalSummaryService.rebuildGlobalSummaryForMonth(
                    transaction = transaction,
                    yearMonth = yearMonth,
                    replacementSnapshots = calculatedSnapshots
                )
            } else {
                val oldSnapshot =
                    oldSnapshotForGlobalByMonth[
                        yearMonth
                    ] ?: getEffectiveSnapshot(
                        clientId = clientId,
                        yearMonth = yearMonth,
                        snapshots = snapshots
                    )

                globalSummaryService.calculateGlobalSummaryDelta(
                    yearMonth = yearMonth,
                    existingData = existingData,
                    oldSnapshot = oldSnapshot,
                    newSnapshot = newSnapshot
                )
            }

        yearMonth to updatedSummary
    }
