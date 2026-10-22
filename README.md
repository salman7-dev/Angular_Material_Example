
package com.clientledger.core.service.client

import java.time.Clock
import java.time.Instant
import java.time.YearMonth

internal object ClientDeletionWindow {

    private const val MAX_PREVIOUS_YEARS = 3

    /**
     * Includes the client's creation month through the current month,
     * limited to the current calendar year and previous three years.
     *
     * Examples when the current month is October 2026:
     * - Created October 2026 -> October 2026
     * - Created October 2025 -> October 2025 through October 2026
     * - Created September 2022 -> January 2023 through October 2026
     */
    fun months(
        clientCreatedAt: Instant,
        clock: Clock
    ): List<YearMonth> {
        val today = java.time.LocalDate.now(clock)
        val currentMonth = YearMonth.from(today)

        val creationDate = clientCreatedAt
            .atZone(clock.zone)
            .toLocalDate()

        require(!creationDate.isAfter(today)) {
            "Client creation date cannot be in the future."
        }

        val creationMonth = YearMonth.from(creationDate)

        // Four calendar years total: current year and previous three years.
        val earliestAllowedMonth = YearMonth.of(
            currentMonth.year - MAX_PREVIOUS_YEARS,
            1
        )

        val firstMonth = maxOf(
            creationMonth,
            earliestAllowedMonth
        )

        val months = mutableListOf<YearMonth>()
        var month = firstMonth

        while (!month.isAfter(currentMonth)) {
            months.add(month)
            month = month.plusMonths(1)
        }

        return months
    }
}
