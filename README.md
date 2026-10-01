Type mismatch.
Required:
GlobalMonthlyRolloverJob
Found:
Unit

package com.clientledger.core.service.monthlyrollover

import com.clientledger.core.pagination.PageResult
import com.clientledger.core.repository.monthlyrollover.GlobalMonthlyRolloverJobRepository
import com.clientledger.core.repository.owner.OwnerRepository
import io.mockk.every
import io.mockk.mockk
import io.mockk.verify
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.Test
import java.time.Clock
import java.time.Instant
import java.time.YearMonth
import java.time.ZoneId
import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.CountDownLatch
import java.util.concurrent.TimeUnit

class MonthlyRolloverOrchestratorTest {

    private val testClock =
        Clock.fixed(
            Instant.parse("2026-09-24T10:00:00Z"),
            ZoneId.of("UTC")
        )

    private val previousMonth =
        YearMonth.of(2026, 8)

    private val currentMonth =
        YearMonth.of(2026, 9)

    @Test
    fun globalRolloverCompletesWhenOneOwnerFails() {

        val ownerRepository =
            mockk<OwnerRepository>()

        val globalJobRepository =
            mockk<GlobalMonthlyRolloverJobRepository>()

        val rolloverService =
            mockk<PerOwnerMonthlyRolloverService>()

        val owners =
            listOf(
                "owner-success-1",
                "owner-failed",
                "owner-success-2"
            )

        every {
            globalJobRepository.tryStart(
                yearMonth = currentMonth.toString(),
                startedAt = testClock.instant()
            )
        } returns mockk(relaxed = true)

        every {
            globalJobRepository.markCompleted(
                yearMonth = currentMonth.toString(),
                completedAt = testClock.instant()
            )
        } returns Unit

        every {
            ownerRepository.findPage(
                size = 100,
                cursor = any()
            )
        } returns PageResult(
            content = owners,
            hasNext = false,
            nextCursor = null
        )

        every {
            rolloverService.rollover(
                ownerId = "owner-success-1",
                previousYearMonth = previousMonth,
                newYearMonth = currentMonth
            )
        } returns Unit

        every {
            rolloverService.rollover(
                ownerId = "owner-failed",
                previousYearMonth = previousMonth,
                newYearMonth = currentMonth
            )
        } throws IllegalStateException(
            "Forced owner rollover failure"
        )

        every {
            rolloverService.rollover(
                ownerId = "owner-success-2",
                previousYearMonth = previousMonth,
                newYearMonth = currentMonth
            )
        } returns Unit

        val orchestrator =
            MonthlyRolloverOrchestrator(
                ownerRepository = ownerRepository,
                globalMonthlyRolloverJobRepository =
                globalJobRepository,
                perOwnerMonthlyRolloverService =
                rolloverService,
                clock = testClock,
                serverPort = "8081"
            )

        val result =
            orchestrator.rollover(
                previousYearMonth = previousMonth,
                newYearMonth = currentMonth
            )

        assertTrue(result)

        /*
         * Every owner must be processed even though one owner failed.
         */
        verify(exactly = 1) {
            rolloverService.rollover(
                ownerId = "owner-success-1",
                previousYearMonth = previousMonth,
                newYearMonth = currentMonth
            )
        }

        verify(exactly = 1) {
            rolloverService.rollover(
                ownerId = "owner-failed",
                previousYearMonth = previousMonth,
                newYearMonth = currentMonth
            )
        }

        verify(exactly = 1) {
            rolloverService.rollover(
                ownerId = "owner-success-2",
                previousYearMonth = previousMonth,
                newYearMonth = currentMonth
            )
        }

        /*
         * The global job must still be marked COMPLETED.
         */
        verify(exactly = 1) {
            globalJobRepository.markCompleted(
                yearMonth = currentMonth.toString(),
                completedAt = testClock.instant()
            )
        }
    }

    @Test
    fun globalRolloverProcessesOwnersInParallel() {

        val ownerRepository =
            mockk<OwnerRepository>()

        val globalJobRepository =
            mockk<GlobalMonthlyRolloverJobRepository>()

        val rolloverService =
            mockk<PerOwnerMonthlyRolloverService>()

        val owners =
            listOf(
                "owner-1",
                "owner-2",
                "owner-3",
                "owner-4"
            )

        val startedOwners =
            ConcurrentHashMap.newKeySet<String>()

        val allOwnersStarted =
            CountDownLatch(owners.size)

        val releaseOwners =
            CountDownLatch(1)

        every {
            globalJobRepository.tryStart(
                yearMonth = currentMonth.toString(),
                startedAt = testClock.instant()
            )
        } returns mockk(relaxed = true)

        every {
            globalJobRepository.markCompleted(
                yearMonth = currentMonth.toString(),
                completedAt = testClock.instant()
            )
        } returns Unit

        every {
            ownerRepository.findPage(
                size = 100,
                cursor = any()
            )
        } returns PageResult(
            content = owners,
            hasNext = false,
            nextCursor = null
        )

        owners.forEach { ownerId ->

            every {
                rolloverService.rollover(
                    ownerId = ownerId,
                    previousYearMonth = previousMonth,
                    newYearMonth = currentMonth
                )
            } answers {

                startedOwners.add(ownerId)

                allOwnersStarted.countDown()

                releaseOwners.await(
                    5,
                    TimeUnit.SECONDS
                )

                Unit
            }
        }

        val orchestrator =
            MonthlyRolloverOrchestrator(
                ownerRepository = ownerRepository,
                globalMonthlyRolloverJobRepository =
                globalJobRepository,
                perOwnerMonthlyRolloverService =
                rolloverService,
                clock = testClock,
                serverPort = "8081"
            )

        val rolloverThread =
            Thread {

                orchestrator.rollover(
                    previousYearMonth = previousMonth,
                    newYearMonth = currentMonth
                )
            }

        rolloverThread.start()

        val allStarted =
            allOwnersStarted.await(
                5,
                TimeUnit.SECONDS
            )

        assertTrue(
            allStarted,
            "Expected all owners to start processing in parallel"
        )

        assertEquals(
            owners.toSet(),
            startedOwners
        )

        releaseOwners.countDown()

        rolloverThread.join(5000)

        assertTrue(
            !rolloverThread.isAlive,
            "Global rollover should finish after all owners complete"
        )

        verify(exactly = 1) {
            globalJobRepository.markCompleted(
                yearMonth = currentMonth.toString(),
                completedAt = testClock.instant()
            )
        }
    }

