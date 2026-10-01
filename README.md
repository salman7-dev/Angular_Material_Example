package com.clientledger.core.service.monthlyrollover

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.repository.monthlyrollover.MonthlyRolloverRetryRepository
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service
import java.time.YearMonth
import java.util.concurrent.Executors
import java.util.concurrent.TimeUnit

@Service
class FailedOwnerRecoveryService(
    private val monthlyRolloverRetryRepository:
    MonthlyRolloverRetryRepository,
    private val perOwnerMonthlyRolloverService:
    PerOwnerMonthlyRolloverService,
    private val properties: ClientLedgerProperties
) {

    private val logger =
        LoggerFactory.getLogger(FailedOwnerRecoveryService::class.java)

    fun recover(
        previousYearMonth: YearMonth,
        newYearMonth: YearMonth
    ) {
        require(
            newYearMonth == previousYearMonth.plusMonths(1)
        ) {
            "newYearMonth must immediately follow previousYearMonth"
        }

        val yearMonth = newYearMonth.toString()

        val configuration =
            properties.maintenance.failedOwnerRecovery

        require(configuration.maxAttempts > 0) {
            "maintenance.failed-owner-recovery.max-attempts must be greater than 0"
        }

        require(configuration.retryDelayMinutes >= 0) {
            "maintenance.failed-owner-recovery.retry-delay-minutes must not be negative"
        }

        repeat(configuration.maxAttempts) { attemptIndex ->

            val attempt = attemptIndex + 1

            /*
             * IMPORTANT:
             * Read the complete failed-owner list only once
             * for this recovery attempt.
             *
             * No findAll() is performed inside owner threads.
             */
            val failedOwners =
                monthlyRolloverRetryRepository.findAll(yearMonth)

            if (failedOwners.isEmpty()) {
                logger.info(
                    "[MONTHLY-ROLLOVER-RECOVERY] Recovery completed - no failed owners | yearMonth={} | attempt={}",
                    yearMonth,
                    attempt
                )
                return
            }

            logger.info(
                "[MONTHLY-ROLLOVER-RECOVERY] Recovery attempt started | yearMonth={} | attempt={}/{} | failedOwnerCount={}",
                yearMonth,
                attempt,
                configuration.maxAttempts,
                failedOwners.size
            )

            /*
             * Every owner from this single snapshot is processed
             * independently in a virtual thread.
             */
            Executors.newVirtualThreadPerTaskExecutor().use { executor ->

                val futures =
                    failedOwners.map { retry ->

                        executor.submit {
                            recoverOwner(
                                ownerId = retry.ownerId,
                                yearMonth = yearMonth,
                                previousYearMonth = previousYearMonth,
                                newYearMonth = newYearMonth
                            )
                        }
                    }

                /*
                 * Wait until every owner in this attempt has finished.
                 */
                futures.forEach { future ->
                    try {
                        future.get()
                    } catch (exception: Exception) {
                        logger.error(
                            "[MONTHLY-ROLLOVER-RECOVERY] Unexpected recovery task failure | yearMonth={}",
                            yearMonth,
                            exception
                        )
                    }
                }
            }

            logger.info(
                "[MONTHLY-ROLLOVER-RECOVERY] Recovery attempt completed | yearMonth={} | attempt={}/{}",
                yearMonth,
                attempt,
                configuration.maxAttempts
            )

            /*
             * Do not wait after the final attempt.
             * The next iteration performs the next findAll().
             */
            if (attempt < configuration.maxAttempts) {
                try {
                    logger.info(
                        "[MONTHLY-ROLLOVER-RECOVERY] Waiting before next recovery attempt | yearMonth={} | delayMinutes={}",
                        yearMonth,
                        configuration.retryDelayMinutes
                    )

                    TimeUnit.MINUTES.sleep(
                        configuration.retryDelayMinutes
                    )

                } catch (exception: InterruptedException) {
                    Thread.currentThread().interrupt()

                    throw IllegalStateException(
                        "Failed-owner recovery interrupted: yearMonth=$yearMonth",
                        exception
                    )
                }
            }
        }

        /*
         * Five attempts were completed.
         *
         * Any remaining retry documents are intentionally preserved.
         * They can be recovered by a later recovery/reconciliation flow.
         */
        logger.warn(
            "[MONTHLY-ROLLOVER-RECOVERY] Maximum recovery attempts exhausted | yearMonth={} | maxAttempts={}",
            yearMonth,
            configuration.maxAttempts
        )
    }

    private fun recoverOwner(
        ownerId: String,
        yearMonth: String,
        previousYearMonth: YearMonth,
        newYearMonth: YearMonth
    ) {
        try {

            logger.info(
                "[MONTHLY-ROLLOVER-RECOVERY] Starting owner recovery | ownerId={} | yearMonth={}",
                ownerId,
                yearMonth
            )

            /*
             * PerOwnerMonthlyRolloverService already acquires
             * the owner operation lock and validates the fence.
             */
            perOwnerMonthlyRolloverService.rollover(
                ownerId = ownerId,
                previousYearMonth = previousYearMonth,
                newYearMonth = newYearMonth
            )


            logger.info(
                "[MONTHLY-ROLLOVER-RECOVERY] Owner recovery SUCCESS | ownerId={} | yearMonth={}",
                ownerId,
                yearMonth
            )

        } catch (exception: Exception) {

            /*
             * Keep the retry entry.
             *
             * It will be picked up by the next recovery attempt.
             */
            logger.error(
                "[MONTHLY-ROLLOVER-RECOVERY] Owner recovery FAILED | ownerId={} | yearMonth={}",
                ownerId,
                yearMonth,
                exception
            )
        }
    }
}

