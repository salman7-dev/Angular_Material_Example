private fun expectedMonths(clientCreatedAt: Instant): List<String> {
    val createdMonth = YearMonth.from(
        clientCreatedAt.atZone(ZoneId.of("UTC"))
    )

    val currentMonth = YearMonth.from(
        testClock.instant().atZone(ZoneId.of("UTC"))
    )

    require(!createdMonth.isAfter(currentMonth)) {
        "Client creation month cannot be after the current month"
    }

    return (createdMonth.monthValue..currentMonth.monthValue)
        .takeIf { createdMonth.year == currentMonth.year }
        ?.map { month ->
            YearMonth.of(createdMonth.year, month).toString()
        }
        ?: buildList {
            var month = createdMonth
            while (!month.isAfter(currentMonth)) {
                add(month.toString())
                month = month.plusMonths(1)
            }
        }
}
