package com.clientledger.core.controller

import com.clientledger.core.app.App
import com.clientledger.core.domain.Client
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientMonthlyHistoryRepository
import com.fasterxml.jackson.databind.ObjectMapper
import com.google.cloud.firestore.Firestore
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.BeforeEach
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

    @Autowired
    private lateinit var firestore: Firestore

    @BeforeEach
    fun cleanup() {
        firestore
            .collection("owners")
            .document("OWNER001")
            .collection("clients")
            .get()
            .get()
            .documents
            .forEach {
                it.reference.delete().get()
            }

        firestore
            .collection("owners")
            .document("OWNER001")
            .collection("master_history_client_data")
            .get()
            .get()
            .documents
            .forEach { yearDocument ->

                yearDocument.reference
                    .listCollections()
                    .forEach { monthCollection ->

                        monthCollection
                            .get()
                            .get()
                            .documents
                            .forEach { bucketDocument ->

                                bucketDocument.reference
                                    .collection("client_histories")
                                    .get()
                                    .get()
                                    .documents
                                    .forEach {
                                        it.reference.delete().get()
                                    }

                                bucketDocument.reference.delete().get()
                            }
                    }

                yearDocument.reference.delete().get()
            }
    }
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
