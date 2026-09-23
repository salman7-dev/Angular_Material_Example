package com.clientledger.core.controller

import com.clientledger.core.app.App
import com.clientledger.core.domain.Client
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientMonthlyHistoryRepository
import com.fasterxml.jackson.databind.ObjectMapper
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.http.MediaType
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import java.time.Clock
import java.time.YearMonth

@SpringBootTest(classes = [App::class])
@AutoConfigureMockMvc
class ClientControllerIntegrationTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Autowired
    private lateinit var objectMapper: ObjectMapper

    @Autowired
    private lateinit var clientRepository: ClientRepository

    @Autowired
    private lateinit var historyRepository: ClientMonthlyHistoryRepository

    @Autowired
    private lateinit var clock: Clock

    @Test
    fun createsClientAndMaterializesEightMonths() {

        val request = mapOf(
            "name" to "ABC Traders",
            "phone" to "9876543210",
            "email" to "abc@example.com",
            "gstNumber" to "27ABCDE1234F1Z5",
            "address" to mapOf(
                "line1" to "Shop 10",
                "city" to "Mumbai",
                "state" to "Maharashtra",
                "pinCode" to "400001"
            ),
            "initialOpeningBalance" to 5000
        )


        val response = mockMvc.perform(
            post("/api/clients")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
        )
            .andReturn()

        println("STATUS = ${response.response.status}")
        println("BODY = ${response.response.contentAsString}")

        val createdClient =
            objectMapper.readValue(response.response.contentAsString, Client::class.java)

        assertNotNull(createdClient.id)
        assertEquals("OWNER001", createdClient.ownerId)
        assertEquals("ABC Traders", createdClient.name)
        assertEquals(5000, createdClient.initialOpeningBalance)
        assertEquals(5000, createdClient.latestAmount)

        val storedClient = clientRepository.findById(createdClient.id)

        assertNotNull(storedClient)
        assertEquals("OWNER001", storedClient?.ownerId)

        val currentMonth = YearMonth.now(clock)

        val expectedMonths = (0L until 8L).map { offset ->
            currentMonth.minusMonths(7L - offset).toString()
        }

        expectedMonths.forEach { yearMonth ->

            val history = historyRepository.find(
                clientId = createdClient.id,
                yearMonth = yearMonth,
                bucketId = createdClient.bucketId
            )

            assertNotNull(history, "Missing history for $yearMonth")
            assertEquals(5000, history?.openingBalance)
            assertEquals(5000, history?.closingBalance)
            assertEquals(5000, history?.receivable)
        }
    }

    @Test
    fun createsSecondClientInExistingBuckets() {

        val firstRequest = mapOf(
            "name" to "ABC Traders",
            "phone" to "9876543210",
            "initialOpeningBalance" to 5000
        )

        val secondRequest = mapOf(
            "name" to "XYZ Traders",
            "phone" to "9876543211",
            "initialOpeningBalance" to 12000
        )

        val firstResponse = mockMvc.perform(
            post("/api/clients")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(firstRequest))
        )
            .andExpect(status().isOk)
            .andReturn()

        val firstClient =
            objectMapper.readValue(firstResponse.response.contentAsString, Client::class.java)

        val secondResponse = mockMvc.perform(
            post("/api/clients")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(secondRequest))
        )
            .andExpect(status().isOk)
            .andReturn()

        val secondClient =
            objectMapper.readValue(secondResponse.response.contentAsString, Client::class.java)

        val currentMonth = YearMonth.now(clock)

        val expectedMonths = (0L until 8L).map { offset ->
            currentMonth.minusMonths(7L - offset).toString()
        }

        expectedMonths.forEach { yearMonth ->

            val firstHistory = historyRepository.find(
                clientId = firstClient.id,
                yearMonth = yearMonth,
                bucketId = firstClient.bucketId
            )

            val secondHistory = historyRepository.find(
                clientId = secondClient.id,
                yearMonth = yearMonth,
                bucketId = secondClient.bucketId
            )

            assertNotNull(firstHistory, "Missing first client history for $yearMonth")
            assertNotNull(secondHistory, "Missing second client history for $yearMonth")

            assertEquals(5000, firstHistory?.openingBalance)
            assertEquals(12000, secondHistory?.openingBalance)

            assertEquals(firstClient.bucketId, secondClient.bucketId)
        }
    }

    @Test
    fun createsNewBucketAfterCapacityReached() {

        val existingClientCount = clientRepository.count()

        val clientsToCreate = 51 - (existingClientCount % 50)

        val createdClients = mutableListOf<Client>()

        for (index in 1..clientsToCreate) {

            val request = mapOf(
                "name" to "Capacity Test Client $index",
                "phone" to "998877${index.toString().padStart(4, '0')}",
                "initialOpeningBalance" to 1000
            )

            val response = mockMvc.perform(
                post("/api/clients")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request))
            )
                .andExpect(status().isOk)
                .andReturn()

            val client =
                objectMapper.readValue(
                    response.response.contentAsString,
                    Client::class.java
                )

            createdClients.add(client)
        }

        val firstBucketNumber = existingClientCount / 50
        val secondBucketNumber = firstBucketNumber + 1

        val firstBucket =
            "bucket_${firstBucketNumber.toString().padStart(3, '0')}"

        val secondBucket =
            "bucket_${secondBucketNumber.toString().padStart(3, '0')}"

        assertEquals(
            firstBucket,
            createdClients.first().bucketId
        )

        assertEquals(
            secondBucket,
            createdClients.last().bucketId
        )
    }

    @Test
    fun updatingSameClientHistoryDoesNotIncreaseBucketSize() {

        val request = mapOf(
            "name" to "Duplicate Test Client",
            "phone" to "9999999999",
            "initialOpeningBalance" to 5000
        )

        val response = mockMvc.perform(
            post("/api/clients")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
        )
            .andExpect(status().isOk)
            .andReturn()

        val createdClient =
            objectMapper.readValue(
                response.response.contentAsString,
                Client::class.java
            )

        val currentMonth = YearMonth.now(clock)
        val yearMonth = currentMonth.toString()

        val initialHistory = historyRepository.find(
            clientId = createdClient.id,
            yearMonth = yearMonth,
            bucketId = createdClient.bucketId
        )

        assertNotNull(initialHistory)
        assertEquals(5000, initialHistory?.openingBalance)

        val initialSize = historyRepository.getBucketSize(
            yearMonth = yearMonth,
            bucketId = createdClient.bucketId
        )

        assertEquals(true, initialSize > 0)

        val updatedHistory = initialHistory!!.copy(
            closingBalance = 7000,
            receivable = 7000
        )

        historyRepository.save(
            history = updatedHistory,
            bucketId = createdClient.bucketId
        )

        val savedHistory = historyRepository.find(
            clientId = createdClient.id,
            yearMonth = yearMonth,
            bucketId = createdClient.bucketId
        )

        assertNotNull(savedHistory)
        assertEquals(7000, savedHistory?.closingBalance)
        assertEquals(7000, savedHistory?.receivable)

        val finalSize = historyRepository.getBucketSize(
            yearMonth = yearMonth,
            bucketId = createdClient.bucketId
        )

        assertEquals(initialSize, finalSize)
    }

    @Test
    fun createsClientWithAdvanceOpeningBalance() {

        val request = mapOf(
            "name" to "Advance Test Client",
            "phone" to "8888888888",
            "initialOpeningBalance" to -5000
        )

        val response = mockMvc.perform(
            post("/api/clients")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
        )
            .andExpect(status().isOk)
            .andReturn()

        val createdClient =
            objectMapper.readValue(
                response.response.contentAsString,
                Client::class.java
            )

        assertNotNull(createdClient.id)
        assertEquals(-5000, createdClient.initialOpeningBalance)
        assertEquals(-5000, createdClient.latestAmount)

        val currentMonth = YearMonth.now(clock)

        val expectedMonths = (0L until 8L).map { offset ->
            currentMonth.minusMonths(7L - offset).toString()
        }

        expectedMonths.forEach { yearMonth ->

            val history = historyRepository.find(
                clientId = createdClient.id,
                yearMonth = yearMonth,
                bucketId = createdClient.bucketId
            )

            assertNotNull(history, "Missing history for $yearMonth")
            assertEquals(-5000, history?.openingBalance)
            assertEquals(-5000, history?.closingBalance)
            assertEquals(0, history?.receivable)
            assertEquals(5000, history?.advance)
            assertEquals(
                com.clientledger.core.domain.ClientType.ADVANCE,
                history?.status
            )
        }
    }

    @Test
    fun createsClientWithSettledOpeningBalance() {

        val request = mapOf(
            "name" to "Settled Test Client",
            "phone" to "7777777777",
            "initialOpeningBalance" to 0
        )

        val response = mockMvc.perform(
            post("/api/clients")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
        )
            .andExpect(status().isOk)
            .andReturn()

        val createdClient =
            objectMapper.readValue(
                response.response.contentAsString,
                Client::class.java
            )

        assertNotNull(createdClient.id)
        assertEquals(0, createdClient.initialOpeningBalance)
        assertEquals(0, createdClient.latestAmount)

        val currentMonth = YearMonth.now(clock)

        val expectedMonths = (0L until 8L).map { offset ->
            currentMonth.minusMonths(7L - offset).toString()
        }

        expectedMonths.forEach { yearMonth ->

            val history = historyRepository.find(
                clientId = createdClient.id,
                yearMonth = yearMonth,
                bucketId = createdClient.bucketId
            )

            assertNotNull(history, "Missing history for $yearMonth")
            assertEquals(0, history?.openingBalance)
            assertEquals(0, history?.closingBalance)
            assertEquals(0, history?.receivable)
            assertEquals(0, history?.advance)
            assertEquals(
                com.clientledger.core.domain.ClientType.SETTLED,
                history?.status
            )
        }
    }
}



