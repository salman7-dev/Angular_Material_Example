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
        val timezone: String = "Asia/Kolkata",
        val restoreNormal: RestoreNormalProperties = RestoreNormalProperties()
    ) {
        data class RestoreNormalProperties(
            val maxAttempts: Int = 6,
            val retryDelaySeconds: Long = 10
        )
    }

    data class LockProperties(
        val leaseSeconds: Long = 30,
        val heartbeatSeconds: Long = 10,
        val acquireRetrySeconds: Long = 1,
        val acquireTimeoutSeconds: Long = 30
    )

    data class OperationProperties(
        val timeoutSeconds: Long = 300
    )
}

package com.clientledger.core.config

import com.google.auth.oauth2.GoogleCredentials
import com.google.firebase.FirebaseApp
import com.google.firebase.FirebaseOptions
import com.google.firebase.auth.FirebaseAuth
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.io.ClassPathResource
import java.io.FileInputStream

@Configuration
@ConditionalOnProperty(
    prefix = "client-ledger",
    name = ["mode"],
    havingValue = "CLOUD"
)
class FirebaseCloudConfig(
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun firebaseApp(): FirebaseApp {

        FirebaseApp.getApps()
            .firstOrNull()
            ?.let { return it }

        val credentialsPath =
            properties.auth.credentialsPath

        require(credentialsPath.isNotBlank()) {
            "client-ledger.auth.credentials-path must not be blank when mode is CLOUD"
        }

        val credentials =
            if (credentialsPath.startsWith("classpath:")) {

                val resourcePath =
                    credentialsPath.removePrefix("classpath:")

                val resource =
                    ClassPathResource(resourcePath)

                require(resource.exists()) {
                    "Firebase service account file not found on classpath: $resourcePath"
                }

                resource.inputStream.use { inputStream ->
                    GoogleCredentials.fromStream(inputStream)
                }

            } else {

                FileInputStream(credentialsPath).use { inputStream ->
                    GoogleCredentials.fromStream(inputStream)
                }
            }

        println(
            """
            =================================================
             FIREBASE CONFIGURATION
            =================================================
             Environment : CLOUD
             Project ID  : ${properties.firestore.projectId}
             Auth        : FIREBASE CLOUD
             Firestore   : FIREBASE CLOUD
            =================================================
            """.trimIndent()
        )

        val options =
            FirebaseOptions.builder()
                .setProjectId(properties.firestore.projectId)
                .setCredentials(credentials)
                .build()

        return FirebaseApp.initializeApp(
            options,
            FirebaseApp.DEFAULT_APP_NAME
        ) ?: error("Failed to initialize Firebase Cloud")
    }

    @Bean
    fun firebaseAuth(
        firebaseApp: FirebaseApp
    ): FirebaseAuth {
        return FirebaseAuth.getInstance(firebaseApp)
    }
}package com.clientledger.core.config

import com.google.auth.oauth2.AccessToken
import com.google.auth.oauth2.GoogleCredentials
import com.google.firebase.FirebaseApp
import com.google.firebase.FirebaseOptions
import com.google.firebase.auth.FirebaseAuth
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.util.Date

@Configuration
@ConditionalOnProperty(
    prefix = "client-ledger",
    name = ["mode"],
    havingValue = "EMULATOR"
)
class FirebaseEmulatorConfig(
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun firebaseApp(): FirebaseApp {

        FirebaseApp.getApps()
            .firstOrNull()
            ?.let { return it }

        val emulatorHost =
            properties.auth.emulatorHost.ifBlank {
                "127.0.0.1:9099"
            }

        val projectId =
            properties.firestore.projectId

        require(projectId.isNotBlank()) {
            "client-ledger.firestore.project-id must not be blank when mode is EMULATOR"
        }

        println(
            """
            =================================================
             FIREBASE CONFIGURATION
            =================================================
             Environment : EMULATOR
             Project ID  : $projectId
             Auth        : FIREBASE AUTH EMULATOR ($emulatorHost)
             Firestore   : FIRESTORE EMULATOR (${properties.firestore.emulatorHost})
            =================================================
            """.trimIndent()
        )

        val emulatorCredentials =
            object : GoogleCredentials() {

                override fun refreshAccessToken(): AccessToken {
                    return AccessToken(
                        "firebase-emulator-token",
                        Date(System.currentTimeMillis() + 60 * 60 * 1000)
                    )
                }
            }

        val options =
            FirebaseOptions.builder()
                .setProjectId(projectId)
                .setCredentials(emulatorCredentials)
                .build()

        return FirebaseApp.initializeApp(
            options,
            FirebaseApp.DEFAULT_APP_NAME
        ) ?: error("Failed to initialize Firebase Emulator")
    }

    @Bean
    fun firebaseAuth(
        firebaseApp: FirebaseApp
    ): FirebaseAuth {
        return FirebaseAuth.getInstance(firebaseApp)
    }
}
package com.clientledger.core.config

