package com.clientledger.core.repository.summary

import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.SummaryClientIndex
import com.google.cloud.NoCredentials

import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals
import kotlin.test.assertFalse
import kotlin.test.assertNotNull
import kotlin.test.assertNull
import kotlin.test.assertTrue

class SummaryClientIndexRepositoryTest {

    companion object {

        private lateinit var firestore: Firestore

        @JvmStatic
        @BeforeAll
        fun setUpClass() {
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
        fun tearDownClass() {
            firestore.close()
        }
    }

    @BeforeEach
    fun setUp() {
        firestore
            .collection("owners")
            .document("test-owner")
            .collection("summary_client_index")
            .document("2026-09")
            .collection("buckets")
            .get()
            .get()
            .documents
            .forEach { bucket ->
                bucket.reference.delete().get()
            }
    }

    private fun repository(): SummaryClientIndexRepository {
        return SummaryClientIndexRepository(
            firestore = firestore
        )
    }

    private fun bucketReference(
        ownerId: String,
        yearMonth: String,
        bucketId: String
    ) =
        firestore
            .collection("owners")
            .document(ownerId)
            .collection("summary_client_index")
            .document(yearMonth)
            .collection("buckets")
            .document(bucketId)

    private fun clientReference(
        ownerId: String,
        yearMonth: String,
        bucketId: String,
        clientId: String
    ) =
        bucketReference(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = bucketId
        )
            .collection("clients")
            .document(clientId)

    private fun insertBucket(
        ownerId: String = "test-owner",
        yearMonth: String = "2026-09",
        bucketId: String = "bucket_000",
        capacity: Long = 300L,
        size: Long = 0L
    ) {
        bucketReference(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = bucketId
        )
            .set(
                mapOf(
                    "capacity" to capacity,
                    "size" to size
                )
            )
            .get()
    }

    private fun insertClient(
        ownerId: String = "test-owner",
        yearMonth: String = "2026-09",
        bucketId: String = "bucket_000",
        clientId: String,
        amount: Long,
        status: ClientType = ClientType.RECEIVABLE
    ) {
        clientReference(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = bucketId,
            clientId = clientId
        )
            .set(
                mapOf(
                    "clientId" to clientId,
                    "amount" to amount,
                    "status" to status.name
                )
            )
            .get()
    }

    private fun insertClients(
        count: Int,
        ownerId: String = "test-owner",
        yearMonth: String = "2026-09",
        bucketId: String = "bucket_000"
    ) {
        insertBucket(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = bucketId,
            capacity = 300L,
            size = count.toLong()
        )

        repeat(count) { index ->
            insertClient(
                ownerId = ownerId,
                yearMonth = yearMonth,
                bucketId = bucketId,
                clientId = "client-${index.toString().padStart(3, '0')}",
                amount = (index + 1) * 100L,
                status = ClientType.RECEIVABLE
            )
        }
    }

    @Test
    fun insertAndFindReceivableClient() {

        val repository = repository()

        insertBucket(
            bucketId = "bucket_000"
        )

        val index = SummaryClientIndex(
            clientId = "client-001",
            amount = 5000L,
            status = ClientType.RECEIVABLE
        )

        repository.insert(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            bucketId = "bucket_000",
            index = index
        )

        val result = repository.find(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            bucketId = "bucket_000",
            clientId = "client-001"
        )

        assertNotNull(result)
        assertEquals("client-001", result.clientId)
        assertEquals(5000L, result.amount)
        assertEquals(ClientType.RECEIVABLE, result.status)
    }

    @Test
    fun insertAndFindAdvanceClient() {

        val repository = repository()

        insertBucket(
            bucketId = "bucket_000"
        )

        val index = SummaryClientIndex(
            clientId = "client-001",
            amount = -3000L,
            status = ClientType.ADVANCE
        )

        repository.insert(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            bucketId = "bucket_000",
            index = index
        )

        val result = repository.find(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            bucketId = "bucket_000",
            clientId = "client-001"
        )

        assertNotNull(result)
        assertEquals("client-001", result.clientId)
        assertEquals(-3000L, result.amount)
        assertEquals(ClientType.ADVANCE, result.status)
    }

    @Test
    fun differentBucketsKeepClientsSeparate() {

        val repository = repository()

        insertBucket(
            bucketId = "bucket_000"
        )

        insertBucket(
            bucketId = "bucket_001"
        )

        repository.insert(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            bucketId = "bucket_000",
            index = SummaryClientIndex(
                clientId = "client-001",
                amount = 1000L,
                status = ClientType.RECEIVABLE
            )
        )

        repository.insert(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            bucketId = "bucket_001",
            index = SummaryClientIndex(
                clientId = "client-002",
                amount = 2000L,
                status = ClientType.RECEIVABLE
            )
        )

        val first = repository.find(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            bucketId = "bucket_000",
            clientId = "client-001"
        )

        val second = repository.find(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            bucketId = "bucket_001",
            clientId = "client-002"
        )

        assertNotNull(first)
        assertNotNull(second)

        assertEquals("client-001", first.clientId)
        assertEquals("client-002", second.clientId)
    }

    @Test
    fun findReturnsNullWhenClientDoesNotExist() {

        val repository = repository()

        insertBucket(
            bucketId = "bucket_000"
        )

        val result = repository.find(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            bucketId = "bucket_000",
            clientId = "does-not-exist"
        )

        assertNull(result)
    }

    @Test
    fun insertInTransactionAndFind() {

        val repository = repository()

        insertBucket(
            bucketId = "bucket_000"
        )

        firestore.runTransaction { transaction ->

            repository.insertInTransaction(
                transaction = transaction,
                ownerId = "test-owner",
                yearMonth = "2026-09",
                bucketId = "bucket_000",
                index = SummaryClientIndex(
                    clientId = "client-001",
                    amount = 7000L,
                    status = ClientType.RECEIVABLE
                )
            )

            null
        }.get()

        val result = repository.find(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            bucketId = "bucket_000",
            clientId = "client-001"
        )

        assertNotNull(result)
        assertEquals("client-001", result.clientId)
        assertEquals(7000L, result.amount)
        assertEquals(ClientType.RECEIVABLE, result.status)
    }

    @Test
    fun findPageReturnsEmptyWhenNoClientsExist() {

        val repository = repository()

        val result = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 10,
            cursor = null
        )

        assertTrue(result.content.isEmpty())
        assertFalse(result.hasNext)
        assertNull(result.nextCursor)
    }

