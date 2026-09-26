package com.clientledger.core.integration.client

import com.clientledger.core.domain.Client
import com.clientledger.core.repository.client.ClientRepository
import com.fasterxml.jackson.databind.ObjectMapper
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.http.MediaType
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.context.TestPropertySource
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import org.springframework.test.web.servlet.result.MockMvcResultHandlers.print

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@TestPropertySource(
    properties = [
        "client-ledger.auth.enabled=false",
        "client-ledger.auth.local-owner-id=integration-owner"
    ]
)
class ClientControllerIntegrationTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Autowired
    private lateinit var objectMapper: ObjectMapper

    @Autowired
    private lateinit var clientRepository: ClientRepository

    private val ownerId = "integration-owner"



    @Test
    fun createClientThroughHttpReturnsCreatedClient() {

        val request =
            Client(
                name = "HTTP Client",
                phone = "9876543210",
                initialOpeningBalance = 10000
            )

        val result =
            mockMvc.perform(
                post("/api/clients")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(
                        objectMapper.writeValueAsString(request)
                    )
            )
                .andDo { result ->
                    println("HTTP STATUS = ${result.response.status}")
                    println("HTTP BODY = ${result.response.contentAsString}")
                }
                .andExpect(status().isCreated)
                .andReturn()

        val response =
            objectMapper.readValue(
                result.response.contentAsString,
                Client::class.java
            )

        assertTrue(
            response.id.isNotBlank()
        )

        assertEquals(
            ownerId,
            response.ownerId
        )

        assertEquals(
            "HTTP Client",
            response.name
        )

        assertEquals(
            "9876543210",
            response.phone
        )

        assertEquals(
            10000,
            response.initialOpeningBalance
        )

        val storedClient =
            clientRepository.findById(
                ownerId = ownerId,
                clientId = response.id
            )

        assertNotNull(storedClient)
    }

    @Test
    fun getClientThroughHttpReturnsClient() {

        val createdClient =
            createClient(
                name = "GET Client",
                phone = "9000000001"
            )

        val result =
            mockMvc.perform(
                get(
                    "/api/clients/${createdClient.id}"
                )
            )
                .andExpect(
                    status().isOk
                )
                .andReturn()

        val response =
            objectMapper.readValue(
                result.response.contentAsString,
                Client::class.java
            )

        assertEquals(
            createdClient.id,
            response.id
        )

        assertEquals(
            ownerId,
            response.ownerId
        )

        assertEquals(
            "GET Client",
            response.name
        )
    }


    @Test
    fun getUnknownClientThroughHttpReturnsNotFound() {

        mockMvc.perform(
            get("/api/clients/CLI-NOT-FOUND")
        )
            .andDo(print())
            .andExpect(
                status().isNotFound
            )
    }



    @Test
    fun getClientsThroughHttpReturnsFirstPage() {

        createClient(
            name = "Client 001",
            phone = "9000000001"
        )

        createClient(
            name = "Client 002",
            phone = "9000000002"
        )

        createClient(
            name = "Client 003",
            phone = "9000000003"
        )

        val result =
            mockMvc.perform(
                get("/api/clients")
                    .param("size", "2")
            )
                .andExpect(
                    status().isOk
                )
                .andReturn()

        val json =
            objectMapper.readTree(
                result.response.contentAsString
            )

        val content =
            json.get("content")

        assertEquals(
            2,
            content.size()
        )

        assertEquals(
            "Client 001",
            content[0].get("name").asText()
        )

        assertEquals(
            "Client 002",
            content[1].get("name").asText()
        )

        assertTrue(
            json.get("hasNext").asBoolean()
        )

        assertTrue(
            json.get("nextCursor").asText().isNotBlank()
        )
    }

    @Test
    fun getClientsThroughHttpReturnsNextPageUsingCursor() {

        createClient(
            name = "Client 001",
            phone = "9000000001"
        )

        createClient(
            name = "Client 002",
            phone = "9000000002"
        )

        createClient(
            name = "Client 003",
            phone = "9000000003"
        )

        val firstPageResult =
            mockMvc.perform(
                get("/api/clients")
                    .param("size", "2")
            )
                .andExpect(
                    status().isOk
                )
                .andReturn()

        val firstPageJson =
            objectMapper.readTree(
                firstPageResult.response.contentAsString
            )

        val cursor =
            firstPageJson
                .get("nextCursor")
                .asText()

        assertTrue(
            cursor.isNotBlank()
        )

        val secondPageResult =
            mockMvc.perform(
                get("/api/clients")
                    .param("size", "2")
                    .param("cursor", cursor)
            )
                .andExpect(
                    status().isOk
                )
                .andReturn()

        val secondPageJson =
            objectMapper.readTree(
                secondPageResult.response.contentAsString
            )

        val content =
            secondPageJson.get("content")

        assertEquals(
            1,
            content.size()
        )

        assertEquals(
            "Client 003",
            content[0].get("name").asText()
        )

        assertTrue(
            !secondPageJson
                .get("hasNext")
                .asBoolean()
        )

        assertTrue(
            secondPageJson
                .get("nextCursor")
                .isNull
        )
    }

    private fun createClient(
        name: String,
        phone: String
    ): Client {

        val request =
            Client(
                name = name,
                phone = phone
            )

        val result =
            mockMvc.perform(
                post("/api/clients")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(
                        objectMapper.writeValueAsString(request)
                    )
            )
                .andDo { result ->
                    println("HTTP STATUS = ${result.response.status}")
                    println("HTTP BODY = ${result.response.contentAsString}")
                }
                .andExpect(status().isCreated)
                .andReturn()

        return objectMapper.readValue(
            result.response.contentAsString,
            Client::class.java
        )
    }
}