import com.google.auth.oauth2.GoogleCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.io.ClassPathResource
import java.io.FileInputStream

@Configuration
@ConditionalOnProperty(
    prefix = "client-ledger",
    name = ["mode"],
    havingValue = "CLOUD"
)
class FireStoreCloudConfig(
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun firestore(): Firestore {

        val projectId =
            properties.firestore.projectId

        val credentialsPath =
            properties.auth.credentialsPath

        require(projectId.isNotBlank()) {
            "client-ledger.firestore.project-id must not be blank when mode is CLOUD"
        }

        require(credentialsPath.isNotBlank()) {
            "client-ledger.auth.credentials-path must not be blank when mode is CLOUD"
        }

        val credentials =
            if (credentialsPath.startsWith("classpath:")) {

                val resourcePath =
                    credentialsPath.removePrefix("classpath:")

                val resource =
                    ClassPathResource(resourcePath)

                require(resource.exists()) {
                    "Firebase service account file not found on classpath: $resourcePath"
                }

                resource.inputStream.use { inputStream ->
                    GoogleCredentials.fromStream(inputStream)
                }

            } else {

                FileInputStream(credentialsPath).use { inputStream ->
                    GoogleCredentials.fromStream(inputStream)
                }
            }

        println(
            """
            =================================================
             FIRESTORE CONFIGURATION
            =================================================
             Environment : CLOUD
             Project ID  : $projectId
             Firestore   : FIREBASE CLOUD
            =================================================
            """.trimIndent()
        )

        return FirestoreOptions.newBuilder()
            .setProjectId(projectId)
            .setCredentials(credentials)
            .build()
            .service
    }
}package com.clientledger.core.config

import com.google.api.gax.grpc.InstantiatingGrpcChannelProvider
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
@ConditionalOnProperty(
    prefix = "client-ledger",
    name = ["mode"],
    havingValue = "EMULATOR"
)
class FireStoreEmulatorConfig(
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun firestore(): Firestore {

        val projectId =
            properties.firestore.projectId

        val emulatorHost =
            properties.firestore.emulatorHost

        require(projectId.isNotBlank()) {
            "client-ledger.firestore.project-id must not be blank when mode is EMULATOR"
        }

        require(emulatorHost.isNotBlank()) {
            "client-ledger.firestore.emulator-host must not be blank when mode is EMULATOR"
        }

        println(
            """
            =================================================
             FIRESTORE CONFIGURATION
            =================================================
             Environment : EMULATOR
             Project ID  : $projectId
             Firestore   : FIRESTORE EMULATOR ($emulatorHost)
            =================================================
            """.trimIndent()
        )

        val channelProvider =
            InstantiatingGrpcChannelProvider.newBuilder()
                .setEndpoint(emulatorHost)
                .setChannelConfigurator { builder ->
                    builder.usePlaintext()
                }
                .build()

        return FirestoreOptions.newBuilder()
            .setProjectId(projectId)
            .setHost(emulatorHost)
            .setChannelProvider(channelProvider)
            .setCredentials(NoCredentials.getInstance())
            .build()
            .service
    }
}
application.yml
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
    restore-normal:
      max-attempts: 6
      retry-delay-seconds: 10

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
	
application-test.yml:
client-ledger:
  mode: EMULATOR
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

  firestore:
    project-id: "client-ledger-dashboard"
    emulator-host: "127.0.0.1:8080"	
	
	
package com.clientledger.core.repository.lock

import com.clientledger.core.domain.MaintenanceState
import com.clientledger.core.domain.OwnerOperationLock
import com.clientledger.core.enums.MaintenanceMode
import com.clientledger.core.repository.maintenance.MaintenanceRepository
import com.google.cloud.firestore.Firestore
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertFalse
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Assertions.assertNull
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import java.time.Instant
import java.util.UUID
import java.util.concurrent.CountDownLatch
import java.util.concurrent.Executors
import java.util.concurrent.TimeUnit

