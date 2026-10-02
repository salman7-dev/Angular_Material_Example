package com.clientledger.core.repository.summary

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.SummaryClientIndex
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertThrows
import org.junit.jupiter.api.BeforeAll

class SummaryClientIndexRepositoryTest {

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

    private fun repository(): SummaryClientIndexRepository {
        return SummaryClientIndexRepository(
            firestore = firestore
        )
    }

    @Test
    fun insertAndFindReceivableClient() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val index = SummaryClientIndex(
            clientId = "client-${System.nanoTime()}",
            amount = 10000,
            status = ClientType.RECEIVABLE
        )

        repository.insert(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_000",
            index = index
        )

        val result = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_000",
            clientId = index.clientId
        )

        assertNotNull(result)
        assertEquals(index.clientId, result!!.clientId)
        assertEquals(10000, result.amount)
        assertEquals(ClientType.RECEIVABLE, result.status)
    }

    @Test
    fun insertAndFindAdvanceClient() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val index = SummaryClientIndex(
            clientId = "client-${System.nanoTime()}",
            amount = 5000,
            status = ClientType.ADVANCE
        )

        repository.insert(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_000",
            index = index
        )

        val result = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_000",
            clientId = index.clientId
        )

        assertNotNull(result)
        assertEquals(index.clientId, result!!.clientId)
        assertEquals(5000, result.amount)
        assertEquals(ClientType.ADVANCE, result.status)
    }

    @Test
    fun differentBucketsKeepClientsSeparate() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val firstClient = SummaryClientIndex(
            clientId = "client-1-${System.nanoTime()}",
            amount = 1000,
            status = ClientType.RECEIVABLE
        )

        val secondClient = SummaryClientIndex(
            clientId = "client-2-${System.nanoTime()}",
            amount = 2000,
            status = ClientType.RECEIVABLE
        )

        repository.insert(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_000",
            index = firstClient
        )

        repository.insert(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_001",
            index = secondClient
        )

        val firstResult = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_000",
            clientId = firstClient.clientId
        )

        val secondResult = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_001",
            clientId = secondClient.clientId
        )

        assertNotNull(firstResult)
        assertNotNull(secondResult)

        assertEquals(firstClient.clientId, firstResult!!.clientId)
        assertEquals(secondClient.clientId, secondResult!!.clientId)
    }

    @Test
    fun findReturnsNullWhenClientDoesNotExist() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        val result = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            bucketId = "bucket_000",
            clientId = "missing-client"
        )

        assertEquals(null, result)
    }

    @Test
    fun insertInTransactionAndFind() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val index = SummaryClientIndex(
            clientId = "client-${System.nanoTime()}",
            amount = 7500,
            status = ClientType.RECEIVABLE
        )

        firestore.runTransaction { transaction ->

            repository.insertInTransaction(
                transaction = transaction,
                ownerId = ownerId,
                yearMonth = yearMonth,
                bucketId = "bucket_000",
                index = index
            )

            null
        }.get()

        val result = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = "bucket_000",
            clientId = index.clientId
        )

        assertNotNull(result)
        assertEquals(index.clientId, result!!.clientId)
        assertEquals(7500, result.amount)
        assertEquals(ClientType.RECEIVABLE, result.status)
    }

    @Test
    fun findPageReturnsEmptyWhenNoClientsExist() {

        val repository = repository()

        val result =
            repository.findPage(
                ownerId = "owner-${System.nanoTime()}",
                yearMonth = "2026-09",
                size = 50,
                cursor = null
            )

        assertEquals(0, result.content.size)
        assertEquals(false, result.hasNext)
        assertEquals(null, result.nextCursor)
    }

    @Test
    fun findPageReturnsSingleClient() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 1
        )

        val result =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = null
            )

        assertEquals(1, result.content.size)
        assertEquals("client-000", result.content.first().clientId)
        assertEquals(false, result.hasNext)
        assertEquals(null, result.nextCursor)
    }

    @Test
    fun findPageReturnsExactly50Clients() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 50
        )

        val result =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = null
            )

        assertEquals(50, result.content.size)
        assertEquals("client-000", result.content.first().clientId)
        assertEquals("client-049", result.content.last().clientId)
        assertEquals(false, result.hasNext)
        assertEquals(null, result.nextCursor)
    }

    @Test
    fun findPageReturns51ClientsAcrossTwoPages() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 51
        )

        val firstPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = null
            )

        assertEquals(50, firstPage.content.size)
        assertEquals("client-000", firstPage.content.first().clientId)
        assertEquals("client-049", firstPage.content.last().clientId)
        assertEquals(true, firstPage.hasNext)
        assertNotNull(firstPage.nextCursor)

        val secondPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = firstPage.nextCursor
            )

        assertEquals(1, secondPage.content.size)
        assertEquals("client-050", secondPage.content.first().clientId)
        assertEquals(false, secondPage.hasNext)
        assertEquals(null, secondPage.nextCursor)
    }

    @Test
    fun findPageReturns100ClientsAcrossTwoPages() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 100
        )

        val firstPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = null
            )

        val secondPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = firstPage.nextCursor
            )

        assertEquals(50, firstPage.content.size)
        assertEquals("client-049", firstPage.content.last().clientId)
        assertEquals(true, firstPage.hasNext)

        assertEquals(50, secondPage.content.size)
        assertEquals("client-050", secondPage.content.first().clientId)
        assertEquals("client-099", secondPage.content.last().clientId)
        assertEquals(false, secondPage.hasNext)
        assertEquals(null, secondPage.nextCursor)
    }

    @Test
    fun findPageReturns101ClientsAcrossThreePages() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 101
        )

        val firstPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = null
            )

        val secondPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = firstPage.nextCursor
            )

        val thirdPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = secondPage.nextCursor
            )

        assertEquals(50, firstPage.content.size)
        assertEquals("client-000", firstPage.content.first().clientId)
        assertEquals("client-049", firstPage.content.last().clientId)
        assertEquals(true, firstPage.hasNext)

        assertEquals(50, secondPage.content.size)
        assertEquals("client-050", secondPage.content.first().clientId)
        assertEquals("client-099", secondPage.content.last().clientId)
        assertEquals(true, secondPage.hasNext)

        assertEquals(1, thirdPage.content.size)
        assertEquals("client-100", thirdPage.content.first().clientId)
        assertEquals(false, thirdPage.hasNext)
        assertEquals(null, thirdPage.nextCursor)
    }

    @Test
    fun findPageDoesNotSkipOrDuplicateClients() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 101
        )

        val firstPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = null
            )

        val secondPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = firstPage.nextCursor
            )

        val thirdPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 50,
                cursor = secondPage.nextCursor
            )

        val clientIds =
            firstPage.content.map { it.clientId } +
                    secondPage.content.map { it.clientId } +
                    thirdPage.content.map { it.clientId }

        assertEquals(101, clientIds.size)
        assertEquals(101, clientIds.distinct().size)

        assertEquals(
            (0..100).map { "client-%03d".format(it) },
            clientIds
        )
    }

    @Test
    fun findPageHandlesMultipleBuckets() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        insertClients(
            repository = repository,
            ownerId = ownerId,
            yearMonth = yearMonth,
            count = 301
        )

        val firstPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 100,
                cursor = null
            )

        val secondPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 100,
                cursor = firstPage.nextCursor
            )

        val thirdPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 100,
                cursor = secondPage.nextCursor
            )

        val fourthPage =
            repository.findPage(
                ownerId = ownerId,
                yearMonth = yearMonth,
                size = 100,
                cursor = thirdPage.nextCursor
            )

        val clientIds =
            firstPage.content.map { it.clientId } +
                    secondPage.content.map { it.clientId } +
                    thirdPage.content.map { it.clientId } +
                    fourthPage.content.map { it.clientId }

        assertEquals(100, firstPage.content.size)
        assertEquals(100, secondPage.content.size)
        assertEquals(100, thirdPage.content.size)
        assertEquals(1, fourthPage.content.size)

        assertEquals(301, clientIds.size)
        assertEquals(301, clientIds.distinct().size)

        assertEquals("client-000", clientIds.first())
        assertEquals("client-300", clientIds.last())
    }

    @Test
    fun findPageRejectsInvalidCursor() {

        val repository = repository()

        val exception =
            assertThrows<IllegalArgumentException> {

                repository.findPage(
                    ownerId = "owner-${System.nanoTime()}",
                    yearMonth = "2026-09",
                    size = 50,
                    cursor = "invalid-cursor"
                )
            }

        assertEquals(
            "Invalid cursor",
            exception.message
        )
    }

    private fun insertClients(
        repository: SummaryClientIndexRepository,
        ownerId: String,
        yearMonth: String,
        count: Int
    ) {

        val bucketIds =
            if (count <= 300) {
                listOf("bucket_000")
            } else {
                listOf("bucket_000", "bucket_001")
            }

        bucketIds.forEach { bucketId ->

            firestore
                .collection("owners")
                .document(ownerId)
                .collection("summary_client_index")
                .document(yearMonth)
                .collection("buckets")
                .document(bucketId)
                .set(
                    mapOf(
                        "capacity" to 300L,
                        "size" to if (bucketId == "bucket_000") {
                            minOf(count, 300).toLong()
                        } else {
                            (count - 300).toLong()
                        }
                    )
                )
                .get()
        }

        repeat(count) { index ->

            val bucketId =
                if (index < 300) {
                    "bucket_000"
                } else {
                    "bucket_001"
                }

            repository.insert(
                ownerId = ownerId,
                yearMonth = yearMonth,
                bucketId = bucketId,
                index = SummaryClientIndex(
                    clientId = "client-%03d".format(index),
                    amount = index.toLong(),
                    status = ClientType.RECEIVABLE
                )
            )
        }
    }

    @Test
    fun `copy index preserves same bucket and client data`() {

        val ownerId = "owner-index-copy-test"
        val previousYearMonth = "2026-09"
        val newYearMonth = "2026-10"
        val status = ClientType.RECEIVABLE
        val bucketId = "bucket_000"

        val bucketRepository =
            SummaryClientIndexBucketRepository(
                firestore = firestore,
                properties = ClientLedgerProperties()
            )

        val indexRepository =
            SummaryClientIndexRepository(
                firestore = firestore
            )

        // Create source bucket.
        firestore
            .collection("owners")
            .document(ownerId)
            .collection("summary_client_index")
            .document(previousYearMonth)
            .collection("buckets")
            .document(bucketId)
            .set(
                mapOf(
                    "capacity" to 300L,
                    "size" to 2L
                )
            )
            .get()

        val clientA =
            SummaryClientIndex(
                clientId = "client-A",
                amount = 15_000,
                status = status
            )

        val clientB =
            SummaryClientIndex(
                clientId = "client-B",
                amount = 8_000,
                status = status
            )

        indexRepository.insert(
            ownerId = ownerId,
            yearMonth = previousYearMonth,
            bucketId = bucketId,
            index = clientA
        )

        indexRepository.insert(
            ownerId = ownerId,
            yearMonth = previousYearMonth,
            bucketId = bucketId,
            index = clientB
        )

        // Read source data.
        val sourceBucket =
            bucketRepository.find(
                ownerId = ownerId,
                yearMonth = previousYearMonth,
                bucketId = bucketId
            )

        val sourceClients =
            indexRepository.findAllInBucket(
                ownerId = ownerId,
                yearMonth = previousYearMonth,
                bucketId = bucketId
            )

        requireNotNull(sourceBucket)

        // Copy bucket + clients atomically.
        firestore
            .runTransaction { transaction ->

                bucketRepository.copyBucketInTransaction(
                    transaction = transaction,
                    ownerId = ownerId,
                    previousYearMonth = previousYearMonth,
                    newYearMonth = newYearMonth,
                    bucketId = bucketId
                )

                sourceClients.forEach { index ->

                    indexRepository.copyInTransaction(
                        transaction = transaction,
                        ownerId = ownerId,
                        newYearMonth = newYearMonth,
                        bucketId = bucketId,
                        index = index
                    )
                }

                null
            }
            .get()

        // Verify bucket ID was preserved.
        val newBucketIds =
            bucketRepository.findBucketIds(
                ownerId = ownerId,
                yearMonth = newYearMonth
            )

        assertEquals(
            listOf(bucketId),
            newBucketIds
        )

        // Verify bucket metadata was preserved.
        val newBucket =
            bucketRepository.find(
                ownerId = ownerId,
                yearMonth = newYearMonth,
                bucketId = bucketId
            )

        assertEquals(
            sourceBucket,
            newBucket
        )

        // Verify client index data was preserved.
        val newClients =
            indexRepository.findAllInBucket(
                ownerId = ownerId,
                yearMonth = newYearMonth,
                bucketId = bucketId
            )

        assertEquals(
            sourceClients,
            newClients
        )
    }
}
