package com.clientledger.core.config

import org.springframework.boot.context.properties.ConfigurationProperties

@ConfigurationProperties(prefix = "client-ledger")
data class ClientLedgerProperties(

    val mode: ApplicationMode = ApplicationMode.EMULATOR,

    val history: HistoryProperties = HistoryProperties(),

    val summary: SummaryProperties = SummaryProperties(),

    val firestore: FireStoreProperties = FireStoreProperties(),

    val auth: AuthProperties = AuthProperties(),

    val adUnlock: AdUnlockProperties = AdUnlockProperties(),

    val trial: TrialProperties = TrialProperties(),

    val maintenance: MaintenanceProperties = MaintenanceProperties(),

    val lock: LockProperties = LockProperties(),

    val operation: OperationProperties = OperationProperties()
) {

    enum class ApplicationMode {
        EMULATOR,
        CLOUD
    }

    data class HistoryProperties(
        val editableMonths: Int = 8,
        val bucketCapacity: Int = 100
    )

    data class FireStoreProperties(
        val projectId: String = "",
        val emulatorHost: String = "127.0.0.1:8080"
    )

    data class SummaryProperties(
        val indexBucketCapacity: Int = 300
    )

    data class AuthProperties(
        val enabled: Boolean = true,
        val webApiKey: String = "",
        val emulatorHost: String = "127.0.0.1:9099",
        val credentialsPath: String = "",
        val localOwnerId: String = ""
    )

    data class AdUnlockProperties(
        val durationMinutes: Long = 10
    )

    data class TrialProperties(
        val months: Long = 3
    )

    data class MaintenanceProperties(
        val enabled: Boolean = true,
        val timezone: String = "Asia/Kolkata"
    )

    data class LockProperties(
        val leaseSeconds: Long = 30,
        val heartbeatSeconds: Long = 10,
        val acquireRetrySeconds: Long = 1,
        val acquireTimeoutSeconds: Long = 30
    )

    data class OperationProperties(
        val timeoutSeconds: Long = 300
    )
}package com.clientledger.core.service.maintenance

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

management:
  endpoints:
    web:
      exposure:
        include: health
server:
  port: 8081
client-ledger:
  mode: CLOUD  # change CLOUD OR EMULATOR if you want to use and below line uncomment in auth
  history:
    editable-months: 2
    bucket-capacity: 100
  summary:
    index-bucket-capacity: 300
  ad-unlock:
    duration-minutes: 10
  trial:
    months: 6
  maintenance:
    enabled: true
    timezone: "Asia/Kolkata"
  lock:
    lease-seconds: 30
    heartbeat-seconds: 10
    acquire-retry-seconds: 1
    acquire-timeout-seconds: 30
  operation:
    timeout-seconds: 300

    # =====================================================
    # FIREBASE EMULATOR
    # Uncomment these for cloud testing
    # always enabled true
    # =====================================================

#  firestore:
#      project-id: "client-ledger-dashboard"
#      emulator-host: "127.0.0.1:8080"
#  auth:
#      enabled: true
#      emulator-host: "127.0.0.1:9099"
#      local-owner-id: "local-owner"


    # =====================================================
    # FIREBASE CLOUD
    # Uncomment these for cloud testing
    # always enabled true
    # =====================================================
  firestore:
    project-id: "client-ledger-dashboard"
  auth:
    enabled: true
    web-api-key: "AIzaSyA94DV2tBkFFgL_8hYclLc2w_qLzmYEgdY"
    credentials-path: "classpath:client-ledger-service-account.json"