@SpringBootTest
class OwnerOperationLockRepositoryTest {

    @Autowired
    private lateinit var repository: OwnerOperationLockRepository

    @Autowired
    private lateinit var firestore: Firestore

    private val ownerIds = mutableSetOf<String>()


    @BeforeEach
    fun resetMaintenanceState() {
        MaintenanceRepository(firestore).save(
            MaintenanceState(
                mode = MaintenanceMode.NORMAL,
                updatedAt = Instant.now()
            )
        )
    }

    @AfterEach
    fun cleanup() {
        ownerIds.forEach { ownerId ->

            val operationLock =
                firestore
                    .collection("owners")
                    .document(ownerId)
                    .collection("operation_lock")

            operationLock
                .document("current")
                .delete()
                .get()

            operationLock
                .document("fence")
                .delete()
                .get()
        }

        ownerIds.clear()
    }

    @Test
    fun acquireShouldSucceedWhenLockDoesNotExist() {
        val ownerId = newOwnerId()
        val token = UUID.randomUUID().toString()

        val acquiredAt = Instant.now()
        val expiresAt = acquiredAt.plusSeconds(30)

        val acquired = repository.tryAcquire(
            ownerId = ownerId,
            lockToken = token,
            acquiredAt = acquiredAt,
            expiresAt = expiresAt
        )

        assertNotNull(acquired)
        assertEquals(ownerId, acquired!!.ownerId)
        assertEquals(token, acquired.lockToken)
        assertEquals(1L, acquired.fence)

        val lock = repository.find(ownerId)

        assertNotNull(lock)
        assertEquals(ownerId, lock!!.ownerId)
        assertEquals(token, lock.lockToken)
        assertEquals(1L, lock.fence)
    }

    @Test
    fun secondAcquireShouldFailWhileLockIsActive() {
        val ownerId = newOwnerId()

        val firstToken = UUID.randomUUID().toString()
        val secondToken = UUID.randomUUID().toString()

        val acquiredAt = Instant.now()
        val expiresAt = acquiredAt.plusSeconds(30)

        val firstAcquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = firstToken,
                acquiredAt = acquiredAt,
                expiresAt = expiresAt
            )

        assertNotNull(firstAcquired)
        assertEquals(1L, firstAcquired!!.fence)

