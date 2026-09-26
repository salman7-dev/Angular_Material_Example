package com.clientledger.core.integration.client

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientType
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.service.client.ClientService
import com.clientledger.core.transaction.FirestoreTransactionExecutor
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
import java.time.Clock
import java.time.Instant
import java.time.YearMonth
import java.time.ZoneId
@ActiveProfiles("test")
@SpringBootTest
class ClientCreateIntegrationTest {

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

    @Test
    fun createClientCreatesConsistentDataAcrossAllCollections() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val client = clientService.create(
            Client(
                ownerId = ownerId,
                name = "Integration Client",
                phone = "9876543210",
                initialOpeningBalance = 10000
            )
        )

        assertTrue(client.id.startsWith("CLI-"))
        assertEquals(ownerId, client.ownerId)
        assertEquals(10000, client.initialOpeningBalance)
        assertEquals(10000, client.latestAmount)
        assertEquals(ClientType.RECEIVABLE, client.type)
        assertTrue(client.bucketId.isNotBlank())

        verifyClient(ownerId, client)
        verifyHistory(ownerId, client)
        verifyGlobalSummary(ownerId)
        verifySummaryIndex(ownerId, client)
        verifyHistoryBucket(ownerId, client)
    }

    private fun verifyClient(
        ownerId: String,
        client: Client
    ) {

        val repository = ClientRepository(firestore)

        val storedClient = repository.findById(
            ownerId = ownerId,
            clientId = client.id
        )

        assertNotNull(storedClient)
        assertEquals(client.id, storedClient!!.id)
        assertEquals(ownerId, storedClient.ownerId)
        assertEquals("Integration Client", storedClient.name)
        assertEquals(10000, storedClient.initialOpeningBalance)
        assertEquals(10000, storedClient.latestAmount)
        assertEquals(ClientType.RECEIVABLE, storedClient.type)
        assertEquals(client.bucketId, storedClient.bucketId)
    }

    private fun verifyHistory(
        ownerId: String,
        client: Client
    ) {

        val repository = ClientHistoryRepository(firestore)

        val expectedMonths = expectedMonths()

        expectedMonths.forEach { yearMonth ->

            val history = repository.find(
                ownerId = ownerId,
                yearMonth = yearMonth,
                bucketId = client.bucketId,
                clientId = client.id
            )

            assertNotNull(history)

            assertEquals(client.id, history!!.clientId)
            assertEquals(ownerId, history.ownerId)
            assertEquals(yearMonth, history.yearMonth)

            assertEquals(10000, history.openingBalance)
            assertEquals(10000, history.closingBalance)
            assertEquals(10000, history.receivable)
            assertEquals(0, history.advance)
            assertEquals(ClientType.RECEIVABLE, history.status)

            assertEquals(0, history.totalInvoiceAmount)
            assertEquals(0, history.totalPayments)
            assertEquals(0, history.totalDiscount)
            assertEquals(0, history.totalExpenses)
            assertEquals(0, history.totalGstAmount)
        }
    }

    private fun verifyGlobalSummary(
        ownerId: String
    ) {

        val repository = GlobalSummaryRepository(firestore)

        val expectedMonths = expectedMonths()

        expectedMonths.forEach { yearMonth ->

            val summary = repository.find(
                ownerId = ownerId,
                yearMonth = yearMonth
            )

            assertNotNull(summary)

            assertEquals(ownerId, summary!!.ownerId)
            assertEquals(yearMonth, summary.yearMonth)

            assertEquals(1, summary.receivableClientCount)
            assertEquals(0, summary.advanceClientCount)
            assertEquals(0, summary.settledClientCount)

            assertEquals(10000, summary.totalReceivableAmount)
            assertEquals(0, summary.totalAdvanceAmount)

            assertEquals(0, summary.cashFlow)
            assertEquals(0, summary.netProfit)

            assertEquals(0, summary.totalInvoiceAmount)
            assertEquals(0, summary.totalInvoiceAmountWithGst)
            assertEquals(0, summary.totalGstAmount)
            assertEquals(0, summary.totalExpenseAmount)
            assertEquals(0, summary.totalPaymentAmount)
        }
    }

    private fun verifySummaryIndex(
        ownerId: String,
        client: Client
    ) {

        val repository = SummaryClientIndexRepository(firestore)

        val expectedMonths = expectedMonths()

        expectedMonths.forEach { yearMonth ->

            val index = repository.find(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = ClientType.RECEIVABLE,
                bucketId = "bucket_000",
                clientId = client.id
            )

            assertNotNull(index)

            assertEquals(client.id, index!!.clientId)
            assertEquals(10000, index.amount)
            assertEquals(ClientType.RECEIVABLE, index.status)
        }
    }

    private fun verifyHistoryBucket(
        ownerId: String,
        client: Client
    ) {

        val repository = ClientHistoryBucketRepository(
            firestore = firestore,
            properties = properties
        )

        val bucket = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            bucketId = client.bucketId
        )

        assertNotNull(bucket)

        assertEquals(100, bucket!!.capacity)
        assertEquals(1, bucket.size)
    }

    private fun createClientService(): ClientService {

        val properties = properties

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

        val clock = Clock.fixed(
            Instant.parse("2026-09-24T10:00:00Z"),
            ZoneId.of("UTC")
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
            properties = properties,
            clock = clock
        )
    }


    private fun expectedMonths(): List<String> {

        val currentMonth = YearMonth.now(
            Clock.fixed(
                Instant.parse("2026-09-24T10:00:00Z"),
                ZoneId.of("UTC")
            )
        )

        val editableMonths =
            properties.history.editableMonths

        return (editableMonths - 1 downTo 0)
            .map { offset ->
                currentMonth.minusMonths(offset.toLong()).toString()
            }
    }


}

