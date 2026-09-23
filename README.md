PS D:\New folder\client-ledger-codespace-main> ./gradlew :app:test
Reusing configuration cache.
Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :app:test

ClientMonthlyHistoryRepositoryTest > savesDifferentClientsConcurrentlyInSameBucket() FAILED
    kotlin.UninitializedPropertyAccessException at ClientMonthlyHistoryRepositoryTest.kt:32

ClientMonthlyHistoryRepositoryTest > savesAndFindsClientMonthlyHistory() FAILED
    kotlin.UninitializedPropertyAccessException at ClientMonthlyHistoryRepositoryTest.kt:32

ClientMonthlyHistoryRepositoryTest > savesMultipleClientsInSameBucket() FAILED
    kotlin.UninitializedPropertyAccessException at ClientMonthlyHistoryRepositoryTest.kt:32

ClientMonthlyHistoryRepositoryTest > rejectsNewClientWhenBucketIsFull() FAILED
    kotlin.UninitializedPropertyAccessException at ClientMonthlyHistoryRepositoryTest.kt:32

ClientMonthlyHistoryRepositoryTest > updatingExistingClientDoesNotIncreaseBucketSize() FAILED
    kotlin.UninitializedPropertyAccessException at ClientMonthlyHistoryRepositoryTest.kt:32

ClientMonthlyHistoryRepositoryTest > bucketSizeIncreasesForNewClients() FAILED
    kotlin.UninitializedPropertyAccessException at ClientMonthlyHistoryRepositoryTest.kt:32

OwnerMonthInitializationServiceTest > createCurrentMonthFromPreviousMonth() FAILED
    org.opentest4j.AssertionFailedError at OwnerMonthInitializationServiceTest.kt:173

60 tests completed, 7 failed

> Task :app:test FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:test'.
> There were failing tests. See the report at: file:///D:/New%20folder/client-ledger-codespace-main/app/build/reports/tests/test/index.html

* Try:
> Run with --scan to get full insights.

BUILD FAILED in 33s
8 actionable tasks: 2 executed, 6 up-to-date
Configuration cache entry reused.
PS D:\New folder\client-ledger-codespace-main> 

code: package com.clientledger.core.repository.history

import com.clientledger.core.domain.ClientMonthlyHistory
import com.clientledger.core.domain.ClientType
import com.clientledger.core.service.CurrentOwnerProvider
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import com.google.cloud.NoCredentials
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertThrows
import java.util.concurrent.CountDownLatch
import java.util.concurrent.Executors
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test

class ClientMonthlyHistoryRepositoryTest {

    private lateinit var firestore: Firestore
    private lateinit var repository: ClientMonthlyHistoryRepository
    private lateinit var currentOwnerProvider: CurrentOwnerProvider

    @BeforeEach
    fun setUp() {
        firestore = FirestoreOptions.newBuilder()
            .setProjectId("client-ledger-codespace")
            .setHost("127.0.0.1:8080")
            .setEmulatorHost("127.0.0.1:8080")
            .setCredentials(NoCredentials.getInstance())
            .build()
            .service

        repository = ClientMonthlyHistoryRepository(firestore, currentOwnerProvider)

        val monthReference = firestore
            .collection("master_history_client_data")
            .document("2026")
            .collection("09")

        val bucketDocuments = monthReference
            .get()
            .get()
            .documents

        bucketDocuments.forEach { bucketDocument ->

            val clientHistoryDocuments = bucketDocument
                .reference
                .collection("client_histories")
                .get()
                .get()
                .documents

            clientHistoryDocuments.forEach {
                it.reference.delete().get()
            }

            bucketDocument.reference.delete().get()
        }
    }

    @Test
    fun savesAndFindsClientMonthlyHistory() {

        val history = ClientMonthlyHistory(
            clientId = "CLI-TEST-001",
            ownerId = "OWNER001",
            yearMonth = "2026-09",
            openingBalance = 5000,
            closingBalance = 5000,
            receivable = 5000,
            advance = 0,
            status = ClientType.RECEIVABLE
        )

        repository.save(
            history = history,
            bucketId = "bucket_test"
        )

        val result = repository.find(
            clientId = "CLI-TEST-001",
            yearMonth = "2026-09",
            bucketId = "bucket_test"
        )

        requireNotNull(result)

        assertEquals("CLI-TEST-001", result.clientId)
        assertEquals("OWNER001", result.ownerId)
        assertEquals("2026-09", result.yearMonth)
        assertEquals(5000, result.openingBalance)
        assertEquals(5000, result.closingBalance)
        assertEquals(5000, result.receivable)
        assertEquals(ClientType.RECEIVABLE, result.status)
    }