        val secondAcquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = secondToken,
                acquiredAt = acquiredAt,
                expiresAt = expiresAt
            )

        assertNull(secondAcquired)

        val lock = repository.find(ownerId)

        assertNotNull(lock)
        assertEquals(firstToken, lock!!.lockToken)
        assertEquals(1L, lock.fence)
    }

    @Test
    fun releaseShouldKeepFenceGeneration() {
        val ownerId = newOwnerId()
        val token = UUID.randomUUID().toString()

        val acquiredAt = Instant.now()
        val expiresAt = acquiredAt.plusSeconds(30)

        val acquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = token,
                acquiredAt = acquiredAt,
                expiresAt = expiresAt
            )

        assertNotNull(acquired)
        assertEquals(1L, acquired!!.fence)

        assertTrue(
            repository.release(
                ownerId = ownerId,
                lockToken = token
            )
        )

        assertNull(repository.find(ownerId))

        val fenceDocument =
            firestore
                .collection("owners")
                .document(ownerId)
                .collection("operation_lock")
                .document("fence")

        val fenceSnapshot =
            fenceDocument
                .get()
                .get()

        assertTrue(fenceSnapshot.exists())
        assertEquals(
            1L,
            fenceSnapshot.getLong("generation")
        )
    }

    @Test
    fun nextAcquireAfterReleaseShouldIncrementFence() {
        val ownerId = newOwnerId()

        val firstToken = UUID.randomUUID().toString()
        val secondToken = UUID.randomUUID().toString()

        val firstAcquiredAt = Instant.now()
        val firstExpiresAt = firstAcquiredAt.plusSeconds(30)

        val firstAcquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = firstToken,
                acquiredAt = firstAcquiredAt,
                expiresAt = firstExpiresAt
            )

        assertNotNull(firstAcquired)
        assertEquals(1L, firstAcquired!!.fence)

        assertTrue(
            repository.release(
                ownerId = ownerId,
                lockToken = firstToken
            )
        )

        val secondAcquiredAt = Instant.now()
        val secondExpiresAt = secondAcquiredAt.plusSeconds(30)

        val secondAcquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = secondToken,
                acquiredAt = secondAcquiredAt,
                expiresAt = secondExpiresAt
            )

        assertNotNull(secondAcquired)
        assertEquals(secondToken, secondAcquired!!.lockToken)
        assertEquals(2L, secondAcquired.fence)

        val lock = repository.find(ownerId)

        assertNotNull(lock)
        assertEquals(secondToken, lock!!.lockToken)
        assertEquals(2L, lock.fence)
    }

    @Test
    fun expiredLockShouldBeRecoverableWithNextFence() {
        val ownerId = newOwnerId()

        val oldToken = UUID.randomUUID().toString()
        val newToken = UUID.randomUUID().toString()

        val oldAcquiredAt = Instant.now().minusSeconds(60)
        val oldExpiresAt = Instant.now().minusSeconds(30)

        val oldAcquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = oldToken,
                acquiredAt = oldAcquiredAt,
                expiresAt = oldExpiresAt
            )

        assertNotNull(oldAcquired)
        assertEquals(1L, oldAcquired!!.fence)

        val newAcquiredAt = Instant.now()
        val newExpiresAt = newAcquiredAt.plusSeconds(30)

        val newAcquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = newToken,
                acquiredAt = newAcquiredAt,
                expiresAt = newExpiresAt
            )

        assertNotNull(newAcquired)
        assertEquals(newToken, newAcquired!!.lockToken)
        assertEquals(2L, newAcquired.fence)

        val lock = repository.find(ownerId)

        assertNotNull(lock)
        assertEquals(newToken, lock!!.lockToken)
        assertEquals(2L, lock.fence)
    }

    @Test
    fun wrongTokenShouldNotRenewLock() {
        val ownerId = newOwnerId()

        val correctToken = UUID.randomUUID().toString()
        val wrongToken = UUID.randomUUID().toString()

        val acquiredAt = Instant.now()
        val originalExpiry = acquiredAt.plusSeconds(30)

        val acquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = correctToken,
                acquiredAt = acquiredAt,
                expiresAt = originalExpiry
            )

        assertNotNull(acquired)
        assertEquals(1L, acquired!!.fence)

        val newExpiry = acquiredAt.plusSeconds(120)

        assertFalse(
            repository.renew(
                ownerId = ownerId,
                lockToken = wrongToken,
                expiresAt = newExpiry
            )
        )

        val lock = repository.find(ownerId)

        assertNotNull(lock)
        assertEquals(
            originalExpiry.toEpochMilli(),
            lock!!.expiresAt.toEpochMilli()
        )
        assertEquals(1L, lock.fence)
    }

    @Test
    fun correctTokenShouldRenewLockWithoutChangingFence() {
        val ownerId = newOwnerId()
        val token = UUID.randomUUID().toString()

        val acquiredAt = Instant.now()
        val originalExpiry = acquiredAt.plusSeconds(30)

        val acquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = token,
                acquiredAt = acquiredAt,
                expiresAt = originalExpiry
            )

        assertNotNull(acquired)
        assertEquals(1L, acquired!!.fence)

        val newExpiry = acquiredAt.plusSeconds(120)

        assertTrue(
            repository.renew(
                ownerId = ownerId,
                lockToken = token,
                expiresAt = newExpiry
            )
        )

        val lock = repository.find(ownerId)

        assertNotNull(lock)
        assertEquals(
            newExpiry.toEpochMilli(),
            lock!!.expiresAt.toEpochMilli()
        )

        assertEquals(
            1L,
            lock.fence
        )
    }

    @Test
    fun wrongTokenShouldNotReleaseLock() {
        val ownerId = newOwnerId()

        val correctToken = UUID.randomUUID().toString()
        val wrongToken = UUID.randomUUID().toString()

        val acquiredAt = Instant.now()
        val expiresAt = acquiredAt.plusSeconds(30)

        val acquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = correctToken,
                acquiredAt = acquiredAt,
                expiresAt = expiresAt
            )

        assertNotNull(acquired)
        assertEquals(1L, acquired!!.fence)

        assertFalse(
            repository.release(
                ownerId = ownerId,
                lockToken = wrongToken
            )
        )

        val lock = repository.find(ownerId)

        assertNotNull(lock)
        assertEquals(1L, lock!!.fence)
    }

    @Test
    fun correctTokenShouldReleaseLock() {
        val ownerId = newOwnerId()
        val token = UUID.randomUUID().toString()

        val acquiredAt = Instant.now()
        val expiresAt = acquiredAt.plusSeconds(30)

        val acquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = token,
                acquiredAt = acquiredAt,
                expiresAt = expiresAt
            )

        assertNotNull(acquired)
        assertEquals(1L, acquired!!.fence)

        assertTrue(
            repository.release(
                ownerId = ownerId,
                lockToken = token
            )
        )

        assertNull(repository.find(ownerId))

        val fenceDocument =
            firestore
                .collection("owners")
                .document(ownerId)
                .collection("operation_lock")
                .document("fence")

        val fenceSnapshot =
            fenceDocument
                .get()
                .get()

        assertTrue(fenceSnapshot.exists())
        assertEquals(
            1L,
            fenceSnapshot.getLong("generation")
        )
    }

    @Test
    fun concurrentAcquireShouldAllowOnlyOneWinnerWithFirstFence() {
        val ownerId = newOwnerId()

        val threadCount = 2
        val startLatch = CountDownLatch(1)
        val executor = Executors.newFixedThreadPool(threadCount)

        try {
            val futures = (1..threadCount).map {
                executor.submit<OwnerOperationLock?> {
                    startLatch.await()

                    val token = UUID.randomUUID().toString()
                    val acquiredAt = Instant.now()

                    repository.tryAcquire(
                        ownerId = ownerId,
                        lockToken = token,
                        acquiredAt = acquiredAt,
                        expiresAt = acquiredAt.plusSeconds(30)
                    )
                }
            }

            startLatch.countDown()

            val results = futures.map {
                it.get(10, TimeUnit.SECONDS)
            }

            assertEquals(
                1,
                results.count { it != null }
            )

            assertEquals(
                threadCount - 1,
                results.count { it == null }
            )

            val winner = results.first { it != null }

            assertNotNull(winner)
            assertEquals(1L, winner!!.fence)

            val lock = repository.find(ownerId)

            assertNotNull(lock)
            assertEquals(1L, lock!!.fence)
        } finally {
            executor.shutdownNow()
        }
    }

    @Test
    fun differentOwnersShouldBeAbleToAcquireConcurrently() {
        val ownerOne = newOwnerId()
        val ownerTwo = newOwnerId()

        val startLatch = CountDownLatch(1)
        val executor = Executors.newFixedThreadPool(2)

        try {
            val futureOne = executor.submit<OwnerOperationLock?> {
                startLatch.await()

                val acquiredAt = Instant.now()

                repository.tryAcquire(
                    ownerId = ownerOne,
                    lockToken = UUID.randomUUID().toString(),
                    acquiredAt = acquiredAt,
                    expiresAt = acquiredAt.plusSeconds(30)
                )
            }

            val futureTwo = executor.submit<OwnerOperationLock?> {
                startLatch.await()

                val acquiredAt = Instant.now()

                repository.tryAcquire(
                    ownerId = ownerTwo,
                    lockToken = UUID.randomUUID().toString(),
                    acquiredAt = acquiredAt,
                    expiresAt = acquiredAt.plusSeconds(30)
                )
            }

            startLatch.countDown()

            val lockOne =
                futureOne.get(10, TimeUnit.SECONDS)

            val lockTwo =
                futureTwo.get(10, TimeUnit.SECONDS)

            assertNotNull(lockOne)
            assertNotNull(lockTwo)

            assertEquals(1L, lockOne!!.fence)
            assertEquals(1L, lockTwo!!.fence)
        } finally {
            executor.shutdownNow()
        }
    }

    @Test
    fun fenceShouldNeverDecreaseAcrossMultipleAcquisitions() {
        val ownerId = newOwnerId()

        repeat(5) { expectedFence ->

            val token = UUID.randomUUID().toString()
            val acquiredAt = Instant.now()
            val expiresAt = acquiredAt.plusSeconds(30)

            val acquired =
                repository.tryAcquire(
                    ownerId = ownerId,
                    lockToken = token,
                    acquiredAt = acquiredAt,
                    expiresAt = expiresAt
                )

            assertNotNull(acquired)
            assertEquals(
                (expectedFence + 1).toLong(),
                acquired!!.fence
            )

            val lock = repository.find(ownerId)

            assertNotNull(lock)
            assertEquals(
                (expectedFence + 1).toLong(),
                lock!!.fence
            )

            assertTrue(
                repository.release(
                    ownerId = ownerId,
                    lockToken = token
                )
            )
        }
    }

    @Test
    fun matchingFenceShouldBeValidInsideTransaction() {
        val ownerId = newOwnerId()
        val token = UUID.randomUUID().toString()

        val acquiredAt = Instant.now()
        val expiresAt = acquiredAt.plusSeconds(30)

        val acquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = token,
                acquiredAt = acquiredAt,
                expiresAt = expiresAt
            )

        assertNotNull(acquired)
        assertEquals(1L, acquired!!.fence)

        val result =
            firestore.runTransaction { transaction ->

                repository.isFenceValidInTransaction(
                    transaction = transaction,
                    ownerId = ownerId,
                    expectedFence = 1L
                )
            }.get()

        assertTrue(result)
    }

    @Test
    fun differentFenceShouldBeInvalidInsideTransaction() {
        val ownerId = newOwnerId()
        val token = UUID.randomUUID().toString()

        val acquiredAt = Instant.now()
        val expiresAt = acquiredAt.plusSeconds(30)

        val acquired =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = token,
                acquiredAt = acquiredAt,
                expiresAt = expiresAt
            )

        assertNotNull(acquired)
        assertEquals(1L, acquired!!.fence)

        val result =
            firestore.runTransaction { transaction ->

                repository.isFenceValidInTransaction(
                    transaction = transaction,
                    ownerId = ownerId,
                    expectedFence = 2L
                )
            }.get()

        assertFalse(result)
    }

    @Test
    fun lockShouldBeAcquiredWhenMaintenanceIsNormal() {

        val ownerId = newOwnerId()
        val token = UUID.randomUUID().toString()

        val maintenanceRepository =
            MaintenanceRepository(firestore)

        maintenanceRepository.save(
            MaintenanceState(
                mode = MaintenanceMode.NORMAL,
                updatedAt = Instant.now()
            )
        )

        val repository =
            OwnerOperationLockRepository(
                firestore = firestore,
                maintenanceRepository = maintenanceRepository
            )

        val acquiredAt = Instant.now()
        val expiresAt = acquiredAt.plusSeconds(30)

        val lock =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = token,
                acquiredAt = acquiredAt,
                expiresAt = expiresAt
            )

        assertNotNull(lock)
        assertEquals(1L, lock!!.fence)
    }


    @Test
    fun lockShouldNotBeAcquiredWhenMaintenanceIsWriteBlocked() {

        val ownerId = newOwnerId()
        val token = UUID.randomUUID().toString()

        val maintenanceRepository =
            MaintenanceRepository(firestore)

        maintenanceRepository.save(
            MaintenanceState(
                mode = MaintenanceMode.WRITE_BLOCKED,
                updatedAt = Instant.now()
            )
        )

        val repository =
            OwnerOperationLockRepository(
                firestore = firestore,
                maintenanceRepository = maintenanceRepository
            )

        val acquiredAt = Instant.now()
        val expiresAt = acquiredAt.plusSeconds(30)

        val lock =
            repository.tryAcquire(
                ownerId = ownerId,
                lockToken = token,
                acquiredAt = acquiredAt,
                expiresAt = expiresAt
            )

        assertNull(lock)
    }

    private fun newOwnerId(): String {
        val ownerId = "lock-test-${UUID.randomUUID()}"
        ownerIds += ownerId
        return ownerId
    }
}