    @Test
    fun globalRolloverReturnsFalseWhenAnotherInstanceOwnsJob() {

        val ownerRepository =
            mockk<OwnerRepository>()

        val globalJobRepository =
            mockk<GlobalMonthlyRolloverJobRepository>()

        val rolloverService =
            mockk<PerOwnerMonthlyRolloverService>()

        every {
            globalJobRepository.tryStart(
                yearMonth = currentMonth.toString(),
                startedAt = testClock.instant()
            )
        } returns null

        val orchestrator =
            MonthlyRolloverOrchestrator(
                ownerRepository = ownerRepository,
                globalMonthlyRolloverJobRepository =
                globalJobRepository,
                perOwnerMonthlyRolloverService =
                rolloverService,
                clock = testClock,
                serverPort = "8081"
            )

        val result =
            orchestrator.rollover(
                previousYearMonth = previousMonth,
                newYearMonth = currentMonth
            )

        assertEquals(
            false,
            result
        )

        /*
         * No owner should be processed when this instance
         * does not acquire the global job.
         */
        verify(exactly = 0) {
            ownerRepository.findPage(
                size = 100,
                cursor = any()
            )
        }

        verify(exactly = 0) {
            rolloverService.rollover(
                ownerId = any(),
                previousYearMonth = any(),
                newYearMonth = any()
            )
        }

        verify(exactly = 0) {
            globalJobRepository.markCompleted(
                yearMonth = any(),
                completedAt = any()
            )
        }
    }

    @Test
    fun globalRolloverProcessesMultipleOwnerPages() {

        val ownerRepository =
            mockk<OwnerRepository>()

        val globalJobRepository =
            mockk<GlobalMonthlyRolloverJobRepository>()

        val rolloverService =
            mockk<PerOwnerMonthlyRolloverService>()

        val firstPageOwners =
            listOf(
                "owner-page-1",
                "owner-page-2"
            )

        val secondPageOwners =
            listOf(
                "owner-page-3"
            )

        every {
            globalJobRepository.tryStart(
                yearMonth = currentMonth.toString(),
                startedAt = testClock.instant()
            )
        } returns mockk(relaxed = true)

        every {
            globalJobRepository.markCompleted(
                yearMonth = currentMonth.toString(),
                completedAt = testClock.instant()
            )
        } returns Unit

        every {
            ownerRepository.findPage(
                size = 100,
                cursor = null
            )
        } returns PageResult(
            content = firstPageOwners,
            hasNext = true,
            nextCursor = "page-2"
        )

        every {
            ownerRepository.findPage(
                size = 100,
                cursor = "page-2"
            )
        } returns PageResult(
            content = secondPageOwners,
            hasNext = false,
            nextCursor = null
        )

        every {
            rolloverService.rollover(
                ownerId = any(),
                previousYearMonth = previousMonth,
                newYearMonth = currentMonth
            )
        } returns Unit

        val orchestrator =
            MonthlyRolloverOrchestrator(
                ownerRepository = ownerRepository,
                globalMonthlyRolloverJobRepository =
                globalJobRepository,
                perOwnerMonthlyRolloverService =
                rolloverService,
                clock = testClock,
                serverPort = "8081"
            )

        val result =
            orchestrator.rollover(
                previousYearMonth = previousMonth,
                newYearMonth = currentMonth
            )

        assertTrue(result)

        verify(exactly = 1) {
            ownerRepository.findPage(
                size = 100,
                cursor = null
            )
        }

        verify(exactly = 1) {
            ownerRepository.findPage(
                size = 100,
                cursor = "page-2"
            )
        }

        firstPageOwners
            .plus(secondPageOwners)
            .forEach { ownerId ->

                verify(exactly = 1) {
                    rolloverService.rollover(
                        ownerId = ownerId,
                        previousYearMonth = previousMonth,
                        newYearMonth = currentMonth
                    )
                }
            }

        verify(exactly = 1) {
            globalJobRepository.markCompleted(
                yearMonth = currentMonth.toString(),
                completedAt = testClock.instant()
            )
        }
    }
}