package com.clientledger.core.controller

import com.clientledger.core.domain.Client
import com.clientledger.core.dto.client.CreateClientRequest
import com.clientledger.core.service.ClientService
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/clients")
class ClientController(
    private val clientService: ClientService
) {

    @PostMapping
    fun createClient(
        @RequestBody request: CreateClientRequest
    ): ResponseEntity<Client> {

        val client = clientService.createClient(
            name = request.name,
            phone = request.phone,
            email = request.email,
            gstNumber = request.gstNumber,
            address = request.address,
            initialOpeningBalance = request.initialOpeningBalance
        )

        return ResponseEntity.ok(client)
    }
}

package com.clientledger.core.service

import com.clientledger.core.domain.Address
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientType
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.utils.IdGenerator
import org.springframework.stereotype.Service
import java.time.Instant

@Service
class ClientService(
    private val clientRepository: ClientRepository,
    private val clientHistoryBucketService: ClientHistoryBucketService,
    private val clientMonthlyHistoryMaterializationService: ClientMonthlyHistoryMaterializationService,
    private val currentOwnerProvider: CurrentOwnerProvider
) {

    fun createClient(
        name: String,
        phone: String,
        email: String = "",
        gstNumber: String? = null,
        address: Address? = null,
        initialOpeningBalance: Long = 0
    ): Client {
        val ownerId = currentOwnerProvider.getOwnerId()
        val clientId = IdGenerator.generateClientId()
        val bucketId = clientHistoryBucketService.allocateBucket()
        val now = Instant.now()

        val client = Client(
            id = clientId,
            ownerId = ownerId,
            name = name,
            phone = phone,
            email = email,
            gstNumber = gstNumber,
            address = address,
            initialOpeningBalance = initialOpeningBalance,
            latestAmount = initialOpeningBalance,
            type = getClientType(initialOpeningBalance),
            bucketId = bucketId,
            createdAt = now,
            updatedAt = now
        )

        clientRepository.save(client)

        clientMonthlyHistoryMaterializationService.materializeForNewClient(client)

        return client
    }

    private fun getClientType(amount: Long): ClientType {
        return when {
            amount > 0 -> ClientType.RECEIVABLE
            amount < 0 -> ClientType.ADVANCE
            else -> ClientType.SETTLED
        }
    }
}