package com.clientledger.core.repository.maintenance

import com.clientledger.core.domain.MaintenanceState
import com.clientledger.core.enums.MaintenanceMode
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Assertions.assertNull
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import java.time.Instant

class MaintenanceRepositoryTest {

    companion object {

        private lateinit var firestore: Firestore

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
        }

        @JvmStatic
        @AfterAll
        fun cleanup() {
            if (::firestore.isInitialized) {
                firestore.close()
            }
        }
    }

    @BeforeEach
    fun cleanMaintenanceDocument() {
        firestore
            .collection("system")
            .document("maintenance")
            .delete()
            .get()
    }

    @Test
    fun saveAndFindMaintenanceState() {

        val repository = MaintenanceRepository(firestore)

        val updatedAt = Instant.now()

        val state = MaintenanceState(
            mode = MaintenanceMode.WRITE_BLOCKED,
            updatedAt = updatedAt,
            reason = "Monthly rollover"
        )

        repository.save(state)

        val result = repository.find()

        assertNotNull(result)

        assertEquals(
            MaintenanceMode.WRITE_BLOCKED,
            result!!.mode
        )

        assertEquals(
            updatedAt.toEpochMilli(),
            result.updatedAt.toEpochMilli()
        )

        assertEquals(
            "Monthly rollover",
            result.reason
        )
    }

    @Test
    fun findReturnsNullWhenDocumentDoesNotExist() {

        val repository = MaintenanceRepository(firestore)

        val result = repository.find()

        assertNull(result)
    }
}package com.clientledger.core.repository.summary

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.ClientType
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles

@SpringBootTest
@ActiveProfiles("test")
class SummaryClientIndexBucketRepositoryTest {

    @Autowired
    private lateinit var properties: ClientLedgerProperties

    companion object {

        private lateinit var firestore: Firestore

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
        }

        @JvmStatic
        @AfterAll
        fun cleanup() {
            firestore.close()
        }
    }

    private fun repository(): SummaryClientIndexBucketRepository {
        return SummaryClientIndexBucketRepository(
            firestore = firestore,
            properties = properties
        )
    }

    @Test
    fun allocateFirstClientUsesBucket000() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        val bucketId = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE
        )

        assertEquals("bucket_000", bucketId)

        val bucket = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000"
        )

        assertNotNull(bucket)
        assertEquals(300, bucket!!["capacity"])
        assertEquals(1, bucket["size"])
    }

    @Test
    fun allocateClient301UsesBucket001() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        repeat(300) {
            repository.allocateBucket(
                ownerId = ownerId,
                yearMonth = "2026-09",
                status = ClientType.RECEIVABLE
            )
        }

        val bucketId = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE
        )

        assertEquals("bucket_001", bucketId)

        val bucket000 = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000"
        )

        val bucket001 = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_001"
        )

        assertNotNull(bucket000)
        assertNotNull(bucket001)

        assertEquals(300, bucket000!!["size"])
        assertEquals(1, bucket001!!["size"])
    }

    @Test
    fun differentStatusUsesSeparateBuckets() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        val receivableBucket = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE
        )

        val advanceBucket = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.ADVANCE
        )

        assertEquals("bucket_000", receivableBucket)
        assertEquals("bucket_000", advanceBucket)

        val receivable = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000"
        )

        val advance = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.ADVANCE,
            bucketId = "bucket_000"
        )

        assertNotNull(receivable)
        assertNotNull(advance)

        assertEquals(1, receivable!!["size"])
        assertEquals(1, advance!!["size"])
    }

    @Test
    fun differentOwnerUsesSeparateBuckets() {

        val repository = repository()

        val owner1 = "owner-1-${System.nanoTime()}"
        val owner2 = "owner-2-${System.nanoTime()}"

        val firstOwnerBucket = repository.allocateBucket(
            ownerId = owner1,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE
        )

        val secondOwnerBucket = repository.allocateBucket(
            ownerId = owner2,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE
        )

        assertEquals("bucket_000", firstOwnerBucket)
        assertEquals("bucket_000", secondOwnerBucket)

        val firstOwner = repository.find(
            ownerId = owner1,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000"
        )

        val secondOwner = repository.find(
            ownerId = owner2,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000"
        )

        assertNotNull(firstOwner)
        assertNotNull(secondOwner)

        assertEquals(1, firstOwner!!["size"])
        assertEquals(1, secondOwner!!["size"])
    }

    @Test
    fun differentMonthUsesSeparateBuckets() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        val septemberBucket = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE
        )

        val octoberBucket = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-10",
            status = ClientType.RECEIVABLE
        )

        assertEquals("bucket_000", septemberBucket)
        assertEquals("bucket_000", octoberBucket)

        val september = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000"
        )

        val october = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-10",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000"
        )

        assertNotNull(september)
        assertNotNull(october)

        assertEquals(1, september!!["size"])
        assertEquals(1, october!!["size"])
    }

    @Test
    fun findReturnsNullWhenBucketDoesNotExist() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        val bucket = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000"
        )

        assertEquals(null, bucket)
    }

    @Test
    fun bucketUsesFlatYearMonthStructure() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE
        )

        val snapshot = firestore
            .collection("owners")
            .document(ownerId)
            .collection("summary_client_index")
            .document("2026-09")
            .collection("status")
            .document("receivable")
            .collection("buckets")
            .document("bucket_000")
            .get()
            .get()

        assertTrue(snapshot.exists())
        assertEquals(300, snapshot.getLong("capacity"))
        assertEquals(1, snapshot.getLong("size"))
    }

    @Test
    fun planBucketAllocationInTransactionFindsAvailableBucket() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val bucketId = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = ClientType.RECEIVABLE
        )

        val plan = firestore.runTransaction { transaction ->

            repository.planBucketAllocationInTransaction(
                transaction = transaction,
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE
            )
        }.get()

        assertEquals(bucketId, plan.bucketId)
        assertEquals(1, plan.currentSize)
        assertEquals(false, plan.isNewBucket)
    }

    @Test
    fun applyBucketAllocationInTransactionIncreasesBucketSize() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val bucketId = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = ClientType.RECEIVABLE
        )

        firestore.runTransaction { transaction ->

            val plan = repository.planBucketAllocationInTransaction(
                transaction = transaction,
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE
            )

            repository.applyBucketAllocationInTransaction(
                transaction = transaction,
                plan = plan
            )

            null
        }.get()

        val bucket = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = ClientType.RECEIVABLE,
            bucketId = bucketId
        )

        assertNotNull(bucket)
        assertEquals(2, bucket!!["size"])
    }

    @Test
    fun planBucketAllocationInTransactionPlansNewBucketWhenAllAreFull() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        repeat(300) {
            repository.allocateBucket(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE
            )
        }

        val plan = firestore.runTransaction { transaction ->

            repository.planBucketAllocationInTransaction(
                transaction = transaction,
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE
            )
        }.get()

        assertEquals("bucket_001", plan.bucketId)
        assertEquals(0, plan.currentSize)
        assertEquals(true, plan.isNewBucket)
    }
}package com.clientledger.core.transaction

