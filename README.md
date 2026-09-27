PS D:\New folder\client-ledger-codespace-main> .\gradlew.bat :app:test --tests "com.clientledger.core.service.owner.OwnerServiceTest"
Reusing configuration cache.
Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :app:test

OwnerServiceTest > createOwnerUsesFirebaseUidAsOwnerId() FAILED
    java.lang.NullPointerException at OwnerServiceTest.kt:62

OwnerServiceTest > createOwnerRejectsBlankOwnerId() FAILED
    org.mockito.exceptions.misusing.InvalidUseOfMatchersException at OwnerServiceTest.kt:34

OwnerServiceTest > createOwnerPersistsOwner() FAILED
    java.lang.NullPointerException at OwnerServiceTest.kt:139

OwnerServiceTest > createOwnerAssignsGeneralRole() FAILED
    org.mockito.exceptions.misusing.InvalidUseOfMatchersException at OwnerServiceTest.kt:34

10 tests completed, 4 failed

> Task :app:test FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:test'.
> There were failing tests. See the report at: file:///D:/New%20folder/client-ledger-codespace-main/app/build/reports/tests/test/index.html

* Try:
> Run with --scan to get full insights.

BUILD FAILED in 11s
9 actionable tasks: 2 executed, 7 up-to-date
Configuration cache entry reused.
PS D:\New folder\client-ledger-codespace-main> 









package com.clientledger.core.service.client

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientType
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
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

