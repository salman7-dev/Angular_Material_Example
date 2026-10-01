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

    private val logger = LoggerFactory.getLogger(MonthlyRolloverOrchestrator::class.java)

    fun rollover(
        previousYearMonth: YearMonth,
        newYearMonth: YearMonth
    ) {

        require(
            newYearMonth == previousYearMonth.plusMonths(1)
        ) {
            "newYearMonth must immediately follow previousYearMonth"
        }

        val currentMonth =
            newYearMonth.toString()

        logger.info( "[MONTHLY-ROLLOVER] Attempting global job acquisition | yearMonth={} | instancePort={}", currentMonth, serverPort )

        val job =
            globalMonthlyRolloverJobRepository.tryStart(
                yearMonth = currentMonth,
                startedAt = clock.instant()
            )

        /*
         * Another application instance already owns
         * this monthly rollover job.
         */
        if (job == null) {
            logger.info( "[MONTHLY-ROLLOVER] Another instance already owns the job - EXIT | yearMonth={} | instancePort={}", currentMonth, serverPort )
            return
        }

        logger.info( "[MONTHLY-ROLLOVER] GLOBAL JOB ACQUIRED | yearMonth={} | instancePort={} | status={}", currentMonth, serverPort, job.status)

        val executor =
            Executors.newVirtualThreadPerTaskExecutor()

        try {
            processOwners(
                executor = executor,
                previousYearMonth = previousYearMonth,
                newYearMonth = newYearMonth
            )

            globalMonthlyRolloverJobRepository.markCompleted(
                yearMonth = currentMonth,
                completedAt = clock.instant()
            )

            logger.info( "[MONTHLY-ROLLOVER] GLOBAL JOB COMPLETED | yearMonth={} | instancePort={}", currentMonth, serverPort )
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

package com.clientledger.core.repository.monthlyrollover

import com.clientledger.core.domain.GlobalMonthlyRolloverJob
import com.google.cloud.Timestamp
import com.google.cloud.firestore.DocumentReference
import com.google.cloud.firestore.DocumentSnapshot
import com.google.cloud.firestore.Firestore
import org.springframework.stereotype.Repository
import java.time.Instant

@Repository
class GlobalMonthlyRolloverJobRepository(
    private val firestore: Firestore
) {

    private fun jobDocument(
        yearMonth: String
    ): DocumentReference {

        validateYearMonth(yearMonth)

        return firestore
            .collection("system")
            .document("monthly_rollover")
            .collection(yearMonth)
            .document("job")
    }

    fun find(
        yearMonth: String
    ): GlobalMonthlyRolloverJob? {

        val snapshot =
            jobDocument(yearMonth)
                .get()
                .get()

        if (!snapshot.exists()) {
            return null
        }

        return toJob(snapshot)
    }

    fun tryStart(
        yearMonth: String,
        startedAt: Instant
    ): GlobalMonthlyRolloverJob? {

        val reference =
            jobDocument(yearMonth)

        return firestore
            .runTransaction { transaction ->

                val existing =
                    transaction
                        .get(reference)
                        .get()

                /*
                 * Another application instance already owns
                 * the scheduler execution for this month.
                 */
                if (existing.exists()) {
                    return@runTransaction null
                }

                val job =
                    GlobalMonthlyRolloverJob(
                        yearMonth = yearMonth,
                        status = GlobalMonthlyRolloverJob.Status.RUNNING,
                        startedAt = startedAt
                    )

                transaction.set(
                    reference,
                    toDocument(job)
                )

                job
            }
            .get()
    }

    fun markCompleted(
        yearMonth: String,
        completedAt: Instant
    ): GlobalMonthlyRolloverJob {

        val reference =
            jobDocument(yearMonth)

        return firestore
            .runTransaction { transaction ->

                val snapshot =
                    transaction
                        .get(reference)
                        .get()

                require(snapshot.exists()) {
                    "Global monthly rollover job does not exist: $yearMonth"
                }

                val current =
                    toJob(snapshot)

                require(
                    current.status ==
                            GlobalMonthlyRolloverJob.Status.RUNNING
                ) {
                    "Global monthly rollover job must be RUNNING before completion: $yearMonth"
                }

                val completed =
                    current.copy(
                        status = GlobalMonthlyRolloverJob.Status.COMPLETED,
                        completedAt = completedAt
                    )

                transaction.set(
                    reference,
                    toDocument(completed)
                )

                completed
            }
            .get()
    }

    private fun toDocument(
        job: GlobalMonthlyRolloverJob
    ): Map<String, Any?> {

        return mapOf(
            "yearMonth" to job.yearMonth,
            "status" to job.status.name,
            "startedAt" to job.startedAt.let(::toTimestamp),
            "completedAt" to job.completedAt?.let(::toTimestamp)
        )
    }

    private fun toJob(
        snapshot: DocumentSnapshot
    ): GlobalMonthlyRolloverJob {

        val yearMonth =
            snapshot.getString("yearMonth")
                ?: throw IllegalStateException(
                    "Global monthly rollover job is missing yearMonth"
                )

        val status =
            snapshot.getString("status")
                ?.let(GlobalMonthlyRolloverJob.Status::valueOf)
                ?: throw IllegalStateException(
                    "Global monthly rollover job is missing status"
                )

        val startedAt =
            snapshot.getTimestamp("startedAt")
                ?.toDate()
                ?.toInstant()
                ?: throw IllegalStateException(
                    "Global monthly rollover job is missing startedAt"
                )

        return GlobalMonthlyRolloverJob(
            yearMonth = yearMonth,
            status = status,
            startedAt = startedAt,
            completedAt = snapshot.getTimestamp("completedAt")
                ?.toDate()
                ?.toInstant()
        )
    }

    private fun toTimestamp(
        instant: Instant
    ): Timestamp =
        Timestamp.ofTimeSecondsAndNanos(
            instant.epochSecond,
            instant.nano
        )

    private fun validateYearMonth(
        yearMonth: String
    ) {
        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }
    }
}

package com.clientledger.core.service.monthlyrollover

import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.MonthlyRolloverState
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.lock.OwnerOperationLockRepository
import com.clientledger.core.repository.monthlyrollover.MonthlyRolloverRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.service.lock.OwnerOperationLockService
import com.clientledger.core.transaction.FirestoreTransactionExecutor
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Service
import java.time.Clock
import java.time.YearMonth

@Service
class PerOwnerMonthlyRolloverService(
    private val clientRepository: ClientRepository,
    private val clientHistoryRepository: ClientHistoryRepository,
    private val clientHistoryBucketRepository: ClientHistoryBucketRepository,
    private val globalSummaryRepository: GlobalSummaryRepository,
    private val summaryClientIndexRepository: SummaryClientIndexRepository,
    private val summaryClientIndexBucketRepository:
    SummaryClientIndexBucketRepository,
    private val monthlyRolloverRepository: MonthlyRolloverRepository,
    private val ownerOperationLockRepository:
    OwnerOperationLockRepository,
    private val ownerOperationLockService:
    OwnerOperationLockService,
    private val transactionExecutor:
    FirestoreTransactionExecutor,
    private val firestore: Firestore,
    private val clock: Clock
) {

    private val logger =
        LoggerFactory.getLogger(PerOwnerMonthlyRolloverService::class.java)

    fun rollover(
        ownerId: String,
        previousYearMonth: YearMonth,
        newYearMonth: YearMonth
    ) {
        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(newYearMonth == previousYearMonth.plusMonths(1)) {
            "newYearMonth must immediately follow previousYearMonth"
        }

        val previousMonth = previousYearMonth.toString()
        val currentMonth = newYearMonth.toString()

        ownerOperationLockService.executeLongRunning(ownerId) { fence ->

            ensurePending(
                ownerId = ownerId,
                yearMonth = currentMonth
            )

            val existing =
                monthlyRolloverRepository.find(
                    ownerId = ownerId,
                    yearMonth = currentMonth
                )

            if (existing?.status ==
                MonthlyRolloverState.Status.COMPLETED
            ) {
                logger.info(
                    "Monthly rollover already completed: ownerId={}, yearMonth={}",
                    ownerId,
                    currentMonth
                )
                return@executeLongRunning
            }

            monthlyRolloverRepository.markRunning(
                ownerId = ownerId,
                yearMonth = currentMonth,
                startedAt = clock.instant()
            )

            try {
                copyHistory(
                    ownerId = ownerId,
                    previousMonth = previousMonth,
                    currentMonth = currentMonth,
                    fence = fence
                )

                copyGlobalSummary(
                    ownerId = ownerId,
                    previousMonth = previousMonth,
                    currentMonth = currentMonth,
                    fence = fence
                )

                copySummaryIndexes(
                    ownerId = ownerId,
                    previousMonth = previousMonth,
                    currentMonth = currentMonth,
                    fence = fence
                )

                monthlyRolloverRepository.markCompleted(
                    ownerId = ownerId,
                    yearMonth = currentMonth,
                    completedAt = clock.instant()
                )

                logger.info(
                    "Monthly rollover completed: ownerId={}, yearMonth={}",
                    ownerId,
                    currentMonth
                )
            } catch (exception: Exception) {

                try {
                    monthlyRolloverRepository.markFailed(
                        ownerId = ownerId,
                        yearMonth = currentMonth,
                        failedAt = clock.instant(),
                        error = exception.message
                    )
                } catch (stateException: Exception) {
                    logger.error(
                        "Failed to mark monthly rollover as FAILED: ownerId={}, yearMonth={}",
                        ownerId,
                        currentMonth,
                        stateException
                    )
                }

                throw exception
            }
        }
    }

    private fun ensurePending(
        ownerId: String,
        yearMonth: String
    ) {
        if (monthlyRolloverRepository.find(ownerId, yearMonth) == null) {
            monthlyRolloverRepository.createPending(
                ownerId = ownerId,
                yearMonth = yearMonth
            )
        }
    }

    private fun copyHistory(
        ownerId: String,
        previousMonth: String,
        currentMonth: String,
        fence: Long
    ) {
        var cursor: String? = null

        while (true) {
            val page =
                clientRepository.findPage(
                    ownerId = ownerId,
                    size = HISTORY_CLIENT_PAGE_SIZE,
                    cursor = cursor
                )

            if (page.content.isEmpty()) {
                break
            }

            page.content.forEach { client ->

                val previousHistory =
                    clientHistoryRepository.find(
                        ownerId = ownerId,
                        yearMonth = previousMonth,
                        bucketId = client.bucketId,
                        clientId = client.id
                    ) ?: return@forEach

                transactionExecutor.execute { transaction ->

                    requireFence(
                        transaction = transaction,
                        ownerId = ownerId,
                        fence = fence
                    )

                    clientHistoryRepository.copyToNewMonthInTransaction(
                        transaction = transaction,
                        previousHistory = previousHistory,
                        bucketId = client.bucketId,
                        newYearMonth = currentMonth
                    )
                }
            }

            cursor = page.nextCursor

            if (cursor == null) {
                break
            }
        }

        /*
         * Bucket metadata must be copied only once.
         *
         * ClientRepository pagination is independent from history bucket
         * pagination. Calling this inside the client-page loop would copy
         * the same source buckets repeatedly when an owner has multiple
         * client pages.
         */
        copyHistoryBuckets(
            ownerId = ownerId,
            previousMonth = previousMonth,
            currentMonth = currentMonth,
            fence = fence
        )
    }

    private fun copyHistoryBuckets(
        ownerId: String,
        previousMonth: String,
        currentMonth: String,
        fence: Long
    ) {
        val bucketIds =
            clientHistoryBucketRepository.findBucketIds(
                ownerId = ownerId,
                yearMonth = previousMonth
            )

        bucketIds.forEach { bucketId ->

            transactionExecutor.execute { transaction ->

                requireFence(
                    transaction = transaction,
                    ownerId = ownerId,
                    fence = fence
                )

                clientHistoryBucketRepository.copyBucketInTransaction(
                    transaction = transaction,
                    ownerId = ownerId,
                    previousYearMonth = previousMonth,
                    newYearMonth = currentMonth,
                    bucketId = bucketId
                )
            }
        }
    }

    private fun copyGlobalSummary(
        ownerId: String,
        previousMonth: String,
        currentMonth: String,
        fence: Long
    ) {
        val previousSummary =
            globalSummaryRepository.find(
                ownerId = ownerId,
                yearMonth = previousMonth
            ) ?: return

        transactionExecutor.execute { transaction ->

            requireFence(
                transaction = transaction,
                ownerId = ownerId,
                fence = fence
            )

            globalSummaryRepository.copyToNewMonthInTransaction(
                transaction = transaction,
                previousSummary = previousSummary,
                newYearMonth = currentMonth
            )
        }
    }

    private fun copySummaryIndexes(
        ownerId: String,
        previousMonth: String,
        currentMonth: String,
        fence: Long
    ) {
        ClientType.entries.forEach { clientType ->

            val bucketIds =
                summaryClientIndexBucketRepository.findBucketIds(
                    ownerId = ownerId,
                    yearMonth = previousMonth,
                    status = clientType
                )

            bucketIds.forEach { bucketId ->

                val indexes =
                    summaryClientIndexRepository.findAllInBucket(
                        ownerId = ownerId,
                        yearMonth = previousMonth,
                        status = clientType,
                        bucketId = bucketId
                    )

                transactionExecutor.execute { transaction ->

                    requireFence(
                        transaction = transaction,
                        ownerId = ownerId,
                        fence = fence
                    )

                    summaryClientIndexBucketRepository.copyBucketInTransaction(
                        transaction = transaction,
                        ownerId = ownerId,
                        previousYearMonth = previousMonth,
                        newYearMonth = currentMonth,
                        status = clientType,
                        bucketId = bucketId
                    )

                    indexes.forEach { index ->

                        summaryClientIndexRepository.copyInTransaction(
                            transaction = transaction,
                            ownerId = ownerId,
                            newYearMonth = currentMonth,
                            status = clientType,
                            bucketId = bucketId,
                            index = index
                        )
                    }
                }
            }
        }
    }

    private fun requireFence(
        transaction: Transaction,
        ownerId: String,
        fence: Long
    ) {
        require(
            ownerOperationLockRepository.isFenceValidInTransaction(
                transaction = transaction,
                ownerId = ownerId,
                expectedFence = fence
            )
        ) {
            "Owner operation lock fence changed: $ownerId"
        }
    }

    companion object {
        private const val HISTORY_CLIENT_PAGE_SIZE = 100
    }
}

package com.clientledger.core.service.maintenance

import com.clientledger.core.domain.MaintenanceState
import com.clientledger.core.enums.MaintenanceMode
import com.clientledger.core.repository.maintenance.MaintenanceRepository
import com.clientledger.core.service.monthlyrollover.MonthlyRolloverOrchestrator
import org.springframework.stereotype.Service
import java.time.Clock
import java.time.Instant
import java.time.YearMonth

@Service
class MaintenanceService(
    private val maintenanceRepository: MaintenanceRepository,
    private val monthlyRolloverOrchestrator: MonthlyRolloverOrchestrator,
    private val clock: Clock
) {

    @Volatile
    private var currentState: MaintenanceState =
        maintenanceRepository.find()
            ?: MaintenanceState(
                mode = MaintenanceMode.NORMAL,
                updatedAt = Instant.now(clock)
            )

    fun getCurrentState(): MaintenanceState {
        return currentState
    }

    fun blockWrites(reason: String? = null): MaintenanceState {
        val state = MaintenanceState(
            mode = MaintenanceMode.WRITE_BLOCKED,
            updatedAt = Instant.now(clock),
            reason = reason
        )

        maintenanceRepository.save(state)
        currentState = state

        return state
    }

    /**
     * Attempts to atomically establish the monthly write freeze.
     *
     * Returns true only for the instance that successfully
     * established WRITE_BLOCKED in Firestore.
     */
    fun tryBlockWrites(reason: String? = null): Boolean {

        val state = MaintenanceState(
            mode = MaintenanceMode.WRITE_BLOCKED,
            updatedAt = Instant.now(clock),
            reason = reason
        )

        val acquired = maintenanceRepository.tryBlockWrites(state)

        if (acquired) {
            currentState = state
            return true
        }

        /*
         * Another instance already established the maintenance state.
         * Refresh this instance's local state so its HTTP gate also
         * reflects the distributed Firestore state.
         */
        currentState =
            maintenanceRepository.find()
                ?: MaintenanceState(
                    mode = MaintenanceMode.NORMAL,
                    updatedAt = Instant.now(clock)
                )

        return false
    }

    fun performMonthlyRollover() {
        val currentYearMonth = YearMonth.now(clock)
        val previousYearMonth = currentYearMonth.minusMonths(1)

        monthlyRolloverOrchestrator.rollover(
            previousYearMonth = previousYearMonth,
            newYearMonth = currentYearMonth
        )
    }

    fun restoreNormal(reason: String? = null): MaintenanceState {
        val state = MaintenanceState(
            mode = MaintenanceMode.NORMAL,
            updatedAt = Instant.now(clock),
            reason = reason
        )

        maintenanceRepository.save(state)
        currentState = state

        return state
    }
}

package com.clientledger.core.repository.maintenance

import com.clientledger.core.domain.MaintenanceState
import com.clientledger.core.enums.MaintenanceMode
import com.google.cloud.firestore.DocumentReference
import com.google.cloud.firestore.DocumentSnapshot
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Repository

@Repository
class MaintenanceRepository(
    private val firestore: Firestore
) {

    private fun maintenanceDocument(): DocumentReference {
        return firestore
            .collection("system")
            .document("maintenance")
    }

    fun find(): MaintenanceState? {

        val snapshot = maintenanceDocument()
            .get()
            .get()

        if (!snapshot.exists()) {
            return null
        }

        return toMaintenanceState(snapshot)
    }

    fun save(state: MaintenanceState) {

        maintenanceDocument()
            .set(
                mapOf(
                    "mode" to state.mode.name,
                    "updatedAt" to state.updatedAt,
                    "reason" to state.reason
                )
            )
            .get()
    }

    /**
     * Atomically establishes WRITE_BLOCKED.
     *
     * Returns true when this instance successfully established
     * the write freeze.
     *
     * Returns false when another instance has already established
     * a maintenance state.
     */
    fun tryBlockWrites(state: MaintenanceState): Boolean {

        val reference = maintenanceDocument()

        return firestore
            .runTransaction { transaction ->

                val snapshot = transaction
                    .get(reference)
                    .get()

                if (snapshot.exists()) {

                    val currentMode =
                        snapshot.getString("mode")
                            ?.let { MaintenanceMode.valueOf(it) }
                            ?: MaintenanceMode.NORMAL

                    if (currentMode != MaintenanceMode.NORMAL) {
                        return@runTransaction false
                    }
                }

                transaction.set(
                    reference,
                    mapOf(
                        "mode" to state.mode.name,
                        "updatedAt" to state.updatedAt,
                        "reason" to state.reason
                    )
                )

                true
            }
            .get()
    }

    private fun toMaintenanceState(
        snapshot: DocumentSnapshot
    ): MaintenanceState {

        val mode = snapshot.getString("mode")
            ?.let { MaintenanceMode.valueOf(it) }
            ?: MaintenanceMode.NORMAL

        val updatedAt = snapshot.getTimestamp("updatedAt")
            ?.toDate()
            ?.toInstant()
            ?: throw IllegalStateException(
                "Maintenance state is missing updatedAt"
            )

        return MaintenanceState(
            mode = mode,
            updatedAt = updatedAt,
            reason = snapshot.getString("reason")
        )
    }

    fun getModeInTransaction(
        transaction: Transaction
    ): MaintenanceMode {

        val snapshot =
            transaction
                .get(maintenanceDocument())
                .get()

        if (!snapshot.exists()) {
            return MaintenanceMode.NORMAL
        }

        return snapshot.getString("mode")
            ?.let { MaintenanceMode.valueOf(it) }
            ?: MaintenanceMode.NORMAL
    }
}