import com.clientledger.core.operation.BusinessOperationContext
import com.google.cloud.firestore.Firestore
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import java.time.Clock
import java.time.Duration
import java.time.Instant
import java.time.ZoneOffset
import java.util.UUID
import kotlin.test.assertEquals
import kotlin.test.assertFailsWith

@SpringBootTest
class FirestoreTransactionExecutorTest {

    @Autowired
    private lateinit var executor: FirestoreTransactionExecutor

    @Autowired
    private lateinit var firestore: Firestore

    private val collectionName =
        "firestore_transaction_executor_test"

    private val createdDocuments =
        mutableListOf<String>()

    @AfterEach
    fun cleanup() {

        createdDocuments.forEach { documentId ->

            firestore
                .collection(collectionName)
                .document(documentId)
                .delete()
                .get()
        }

        createdDocuments.clear()
    }

    @Test
    fun existingExecuteShouldCommitTransaction() {

        val documentId =
            UUID.randomUUID().toString()

        createdDocuments += documentId

        executor.execute { transaction ->

            val document =
                firestore
                    .collection(collectionName)
                    .document(documentId)

            transaction.set(
                document,
                mapOf("value" to "committed")
            )
        }

        val snapshot =
            firestore
                .collection(collectionName)
                .document(documentId)
                .get()
                .get()

        assertEquals(
            "committed",
            snapshot.getString("value")
        )
    }

