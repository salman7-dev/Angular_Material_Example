package com.clientledger.core.repository.summary

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
}