    @Test
    fun findPageReturnsSingleClient() {

        val repository = repository()

        insertClients(1)

        val result = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 10,
            cursor = null
        )

        assertEquals(1, result.content.size)
        assertEquals("client-000", result.content[0].clientId)
        assertFalse(result.hasNext)
        assertNull(result.nextCursor)
    }

    @Test
    fun findPageReturnsExactly50Clients() {

        val repository = repository()

        insertClients(50)

        val result = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 50,
            cursor = null
        )

        assertEquals(50, result.content.size)
        assertEquals("client-000", result.content.first().clientId)
        assertEquals("client-049", result.content.last().clientId)
        assertFalse(result.hasNext)
        assertNull(result.nextCursor)
    }

    @Test
    fun findPageReturns51ClientsAcrossTwoPages() {

        val repository = repository()

        insertClients(51)

        val firstPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 50,
            cursor = null
        )

        assertEquals(50, firstPage.content.size)
        assertTrue(firstPage.hasNext)
        assertNotNull(firstPage.nextCursor)

        val secondPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 50,
            cursor = firstPage.nextCursor
        )

        assertEquals(1, secondPage.content.size)
        assertEquals("client-050", secondPage.content[0].clientId)
        assertFalse(secondPage.hasNext)
        assertNull(secondPage.nextCursor)
    }

    @Test
    fun findPageReturns100ClientsAcrossTwoPages() {

        val repository = repository()

        insertClients(100)

        val firstPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 50,
            cursor = null
        )

        assertEquals(50, firstPage.content.size)
        assertTrue(firstPage.hasNext)

        val secondPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 50,
            cursor = firstPage.nextCursor
        )

        assertEquals(50, secondPage.content.size)
        assertFalse(secondPage.hasNext)
        assertNull(secondPage.nextCursor)

        assertEquals("client-050", secondPage.content.first().clientId)
        assertEquals("client-099", secondPage.content.last().clientId)
    }

    @Test
    fun findPageReturns101ClientsAcrossThreePages() {

        val repository = repository()

        insertClients(101)

        val firstPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 50,
            cursor = null
        )

        assertEquals(50, firstPage.content.size)
        assertTrue(firstPage.hasNext)

        val secondPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 50,
            cursor = firstPage.nextCursor
        )

        assertEquals(50, secondPage.content.size)
        assertTrue(secondPage.hasNext)

        val thirdPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 50,
            cursor = secondPage.nextCursor
        )

        assertEquals(1, thirdPage.content.size)
        assertEquals("client-100", thirdPage.content[0].clientId)
        assertFalse(thirdPage.hasNext)
        assertNull(thirdPage.nextCursor)
    }

    @Test
    fun findPageDoesNotSkipOrDuplicateClients() {

        val repository = repository()

        insertClients(101)

        val firstPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 40,
            cursor = null
        )

        val secondPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 40,
            cursor = firstPage.nextCursor
        )

        val thirdPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 40,
            cursor = secondPage.nextCursor
        )

        val allIds =
            firstPage.content.map { it.clientId } +
                    secondPage.content.map { it.clientId } +
                    thirdPage.content.map { it.clientId }

        assertEquals(101, allIds.size)
        assertEquals(101, allIds.toSet().size)

        assertEquals(
            (0 until 101).map {
                "client-${it.toString().padStart(3, '0')}"
            },
            allIds
        )
    }

    @Test
    fun findPageHandlesMultipleBuckets() {

        val repository = repository()

        insertClients(
            count = 3,
            bucketId = "bucket_000"
        )

        insertClients(
            count = 3,
            bucketId = "bucket_001"
        )

        val firstPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 2,
            cursor = null
        )

        assertEquals(2, firstPage.content.size)
        assertEquals("client-000", firstPage.content[0].clientId)
        assertEquals("client-001", firstPage.content[1].clientId)
        assertTrue(firstPage.hasNext)

        val secondPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 2,
            cursor = firstPage.nextCursor
        )

        assertEquals(2, secondPage.content.size)
        assertEquals("client-002", secondPage.content[0].clientId)
        assertEquals("client-000", secondPage.content[1].clientId)
        assertTrue(secondPage.hasNext)

        val thirdPage = repository.findPage(
            ownerId = "test-owner",
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            size = 2,
            cursor = secondPage.nextCursor
        )

        assertEquals(2, thirdPage.content.size)
        assertEquals("client-001", thirdPage.content[0].clientId)
        assertEquals("client-002", thirdPage.content[1].clientId)
        assertFalse(thirdPage.hasNext)
        assertNull(thirdPage.nextCursor)
    }

    @Test
    fun findPageRejectsInvalidCursor() {

        val repository = repository()

        insertClients(1)

        val exception = runCatching {
            repository.findPage(
                ownerId = "test-owner",
                yearMonth = "2026-09",
                status = ClientType.RECEIVABLE,
                size = 10,
                cursor = "invalid-cursor"
            )
        }.exceptionOrNull()

        assertNotNull(exception)
        assertTrue(
            exception is IllegalArgumentException ||
                    exception.cause is IllegalArgumentException
        )
    }
}

