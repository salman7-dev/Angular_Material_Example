package com.clientledger.core.service.monthlyrollover

import com.clientledger.core.repository.monthlyrollover.GlobalMonthlyRolloverJobRepository
import com.clientledger.core.repository.owner.OwnerRepository
import org.slf4j.LoggerFactory
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Service
import java.time.Clock
import java.time.YearMonth
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors
import java.util.concurrent.Future

@Service
class MonthlyRolloverOrchestrator(
    private val ownerRepository: OwnerRepository,
    private val globalMonthlyRolloverJobRepository:
    GlobalMonthlyRolloverJobRepository,
    private val perOwnerMonthlyRolloverService:
    PerOwnerMonthlyRolloverService,
    private val clock: Clock,
    @Value("\${server.port}") private val serverPort: String
) {

    private val logger =
        LoggerFactory.getLogger(MonthlyRolloverOrchestrator::class.java)

    /**
     * Executes the global monthly rollover when this instance
     * successfully acquires the global job.
     *
     * Returns:
     * - true  -> this instance acquired and completed the global job
     * - false -> another instance already owns the job
     *
     * Unexpected failures are propagated to the caller.
     */
    fun rollover(
        previousYearMonth: YearMonth,
        newYearMonth: YearMonth
    ): Boolean {

        require(
            newYearMonth == previousYearMonth.plusMonths(1)
        ) {
            "newYearMonth must immediately follow previousYearMonth"
        }

        val currentMonth =
            newYearMonth.toString()

        logger.info(
            "[MONTHLY-ROLLOVER] Attempting global job acquisition | yearMonth={} | instancePort={}",
            currentMonth,
            serverPort
        )

        val job =
            globalMonthlyRolloverJobRepository.tryStart(
                yearMonth = currentMonth,
                startedAt = clock.instant()
            )

        /*
         * Another application instance already owns
         * this monthly rollover job.
         *
         * This instance must NOT restore NORMAL or perform
         * any rollover recovery.
         */
        if (job == null) {
            logger.info(
                "[MONTHLY-ROLLOVER] Another instance already owns the job - EXIT | yearMonth={} | instancePort={}",
                currentMonth,
                serverPort
            )

            return false
        }

        logger.info(
            "[MONTHLY-ROLLOVER] GLOBAL JOB ACQUIRED | yearMonth={} | instancePort={} | status={}",
            currentMonth,
            serverPort,
            job.status
        )

        val executor =
            Executors.newVirtualThreadPerTaskExecutor()

        try {

            processOwners(
                executor = executor,
                previousYearMonth = previousYearMonth,
                newYearMonth = newYearMonth
            )

            /*
             * At this point all owner pages have been processed.
             *
             * Individual owner failures are intentionally isolated
             * and already recorded by PerOwnerMonthlyRolloverService.
             *
             * Only after the global orchestration itself completes
             * do we mark the global job COMPLETED.
             */
            globalMonthlyRolloverJobRepository.markCompleted(
                yearMonth = currentMonth,
                completedAt = clock.instant()
            )

            logger.info(
                "[MONTHLY-ROLLOVER] GLOBAL JOB COMPLETED | yearMonth={} | instancePort={}",
                currentMonth,
                serverPort
            )

            /*
             * true means this exact instance:
             * 1. acquired the global job
             * 2. completed global owner processing
             * 3. successfully marked the job COMPLETED
             *
             * MaintenanceService can now safely perform the
             * WRITE_BLOCKED -> NORMAL transition.
             */
            return true

        } finally {
            executor.close()
        }
    }

    private fun processOwners(
        executor: ExecutorService,
        previousYearMonth: YearMonth,
        newYearMonth: YearMonth
    ) {
        var cursor: String? = null

        while (true) {

            val page =
                ownerRepository.findPage(
                    size = OWNER_PAGE_SIZE,
                    cursor = cursor
                )

            if (page.content.isEmpty()) {
                break
            }

            val futures =
                page.content.map { ownerId ->

                    executor.submit {
                        processOwner(
                            ownerId = ownerId,
                            previousYearMonth = previousYearMonth,
                            newYearMonth = newYearMonth
                        )
                    }
                }

            waitForPage(
                futures = futures
            )

            cursor = page.nextCursor

            if (cursor == null) {
                break
            }
        }
    }

    private fun processOwner(
        ownerId: String,
        previousYearMonth: YearMonth,
        newYearMonth: YearMonth
    ) {
        try {

            perOwnerMonthlyRolloverService.rollover(
                ownerId = ownerId,
                previousYearMonth = previousYearMonth,
                newYearMonth = newYearMonth
            )

        } catch (exception: Exception) {

            /*
             * PerOwnerMonthlyRolloverService already records
             * the owner as FAILED and creates the retry entry.
             *
             * The global rollover must continue with other owners.
             */
            logger.error(
                "Monthly rollover failed for ownerId={}, yearMonth={}",
                ownerId,
                newYearMonth,
                exception
            )
        }
    }

    private fun waitForPage(
        futures: List<Future<*>>
    ) {
        futures.forEach { future ->

            try {
                future.get()

            } catch (exception: Exception) {

                /*
                 * Owner-level exceptions are already handled inside
                 * processOwner. This protects the scheduler from
                 * an unexpected task-level failure.
                 */
                logger.error(
                    "Unexpected monthly rollover task failure",
                    exception
                )
            }
        }
    }

    companion object {
        private const val OWNER_PAGE_SIZE = 100
    }
}

