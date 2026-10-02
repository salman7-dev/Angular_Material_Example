package com.clientledger.core.service.monthlyrollover

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.GlobalSummary
import com.clientledger.core.domain.MaintenanceState
import com.clientledger.core.domain.MonthlyRolloverState
import com.clientledger.core.domain.SummaryClientIndex
import com.clientledger.core.enums.MaintenanceMode
import com.clientledger.core.domain.ClientType
import com.clientledger.core.pagination.PageResult
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.lock.OwnerOperationLockRepository
import com.clientledger.core.repository.maintenance.MaintenanceRepository
import com.clientledger.core.repository.monthlyrollover.MonthlyRolloverRepository
import com.clientledger.core.repository.monthlyrollover.MonthlyRolloverRetryRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.service.client.ClientService
import com.clientledger.core.service.lock.OwnerOperationLockService
import com.clientledger.core.transaction.FirestoreTransactionExecutor
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles
import io.mockk.every
import io.mockk.mockk
import org.junit.jupiter.api.Assertions.assertThrows
import java.time.Clock
import java.time.Instant
import java.time.YearMonth
import java.time.ZoneId

@SpringBootTest
@ActiveProfiles("test")
class PerOwnerMonthlyRolloverServiceTest {


    @Autowired
    private lateinit var properties: ClientLedgerProperties

    @Autowired
    private lateinit var ownerOperationLockService: OwnerOperationLockService


    companion object {

        private lateinit var firestore: Firestore

        private val testClock = Clock.fixed(
            Instant.parse("2026-09-24T10:00:00Z"),
            ZoneId.of("UTC")
        )

        @JvmStatic
        @BeforeAll
        fun setup() {
            firestore = FirestoreOptions.newBuilder()
                .setProjectId("client-ledger-dashboard")
                .setHost("127.0.0.1:8080")
                .setEmulatorHost("127.0.0.1:8080")
                .setCredentials(NoCredentials.getInstance())
                .build()
                .service

            setMaintenanceMode(MaintenanceMode.NORMAL)
        }

        @JvmStatic
        @AfterAll
        fun cleanup() {
            firestore.close()
        }

        private fun setMaintenanceMode(mode: MaintenanceMode) {
            MaintenanceRepository(firestore).save(
                MaintenanceState(
                    mode = mode,
                    updatedAt = Instant.now()
                )
            )
        }
    }