package com.clientledger.core.service

import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientMonthlyHistory
import com.clientledger.core.repository.history.ClientMonthlyHistoryRepository
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Service
import java.time.Clock
import java.time.YearMonth

@Service
class ClientMonthlyHistoryMaterializationService(
    private val historyRepository: ClientMonthlyHistoryRepository,
    private val clock: Clock,
    @Value("\${ledger.editable-months:8}")
    private val editableMonths: Int
) {

    fun materializeForNewClient(client: Client) {
        val currentMonth = YearMonth.now(clock)
        val startMonth = currentMonth.minusMonths(editableMonths.toLong() - 1)

        var openingBalance = client.initialOpeningBalance

        var month = startMonth
        while (!month.isAfter(currentMonth)) {
            val yearMonth = month.toString()

            val history = ClientMonthlyHistory(
                clientId = client.id,
                ownerId = client.ownerId,
                yearMonth = yearMonth,
                openingBalance = openingBalance,
                closingBalance = openingBalance,
                receivable = if (openingBalance > 0) openingBalance else 0,
                advance = if (openingBalance < 0) -openingBalance else 0,
                status = getClientType(openingBalance)
            )

            historyRepository.save(
                history = history,
                bucketId = client.bucketId
            )

            month = month.plusMonths(1)
        }
    }

    private fun getClientType(amount: Long) =
        when {
            amount > 0 -> com.clientledger.core.domain.ClientType.RECEIVABLE
            amount < 0 -> com.clientledger.core.domain.ClientType.ADVANCE
            else -> com.clientledger.core.domain.ClientType.SETTLED
        }
}

