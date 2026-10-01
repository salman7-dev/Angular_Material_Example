package com.clientledger.core.service.monthlyrollover

import com.clientledger.core.domain.MonthlyRolloverRetry
import com.clientledger.core.repository.monthlyrollover.GlobalMonthlyRolloverJobRepository
import com.clientledger.core.repository.monthlyrollover.MonthlyRolloverRetryRepository
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
    private val monthlyRolloverRetryRepository:
    MonthlyRolloverRetryRepository,
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
             * Individual owner failures were isolated and their
             * retry entries were successfully persisted.
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
             * 3. successfully recorded all failed-owner retries
             * 4. successfully marked the job COMPLETED
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

                    executor.submit<Boolean> {
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

    /**
     * Returns true when the owner processing completed safely.
     *
     * An owner rollover failure is considered safely handled only
     * when its retry entry is successfully persisted.
     *
     * Returns false when the retry entry could not be persisted.
     */
    private fun processOwner(
        ownerId: String,
        previousYearMonth: YearMonth,
        newYearMonth: YearMonth
    ): Boolean {
        try {

            perOwnerMonthlyRolloverService.rollover(
                ownerId = ownerId,
                previousYearMonth = previousYearMonth,
                newYearMonth = newYearMonth
            )

            return true

        } catch (exception: Exception) {

            logger.error(
                "Monthly rollover failed for ownerId={}, yearMonth={}",
                ownerId,
                newYearMonth,
                exception
            )

            return try {

                monthlyRolloverRetryRepository.save(
                    MonthlyRolloverRetry(
                        yearMonth = newYearMonth.toString(),
                        ownerId = ownerId,
                        failedAt = clock.instant(),
                        error = exception.message
                    )
                )

                logger.info(
                    "[MONTHLY-ROLLOVER] Failed owner recorded for recovery | ownerId={} | yearMonth={}",
                    ownerId,
                    newYearMonth
                )

                true

            } catch (retryException: Exception) {

                logger.error(
                    "[MONTHLY-ROLLOVER] CRITICAL: Failed to record owner retry | ownerId={} | yearMonth={}",
                    ownerId,
                    newYearMonth,
                    retryException
                )

                false
            }
        }
    }

    private fun waitForPage(
        futures: List<Future<Boolean>>
    ) {
        futures.forEach { future ->

            try {

                val completedSafely = future.get()

                if (!completedSafely) {
                    throw IllegalStateException(
                        "Failed to persist monthly rollover retry entry"
                    )
                }

            } catch (exception: Exception) {

                /*
                 * A failure to persist the retry entry means the
                 * failed owner could be lost from recovery.
                 *
                 * Therefore the global rollover must fail instead
                 * of marking the global job COMPLETED.
                 */
                throw IllegalStateException(
                    "Monthly rollover page processing failed",
                    exception
                )
            }
        }
    }

    companion object {
        private const val OWNER_PAGE_SIZE = 100
    }
}