    @Test
    fun rolloverCopiesPreviousMonthStateToCurrentMonth() {

        val ownerId = "owner-${System.nanoTime()}"

        val previousMonth =
            YearMonth.of(2026, 8)

        val currentMonth =
            YearMonth.of(2026, 9)

        setMaintenanceMode(MaintenanceMode.NORMAL)

        val clientService = createClientService()

        val client =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Rollover Client",
                    phone = "9000000001",
                    initialOpeningBalance = 5000
                )
            )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val globalSummaryRepository =
            GlobalSummaryRepository(firestore)

        val summaryIndexRepository =
            SummaryClientIndexRepository(firestore)

        val summaryIndexBucketRepository =
            SummaryClientIndexBucketRepository(
                firestore = firestore,
                properties = properties
            )

        /*
         * ---------------------------------------------------------
         * Verify previous month history exists.
         * ---------------------------------------------------------
         */
        val previousHistory =
            historyRepository.find(
                ownerId = ownerId,
                yearMonth = previousMonth.toString(),
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(previousHistory)

        assertEquals(
            5000,
            previousHistory!!.closingBalance
        )

        /*
         * ---------------------------------------------------------
         * Create a known previous-month GlobalSummary.
         *
         * These values allow us to verify exactly which fields
         * are carried forward and which fields are reset.
         * ---------------------------------------------------------
         */
        val previousSummary =
            GlobalSummary(
                ownerId = ownerId,
                yearMonth = previousMonth.toString(),

                receivableClientCount = 10,
                advanceClientCount = 2,
                settledClientCount = 3,

                totalReceivableAmount = 150000,
                totalAdvanceAmount = 25000,

                cashFlow = 50000,
                netProfit = 30000,

                totalInvoiceAmount = 70000,
                totalInvoiceAmountWithGst = 82600,
                totalGstAmount = 12600,
                totalExpenseAmount = 20000,
                totalPaymentAmount = 40000
            )

        writeGlobalSummary(previousSummary)

        /*
         * ---------------------------------------------------------
         * Verify the previous summary was written correctly.
         * ---------------------------------------------------------
         */
        val storedPreviousSummary =
            globalSummaryRepository.find(
                ownerId = ownerId,
                yearMonth = previousMonth.toString()
            )

        assertNotNull(storedPreviousSummary)

        assertEquals(
            previousSummary,
            storedPreviousSummary
        )

        /*
         * ---------------------------------------------------------
         * Create a known previous-month SummaryClientIndex.
         *
         * The bucket is intentionally obtained from the existing
         * summary index structure. Rollover must copy this exact
         * bucket ID instead of allocating a new bucket.
         * ---------------------------------------------------------
         */
        val previousIndexBucketId =
            summaryIndexBucketRepository.findBucketIds(
                ownerId = ownerId,
                yearMonth = previousMonth.toString(),
                status = ClientType.RECEIVABLE
            ).firstOrNull()
                ?: summaryIndexBucketRepository.allocateBucket(
                    ownerId = ownerId,
                    yearMonth = previousMonth.toString(),
                    status = ClientType.RECEIVABLE
                )

        val previousIndex =
            SummaryClientIndex(
                clientId = client.id,
                amount = 15000,
                status = ClientType.RECEIVABLE
            )

        summaryIndexRepository.insert(
            ownerId = ownerId,
            yearMonth = previousMonth.toString(),
            bucketId = previousIndexBucketId,
            index = previousIndex
        )

        /*
         * ---------------------------------------------------------
         * Start monthly rollover.
         * ---------------------------------------------------------
         */
        setMaintenanceMode(MaintenanceMode.WRITE_BLOCKED)

        val rolloverService =
            createRolloverService()

        rolloverService.rollover(
            ownerId = ownerId,
            previousYearMonth = previousMonth,
            newYearMonth = currentMonth
        )

        /*
         * =========================================================
         * 1. HISTORY
         * =========================================================
         */
        val currentHistory =
            historyRepository.find(
                ownerId = ownerId,
                yearMonth = currentMonth.toString(),
                bucketId = client.bucketId,
                clientId = client.id
            )

        assertNotNull(currentHistory)

        /*
         * Carry-forward values.
         */
        assertEquals(
            previousHistory.closingBalance,
            currentHistory!!.openingBalance
        )

        assertEquals(
            previousHistory.closingBalance,
            currentHistory.closingBalance
        )

        assertEquals(
            previousHistory.receivable,
            currentHistory.receivable
        )

        assertEquals(
            previousHistory.advance,
            currentHistory.advance
        )

        assertEquals(
            previousHistory.status,
            currentHistory.status
        )

        /*
         * New month activity must start at zero.
         */
        assertEquals(
            0,
            currentHistory.totalInvoiceAmount
        )

        assertEquals(
            0,
            currentHistory.totalPayments
        )

        assertEquals(
            0,
            currentHistory.totalDiscount
        )

        assertEquals(
            0,
            currentHistory.totalExpenses
        )

        assertEquals(
            0,
            currentHistory.totalGstAmount
        )

        /*
         * =========================================================
         * 2. GLOBAL SUMMARY
         * =========================================================
         */
        val currentSummary =
            globalSummaryRepository.find(
                ownerId = ownerId,
                yearMonth = currentMonth.toString()
            )

        assertNotNull(currentSummary)

        /*
         * Client financial position must carry forward.
         */
        assertEquals(
            previousSummary.receivableClientCount,
            currentSummary!!.receivableClientCount
        )

        assertEquals(
            previousSummary.advanceClientCount,
            currentSummary.advanceClientCount
        )

        assertEquals(
            previousSummary.settledClientCount,
            currentSummary.settledClientCount
        )

        assertEquals(
            previousSummary.totalReceivableAmount,
            currentSummary.totalReceivableAmount
        )

        assertEquals(
            previousSummary.totalAdvanceAmount,
            currentSummary.totalAdvanceAmount
        )

        /*
         * New month activity must reset.
         */
        assertEquals(
            0,
            currentSummary.cashFlow
        )

        assertEquals(
            0,
            currentSummary.netProfit
        )

        assertEquals(
            0,
            currentSummary.totalInvoiceAmount
        )

        assertEquals(
            0,
            currentSummary.totalInvoiceAmountWithGst
        )

        assertEquals(
            0,
            currentSummary.totalGstAmount
        )

        assertEquals(
            0,
            currentSummary.totalExpenseAmount
        )

        assertEquals(
            0,
            currentSummary.totalPaymentAmount
        )

        /*
         * =========================================================
         * 3. SUMMARY CLIENT INDEX
         * =========================================================
         *
         * The same bucket ID must exist in the new month.
         */
        val currentBucketIds =
            summaryIndexBucketRepository.findBucketIds(
                ownerId = ownerId,
                yearMonth = currentMonth.toString(),
                status = ClientType.RECEIVABLE
            )

        assert(
            currentBucketIds.contains(previousIndexBucketId)
        ) {
            "Expected rollover to copy bucket $previousIndexBucketId"
        }

        /*
         * The client index must be copied into that exact bucket.
         */
        val currentIndexes =
            summaryIndexRepository.findAllInBucket(
                ownerId = ownerId,
                yearMonth = currentMonth.toString(),
                status = ClientType.RECEIVABLE,
                bucketId = previousIndexBucketId
            )

        assertEquals(
            1,
            currentIndexes.size
        )

        val currentIndex =
            currentIndexes.single()

        assertEquals(
            previousIndex.clientId,
            currentIndex.clientId
        )

        assertEquals(
            previousIndex.amount,
            currentIndex.amount
        )

        assertEquals(
            previousIndex.status,
            currentIndex.status
        )

        /*
         * =========================================================
         * 4. ROLLOVER STATE
         * =========================================================
         */
        val rolloverRepository =
            MonthlyRolloverRepository(firestore)

        val state =
            rolloverRepository.find(
                ownerId = ownerId,
                yearMonth = currentMonth.toString()
            )

        assertNotNull(state)

        assertEquals(
            MonthlyRolloverState.Status.COMPLETED,
            state!!.status
        )

        assertEquals(
            1,
            state.attempt
        )

        assertNotNull(state.startedAt)
        assertNotNull(state.completedAt)

        /*
         * The system must be returned to NORMAL by the test
         * so other tests are not affected.
         */
        setMaintenanceMode(MaintenanceMode.NORMAL)
    }

    @Test
    fun rolloverRetriesTwiceAndSucceedsOnThirdAttempt() {

        val ownerId =
            "retry-success-${System.nanoTime()}"

        val previousMonth =
            YearMonth.of(2026, 8)

        val currentMonth =
            YearMonth.of(2026, 9)

        setMaintenanceMode(
            MaintenanceMode.WRITE_BLOCKED
        )

        val clientRepository =
            mockk<ClientRepository>()

        var findPageCalls = 0

        every {
            clientRepository.findPage(
                ownerId = ownerId,
                size = 100,
                cursor = any()
            )
        } answers {

            findPageCalls++

            when (findPageCalls) {

                1,
                2 -> {
                    throw IllegalStateException(
                        "Forced rollover failure"
                    )
                }

                else -> {
                    PageResult(
                        content = emptyList(),
                        hasNext = false,
                        nextCursor = null
                    )
                }
            }
        }

        val rolloverService =
            createRolloverService(
                clientRepository = clientRepository
            )

        rolloverService.rollover(
            ownerId = ownerId,
            previousYearMonth = previousMonth,
            newYearMonth = currentMonth
        )

        val rolloverRepository =
            MonthlyRolloverRepository(firestore)

        val state =
            rolloverRepository.find(
                ownerId = ownerId,
                yearMonth = currentMonth.toString()
            )

        assertNotNull(state)

        assertEquals(
            MonthlyRolloverState.Status.COMPLETED,
            state!!.status
        )

        assertEquals(
            3,
            state.attempt
        )

        assertEquals(
            3,
            findPageCalls
        )

        val retryRepository =
            MonthlyRolloverRetryRepository(firestore)

        val retry =
            retryRepository.find(
                yearMonth = currentMonth.toString(),
                ownerId = ownerId
            )

        assertEquals(
            null,
            retry
        )

        setMaintenanceMode(
            MaintenanceMode.NORMAL
        )
    }

    @Test
    fun rolloverFailsAfterFourAttemptsAndCreatesRetryRecord() {

        val ownerId =
            "retry-failed-${System.nanoTime()}"

        val previousMonth =
            YearMonth.of(2026, 8)

        val currentMonth =
            YearMonth.of(2026, 9)

        setMaintenanceMode(
            MaintenanceMode.WRITE_BLOCKED
        )

        val clientRepository =
            mockk<ClientRepository>()

        var findPageCalls = 0

        every {
            clientRepository.findPage(
                ownerId = ownerId,
                size = 100,
                cursor = any()
            )
        } answers {

            findPageCalls++

            throw IllegalStateException(
                "Forced rollover failure"
            )
        }

        val rolloverService =
            createRolloverService(
                clientRepository = clientRepository
            )

        assertThrows(
            IllegalStateException::class.java
        ) {
            rolloverService.rollover(
                ownerId = ownerId,
                previousYearMonth = previousMonth,
                newYearMonth = currentMonth
            )
        }

        val rolloverRepository =
            MonthlyRolloverRepository(firestore)

        val state =
            rolloverRepository.find(
                ownerId = ownerId,
                yearMonth = currentMonth.toString()
            )

        assertNotNull(state)

        assertEquals(
            MonthlyRolloverState.Status.FAILED,
            state!!.status
        )

        assertEquals(
            4,
            state.attempt
        )

        assertEquals(
            4,
            findPageCalls
        )

        val retryRepository =
            MonthlyRolloverRetryRepository(firestore)

        val retry =
            retryRepository.find(
                yearMonth = currentMonth.toString(),
                ownerId = ownerId
            )

        assertNotNull(retry)

        assertEquals(
            ownerId,
            retry!!.ownerId
        )

        assertEquals(
            currentMonth.toString(),
            retry.yearMonth
        )

        setMaintenanceMode(
            MaintenanceMode.NORMAL
        )
    }

    private fun createRolloverService(
        clientRepository: ClientRepository =
            ClientRepository(firestore)
    ): PerOwnerMonthlyRolloverService {

        val transactionExecutor =
            FirestoreTransactionExecutor(firestore)

        val historyBucketRepository =
            ClientHistoryBucketRepository(
                firestore = firestore,
                properties = properties
            )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val globalSummaryRepository =
            GlobalSummaryRepository(firestore)

        val summaryIndexBucketRepository =
            SummaryClientIndexBucketRepository(
                firestore = firestore,
                properties = properties
            )

        val summaryIndexRepository =
            SummaryClientIndexRepository(firestore)

        val maintenanceRepository =
            MaintenanceRepository(firestore)

        val ownerOperationLockRepository =
            OwnerOperationLockRepository(
                firestore = firestore,
                maintenanceRepository = maintenanceRepository
            )

        val monthlyRolloverRepository =
            MonthlyRolloverRepository(firestore)

        return PerOwnerMonthlyRolloverService(
            clientRepository = clientRepository,
            clientHistoryRepository = historyRepository,
            clientHistoryBucketRepository = historyBucketRepository,
            globalSummaryRepository = globalSummaryRepository,
            summaryClientIndexRepository = summaryIndexRepository,
            summaryClientIndexBucketRepository = summaryIndexBucketRepository,
            monthlyRolloverRepository = monthlyRolloverRepository,
            ownerOperationLockRepository = ownerOperationLockRepository,
            ownerOperationLockService = ownerOperationLockService,
            transactionExecutor = transactionExecutor,
            firestore = firestore,
            clock = testClock
        )
    }

    private fun createClientService(): ClientService {

        val transactionExecutor =
            FirestoreTransactionExecutor(firestore)

        val clientRepository =
            ClientRepository(firestore)

        val historyBucketRepository =
            ClientHistoryBucketRepository(
                firestore = firestore,
                properties = properties
            )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val globalSummaryRepository =
            GlobalSummaryRepository(firestore)

        val summaryIndexBucketRepository =
            SummaryClientIndexBucketRepository(
                firestore = firestore,
                properties = properties
            )

        val summaryIndexRepository =
            SummaryClientIndexRepository(firestore)

        val maintenanceRepository =
            MaintenanceRepository(firestore)

        val ownerOperationLockRepository =
            OwnerOperationLockRepository(
                firestore = firestore,
                maintenanceRepository = maintenanceRepository
            )

        return ClientService(
            firestore = firestore,
            transactionExecutor = transactionExecutor,
            clientRepository = clientRepository,
            historyBucketRepository = historyBucketRepository,
            historyRepository = historyRepository,
            globalSummaryRepository = globalSummaryRepository,
            summaryIndexBucketRepository = summaryIndexBucketRepository,
            summaryIndexRepository = summaryIndexRepository,
            ownerOperationLockRepository = ownerOperationLockRepository,
            properties = properties,
            clock = testClock,
            ownerOperationLockService = ownerOperationLockService
        )
    }

    private fun writeGlobalSummary(
        summary: GlobalSummary
    ) {
        firestore
            .collection("owners")
            .document(summary.ownerId)
            .collection("global_summary")
            .document(summary.yearMonth)
            .set(summary)
            .get()
    }
}