@SpringBootTest
@ActiveProfiles("test")
class ClientServiceTest {
    @Autowired
    private lateinit var properties: ClientLedgerProperties
    
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
        }

        @JvmStatic
        @AfterAll
        fun cleanup() {
            firestore.close()
        }
    }

    @Test
    fun createClientCreatesClientAndEightMonthHistory() {

        val clientService = createClientService()

        val client = Client(
            ownerId = "owner-${System.nanoTime()}",
            name = "ABC Traders",
            phone = "9876543210",
            initialOpeningBalance = 5000
        )

        val savedClient = clientService.create(client)

        assertTrue(savedClient.id.startsWith("CLI-"))
        assertEquals(client.ownerId, savedClient.ownerId)
        assertEquals("ABC Traders", savedClient.name)
        assertEquals(5000, savedClient.initialOpeningBalance)
        assertEquals(5000, savedClient.latestAmount)
        assertEquals("", client.bucketId)
        assertTrue(savedClient.bucketId.isNotBlank())

        val clientRepository = ClientRepository(firestore)

        val storedClient = clientRepository.findById(
            ownerId = client.ownerId,
            clientId = savedClient.id
        )

        assertNotNull(storedClient)
        assertEquals(savedClient.bucketId, storedClient!!.bucketId)

        val historyRepository = ClientHistoryRepository(firestore)

        val expectedMonths = expectedMonths()

        expectedMonths.forEach { yearMonth ->

            val history = historyRepository.find(
                ownerId = client.ownerId,
                yearMonth = yearMonth,
                bucketId = savedClient.bucketId,
                clientId = savedClient.id
            )

            assertNotNull(history)
            assertEquals(savedClient.id, history!!.clientId)
            assertEquals(client.ownerId, history.ownerId)
            assertEquals(yearMonth, history.yearMonth)
            assertEquals(5000, history.openingBalance)
            assertEquals(5000, history.closingBalance)
            assertEquals(5000, history.receivable)
            assertEquals(0, history.advance)
            assertEquals(ClientType.RECEIVABLE, history.status)
        }
    }

    @Test
    fun createClientRejectsDuplicateClientId() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val firstClient = Client(
            id = "CLI-DUPLICATE-001",
            ownerId = ownerId,
            name = "ABC Traders",
            phone = "9876543210",
            initialOpeningBalance = 5000
        )

        clientService.create(firstClient)

        val exception =
            org.junit.jupiter.api.assertThrows<IllegalArgumentException> {
                clientService.create(
                    firstClient.copy(
                        name = "XYZ Traders"
                    )
                )
            }

        assertEquals(
            "Client already exists: CLI-DUPLICATE-001",
            exception.message
        )
    }

    @Test
    fun createClientUsesNextBucketWhenCurrentBucketIsFull() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        repeat(100) { index ->
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Client $index",
                    phone = "900000000$index"
                )
            )
        }

        val hundredFirstClient = clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 101",
                phone = "9111111111"
            )
        )

        assertEquals("bucket_001", hundredFirstClient.bucketId)

        val historyRepository = ClientHistoryRepository(firestore)

        val months = expectedMonths()

        months.forEach { yearMonth ->

            val history = historyRepository.find(
                ownerId = ownerId,
                yearMonth = yearMonth,
                bucketId = "bucket_001",
                clientId = hundredFirstClient.id
            )

            assertNotNull(history)
            assertEquals(
                hundredFirstClient.id,
                history!!.clientId
            )
            assertEquals(
                yearMonth,
                history.yearMonth
            )
        }
    }

    @Test
    fun createClientWithReceivableOpeningBalanceCreatesReceivableHistory() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val client = clientService.create(
            Client(
                ownerId = ownerId,
                name = "Receivable Client",
                phone = "9000000001",
                initialOpeningBalance = 10000
            )
        )

        assertEquals(10000, client.latestAmount)

        val history = findCurrentHistory(
            ownerId = ownerId,
            client = client
        )

        assertEquals(10000, history.openingBalance)
        assertEquals(10000, history.closingBalance)
        assertEquals(10000, history.receivable)
        assertEquals(0, history.advance)
        assertEquals(ClientType.RECEIVABLE, history.status)
    }

    @Test
    fun createClientWithAdvanceOpeningBalanceCreatesAdvanceHistory() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val client = clientService.create(
            Client(
                ownerId = ownerId,
                name = "Advance Client",
                phone = "9000000002",
                initialOpeningBalance = -5000
            )
        )

        assertEquals(-5000, client.latestAmount)

        val history = findCurrentHistory(
            ownerId = ownerId,
            client = client
        )

        assertEquals(-5000, history.openingBalance)
        assertEquals(-5000, history.closingBalance)
        assertEquals(0, history.receivable)
        assertEquals(5000, history.advance)
        assertEquals(ClientType.ADVANCE, history.status)
    }

    @Test
    fun createClientWithZeroOpeningBalanceCreatesSettledHistory() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val client = clientService.create(
            Client(
                ownerId = ownerId,
                name = "Settled Client",
                phone = "9000000003",
                initialOpeningBalance = 0
            )
        )

        assertEquals(0, client.latestAmount)

        val history = findCurrentHistory(
            ownerId = ownerId,
            client = client
        )

        assertEquals(0, history.openingBalance)
        assertEquals(0, history.closingBalance)
        assertEquals(0, history.receivable)
        assertEquals(0, history.advance)
        assertEquals(ClientType.SETTLED, history.status)
    }


    @Test
    fun findByIdReturnsExistingClient() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val createdClient =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Find Client",
                    phone = "9000000004",
                    initialOpeningBalance = 5000
                )
            )

        val foundClient =
            clientService.findById(
                ownerId = ownerId,
                clientId = createdClient.id
            )

        assertNotNull(foundClient)

        assertEquals(
            createdClient.id,
            foundClient!!.id
        )

        assertEquals(
            ownerId,
            foundClient.ownerId
        )

        assertEquals(
            "Find Client",
            foundClient.name
        )

        assertEquals(
            5000,
            foundClient.initialOpeningBalance
        )
    }

    @Test
    fun findByIdReturnsNullWhenClientDoesNotExist() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val result =
            clientService.findById(
                ownerId = ownerId,
                clientId = "CLI-NOT-FOUND"
            )

        assertEquals(
            null,
            result
        )
    }

    @Test
    fun findPageReturnsFirstPage() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 001",
                phone = "9000000001"
            )
        )

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 002",
                phone = "9000000002"
            )
        )

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 003",
                phone = "9000000003"
            )
        )

        val page =
            clientService.findPage(
                ownerId = ownerId,
                size = 2,
                cursor = null
            )

        assertEquals(
            2,
            page.content.size
        )

        assertEquals(
            "Client 001",
            page.content[0].name
        )

        assertEquals(
            "Client 002",
            page.content[1].name
        )

        assertTrue(page.hasNext)
        assertNotNull(page.nextCursor)
    }

    @Test
    fun findPageReturnsNextPageUsingCursor() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 001",
                phone = "9000000001"
            )
        )

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 002",
                phone = "9000000002"
            )
        )

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 003",
                phone = "9000000003"
            )
        )

        val firstPage =
            clientService.findPage(
                ownerId = ownerId,
                size = 2,
                cursor = null
            )

        assertNotNull(firstPage.nextCursor)

        val secondPage =
            clientService.findPage(
                ownerId = ownerId,
                size = 2,
                cursor = firstPage.nextCursor
            )

        assertEquals(
            1,
            secondPage.content.size
        )

        assertEquals(
            "Client 003",
            secondPage.content[0].name
        )

        assertTrue(
            !secondPage.hasNext
        )

        assertEquals(
            null,
            secondPage.nextCursor
        )
    }

    private fun expectedMonths(): List<String> {

        val currentMonth = YearMonth.now(testClock)

        val editableMonths =
            properties.history.editableMonths

        return (editableMonths - 1 downTo 0)
            .map { offset ->
                currentMonth.minusMonths(offset.toLong()).toString()
            }
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

        val clock = testClock

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

    private fun findCurrentHistory(
        ownerId: String,
        client: Client
    ): com.clientledger.core.domain.ClientMonthlyHistory {

        val historyRepository =
            ClientHistoryRepository(firestore)

        val history = historyRepository.find(
            ownerId = ownerId,
            yearMonth = YearMonth.now(testClock).toString(),
            bucketId = client.bucketId,
            clientId = client.id
        )

        assertNotNull(history)

        return history!!
    }
}