package com.clientledger.core.repository.history

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

        currentOwnerProvider = object : CurrentOwnerProvider {
            override fun getOwnerId(): String = "OWNER001"
        }

        repository = ClientMonthlyHistoryRepository(
            firestore,
            currentOwnerProvider
        )

        val monthReference = firestore
            .collection("owners")
            .document("OWNER001")
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
package com.clientledger.core.service

import com.clientledger.core.domain.ClientMonthlyHistory
import com.clientledger.core.repository.history.ClientMonthlyHistoryRepository
import org.springframework.stereotype.Service
import java.time.Clock
import java.time.YearMonth

@Service
class OwnerMonthInitializationService(
    private val historyRepository: ClientMonthlyHistoryRepository,
    private val clock: Clock
) {

    fun initializeOwner(ownerId: String) {

        val currentMonth = YearMonth.now(clock)
        val previousMonth = currentMonth.minusMonths(1)

        val currentYearMonth = currentMonth.toString()
        val previousYearMonth = previousMonth.toString()

        if (historyRepository.existsForOwner(ownerId, currentYearMonth)) {
            return
        }

        val previousHistories = historyRepository.findAllForOwner(
            ownerId = ownerId,
            yearMonth = previousYearMonth
        )

        previousHistories.forEach { (bucketId, previous) ->

            val closingBalance = previous.closingBalance

            val currentHistory = ClientMonthlyHistory(
                clientId = previous.clientId,
                ownerId = previous.ownerId,
                yearMonth = currentYearMonth,
                openingBalance = closingBalance,
                totalInvoiceAmount = 0,
                totalPayments = 0,
                totalDiscount = 0,
                totalExpenses = 0,
                totalGstAmount = 0,
                closingBalance = closingBalance,
                receivable = if (closingBalance > 0) closingBalance else 0,
                advance = if (closingBalance < 0) -closingBalance else 0,
                status = getClientType(closingBalance)
            )

            historyRepository.save(
                history = currentHistory,
                bucketId = bucketId
            )
        }
    }

    private fun getClientType(amount: Long) =
        when {
            amount > 0 -> com.clientledger.core.domain.ClientType.RECEIVABLE
            amount < 0 -> com.clientledger.core.domain.ClientType.ADVANCE
            else -> com.clientledger.core.domain.ClientType.SETTLED
        }
}

package com.clientledger.core.service

import com.clientledger.core.app.App
import com.clientledger.core.domain.ClientHistoryBucket
import com.clientledger.core.domain.ClientMonthlyHistory
import com.clientledger.core.domain.ClientType
import com.clientledger.core.repository.history.ClientMonthlyHistoryRepository
import com.google.cloud.firestore.Firestore
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import java.time.Clock
import java.time.Instant
import java.time.ZoneId
import kotlin.test.assertEquals

@SpringBootTest(classes = [App::class])
class OwnerMonthInitializationServiceTest {

    @Autowired
    private lateinit var firestore: Firestore

    @Autowired
    private lateinit var repository: ClientMonthlyHistoryRepository