    @Test
    fun savesMultipleClientsInSameBucket() {

        val firstClient = ClientMonthlyHistory(
            clientId = "CLI-001",
            ownerId = "OWNER001",
            yearMonth = "2026-09",
            openingBalance = 5000,
            closingBalance = 5000,
            receivable = 5000,
            status = ClientType.RECEIVABLE
        )

        val secondClient = ClientMonthlyHistory(
            clientId = "CLI-002",
            ownerId = "OWNER001",
            yearMonth = "2026-09",
            openingBalance = 2000,
            closingBalance = 2000,
            receivable = 2000,
            status = ClientType.RECEIVABLE
        )

        repository.save(firstClient, "bucket_000")
        repository.save(secondClient, "bucket_000")

        val firstResult =
            repository.find("CLI-001", "2026-09", "bucket_000")

        val secondResult =
            repository.find("CLI-002", "2026-09", "bucket_000")

        assertEquals(5000, firstResult?.closingBalance)
        assertEquals(2000, secondResult?.closingBalance)
    }

    @Test
    fun bucketSizeIncreasesForNewClients() {

        val firstClient = ClientMonthlyHistory(
            clientId = "CLI-001",
            ownerId = "OWNER001",
            yearMonth = "2026-09"
        )

        val secondClient = ClientMonthlyHistory(
            clientId = "CLI-002",
            ownerId = "OWNER001",
            yearMonth = "2026-09"
        )

        repository.save(firstClient, "bucket_000")

        assertEquals(
            1,
            repository.getBucketSize(
                "2026-09",
                "bucket_000"
            )
        )

        repository.save(secondClient, "bucket_000")

        assertEquals(
            2,
            repository.getBucketSize(
                "2026-09",
                "bucket_000"
            )
        )
    }

    @Test
    fun updatingExistingClientDoesNotIncreaseBucketSize() {

        val history = ClientMonthlyHistory(
            clientId = "CLI-001",
            ownerId = "OWNER001",
            yearMonth = "2026-09",
            closingBalance = 5000
        )

        repository.save(history, "bucket_000")

        assertEquals(
            1,
            repository.getBucketSize(
                "2026-09",
                "bucket_000"
            )
        )

        val updatedHistory = history.copy(
            closingBalance = 8000,
            receivable = 8000
        )

        repository.save(updatedHistory, "bucket_000")

        assertEquals(
            1,
            repository.getBucketSize(
                "2026-09",
                "bucket_000"
            )
        )

        val result = repository.find(
            "CLI-001",
            "2026-09",
            "bucket_000"
        )

        assertEquals(8000, result?.closingBalance)
        assertEquals(8000, result?.receivable)
    }

    @Test
    fun rejectsNewClientWhenBucketIsFull() {

        val bucketId = "bucket_full"

        repeat(50) { index ->

            val history = ClientMonthlyHistory(
                clientId = "CLI-${index + 1}",
                ownerId = "OWNER001",
                yearMonth = "2026-09"
            )

            repository.save(
                history,
                bucketId
            )
        }

        assertEquals(
            50,
            repository.getBucketSize(
                "2026-09",
                bucketId
            )
        )

        val newClient = ClientMonthlyHistory(
            clientId = "CLI-51",
            ownerId = "OWNER001",
            yearMonth = "2026-09"
        )

        val exception = assertThrows(
            java.util.concurrent.ExecutionException::class.java
        ) {
            repository.save(
                newClient,
                bucketId
            )
        }

        assertEquals(
            IllegalStateException::class.java,
            exception.cause?.javaClass
        )
    }

    @Test
    fun savesDifferentClientsConcurrentlyInSameBucket() {

        val executor = Executors.newFixedThreadPool(2)
        val startLatch = CountDownLatch(1)

        val firstClient = ClientMonthlyHistory(
            clientId = "CLI-CONCURRENT-001",
            ownerId = "OWNER001",
            yearMonth = "2026-09",
            closingBalance = 5000,
            receivable = 5000,
            status = ClientType.RECEIVABLE
        )

        val secondClient = ClientMonthlyHistory(
            clientId = "CLI-CONCURRENT-002",
            ownerId = "OWNER001",
            yearMonth = "2026-09",
            closingBalance = 7000,
            receivable = 7000,
            status = ClientType.RECEIVABLE
        )

        val firstTask = executor.submit {
            startLatch.await()
            repository.save(firstClient, "bucket_concurrent")
        }

        val secondTask = executor.submit {
            startLatch.await()
            repository.save(secondClient, "bucket_concurrent")
        }

        startLatch.countDown()

        firstTask.get()
        secondTask.get()

        executor.shutdown()

        assertEquals(
            2,
            repository.getBucketSize(
                "2026-09",
                "bucket_concurrent"
            )
        )

        val firstResult = repository.find(
            "CLI-CONCURRENT-001",
            "2026-09",
            "bucket_concurrent"
        )

        val secondResult = repository.find(
            "CLI-CONCURRENT-002",
            "2026-09",
            "bucket_concurrent"
        )

        assertEquals(5000, firstResult?.closingBalance)
        assertEquals(7000, secondResult?.closingBalance)
    }
}