    @Test
    fun contextExecuteShouldCommitWhenWithinDeadline() {

        val documentId =
            UUID.randomUUID().toString()

        createdDocuments += documentId

        val context =
            BusinessOperationContext(
                clock = Clock.systemUTC(),
                timeout = Duration.ofSeconds(30)
            )

        executor.execute(context) { transaction, operationContext ->

            operationContext.checkDeadline()

            val document =
                firestore
                    .collection(collectionName)
                    .document(documentId)

            transaction.set(
                document,
                mapOf("value" to "committed")
            )
        }

        val snapshot =
            firestore
                .collection(collectionName)
                .document(documentId)
                .get()
                .get()

        assertEquals(
            "committed",
            snapshot.getString("value")
        )
    }

    @Test
    fun expiredContextShouldNotStartTransaction() {

        val documentId =
            UUID.randomUUID().toString()

        val expiredClock =
            Clock.fixed(
                Instant.parse("2026-09-29T10:00:00Z"),
                ZoneOffset.UTC
            )

        val context =
            BusinessOperationContext(
                clock = expiredClock,
                timeout = Duration.ZERO
            )

        assertFailsWith<RuntimeException> {

            executor.execute(context) { transaction, _ ->

                createdDocuments += documentId

                val document =
                    firestore
                        .collection(collectionName)
                        .document(documentId)

                transaction.set(
                    document,
                    mapOf("value" to "must-not-commit")
                )
            }
        }

        val snapshot =
            firestore
                .collection(collectionName)
                .document(documentId)
                .get()
                .get()

        assertEquals(
            false,
            snapshot.exists()
        )
    }
}	