    private lateinit var service: OwnerMonthInitializationService

    private val clock = Clock.fixed(
        Instant.parse("2026-10-01T00:00:00Z"),
        ZoneId.of("UTC")
    )

    @BeforeEach
    fun setup() {
        service = OwnerMonthInitializationService(
            historyRepository = repository,
            clock = clock
        )

        firestore
            .collection("master_history_client_data")
            .document("2026")
            .collection("09")
            .get()
            .get()
            .documents
            .forEach { it.reference.delete().get() }

        firestore
            .collection("master_history_client_data")
            .document("2026")
            .collection("10")
            .get()
            .get()
            .documents
            .forEach { it.reference.delete().get() }
    }

    @Test
    fun skipWhenCurrentMonthAlreadyExists() {

        val existingHistory = ClientMonthlyHistory(
            clientId = "CLI-001",
            ownerId = "OWNER-A",
            yearMonth = "2026-10",
            openingBalance = 10000,
            closingBalance = 10000,
            receivable = 10000,
            status = ClientType.RECEIVABLE
        )

        val bucketReference = firestore
            .collection("master_history_client_data")
            .document("2026")
            .collection("10")
            .document("bucket-001")

        bucketReference
            .set(
                ClientHistoryBucket(
                    capacity = 50,
                    size = 1
                )
            )
            .get()

        bucketReference
            .collection("client_histories")
            .document(existingHistory.clientId)
            .set(existingHistory)
            .get()

        service.initializeOwner("OWNER-A")

        val bucketSnapshot = bucketReference
            .get()
            .get()

        val bucket = bucketSnapshot
            .toObject(ClientHistoryBucket::class.java)

        val historySnapshot = bucketReference
            .collection("client_histories")
            .document("CLI-001")
            .get()
            .get()

        val history = historySnapshot
            .toObject(ClientMonthlyHistory::class.java)

        assertEquals(1, bucket?.size)
        assertEquals(existingHistory, history)
    }

    @Test
    fun createCurrentMonthFromPreviousMonth() {

        val previousHistory = ClientMonthlyHistory(
            clientId = "CLI-001",
            ownerId = "OWNER-A",
            yearMonth = "2026-09",
            openingBalance = 8000,
            totalInvoiceAmount = 5000,
            totalPayments = 2000,
            totalDiscount = 500,
            totalExpenses = 1000,
            totalGstAmount = 900,
            closingBalance = 10500,
            receivable = 10500,
            advance = 0,
            status = ClientType.RECEIVABLE
        )

        val previousBucketReference = firestore
            .collection("master_history_client_data")
            .document("2026")
            .collection("09")
            .document("bucket-001")

        previousBucketReference
            .set(
                ClientHistoryBucket(
                    capacity = 50,
                    size = 1
                )
            )
            .get()

        previousBucketReference
            .collection("client_histories")
            .document(previousHistory.clientId)
            .set(previousHistory)
            .get()

        service.initializeOwner("OWNER-A")

        val currentBucketReference = firestore
            .collection("master_history_client_data")
            .document("2026")
            .collection("10")
            .document("bucket-001")

        val currentHistorySnapshot = currentBucketReference
            .collection("client_histories")
            .document("CLI-001")
            .get()
            .get()

        val currentHistory = currentHistorySnapshot
            .toObject(ClientMonthlyHistory::class.java)

        assertEquals(10500, currentHistory?.openingBalance)
        assertEquals(10500, currentHistory?.closingBalance)
        assertEquals(0, currentHistory?.totalInvoiceAmount)
        assertEquals(0, currentHistory?.totalPayments)
        assertEquals(0, currentHistory?.totalDiscount)
        assertEquals(0, currentHistory?.totalExpenses)
        assertEquals(0, currentHistory?.totalGstAmount)
        assertEquals(10500, currentHistory?.receivable)
        assertEquals(0, currentHistory?.advance)
        assertEquals(ClientType.RECEIVABLE, currentHistory?.status)
    }

    @Test
    fun doNothingWhenPreviousMonthHasNoData() {

        service.initializeOwner("OWNER-A")

        val snapshot = firestore
            .collection("master_history_client_data")
            .document("2026")
            .collection("10")
            .get()
            .get()

        assertEquals(0, snapshot.size())
    }
}

