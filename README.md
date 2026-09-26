createClientThroughHttpReturnsCreatedClient()
java.lang.AssertionError: Status expected:<201> but was:<401>
	at org.springframework.test.util.AssertionErrors.fail(AssertionErrors.java:59)
	at org.springframework.test.util.AssertionErrors.assertEquals(AssertionErrors.java:122)
	at org.springframework.test.web.servlet.result.StatusResultMatchers.lambda$matcher$9(StatusResultMatchers.java:637)
	at org.springframework.test.web.servlet.MockMvc$1.andExpect(MockMvc.java:214)
	at com.clientledger.core.integration.client.ClientControllerIntegrationTest.createClientThroughHttpReturnsCreatedClient(ClientControllerIntegrationTest.kt:77)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
getClientThroughHttpReturnsClient()
java.lang.AssertionError: Status expected:<201> but was:<401>
	at org.springframework.test.util.AssertionErrors.fail(AssertionErrors.java:59)
	at org.springframework.test.util.AssertionErrors.assertEquals(AssertionErrors.java:122)
	at org.springframework.test.web.servlet.result.StatusResultMatchers.lambda$matcher$9(StatusResultMatchers.java:637)
	at org.springframework.test.web.servlet.MockMvc$1.andExpect(MockMvc.java:214)
	at com.clientledger.core.integration.client.ClientControllerIntegrationTest.createClient(ClientControllerIntegrationTest.kt:286)
	at com.clientledger.core.integration.client.ClientControllerIntegrationTest.getClientThroughHttpReturnsClient(ClientControllerIntegrationTest.kt:105)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
getClientsThroughHttpReturnsFirstPage()
java.lang.AssertionError: Status expected:<201> but was:<401>
	at org.springframework.test.util.AssertionErrors.fail(AssertionErrors.java:59)
	at org.springframework.test.util.AssertionErrors.assertEquals(AssertionErrors.java:122)
	at org.springframework.test.web.servlet.result.StatusResultMatchers.lambda$matcher$9(StatusResultMatchers.java:637)
	at org.springframework.test.web.servlet.MockMvc$1.andExpect(MockMvc.java:214)
	at com.clientledger.core.integration.client.ClientControllerIntegrationTest.createClient(ClientControllerIntegrationTest.kt:286)
	at com.clientledger.core.integration.client.ClientControllerIntegrationTest.getClientsThroughHttpReturnsFirstPage(ClientControllerIntegrationTest.kt:142)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
getClientsThroughHttpReturnsNextPageUsingCursor()
java.lang.AssertionError: Status expected:<201> but was:<401>
	at org.springframework.test.util.AssertionErrors.fail(AssertionErrors.java:59)
	at org.springframework.test.util.AssertionErrors.assertEquals(AssertionErrors.java:122)
	at org.springframework.test.web.servlet.result.StatusResultMatchers.lambda$matcher$9(StatusResultMatchers.java:637)
	at org.springframework.test.web.servlet.MockMvc$1.andExpect(MockMvc.java:214)
	at com.clientledger.core.integration.client.ClientControllerIntegrationTest.createClient(ClientControllerIntegrationTest.kt:286)
	at com.clientledger.core.integration.client.ClientControllerIntegrationTest.getClientsThroughHttpReturnsNextPageUsingCursor(ClientControllerIntegrationTest.kt:195)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
getUnknownClientThroughHttpReturnsNotFound()
java.lang.AssertionError: Status expected:<404> but was:<401>
	at org.springframework.test.util.AssertionErrors.fail(AssertionErrors.java:59)
	at org.springframework.test.util.AssertionErrors.assertEquals(AssertionErrors.java:122)
	at org.springframework.test.web.servlet.result.StatusResultMatchers.lambda$matcher$9(StatusResultMatchers.java:637)
	at org.springframework.test.web.servlet.MockMvc$1.andExpect(MockMvc.java:214)
	at com.clientledger.core.integration.client.ClientControllerIntegrationTest.getUnknownClientThroughHttpReturnsNotFound(ClientControllerIntegrationTest.kt:136)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)


  PS D:\New folder\client-ledger-codespace-main> .\gradlew.bat :app:test                                                    
Reusing configuration cache.
Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :app:test

ClientControllerIntegrationTest > createClientThroughHttpReturnsCreatedClient() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:77

ClientControllerIntegrationTest > getClientsThroughHttpReturnsFirstPage() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:286

ClientControllerIntegrationTest > getClientThroughHttpReturnsClient() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:286

ClientControllerIntegrationTest > getClientsThroughHttpReturnsNextPageUsingCursor() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:286

ClientControllerIntegrationTest > getUnknownClientThroughHttpReturnsNotFound() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:136

97 tests completed, 5 failed

> Task :app:test FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:test'.
> There were failing tests. See the report at: file:///D:/New%20folder/client-ledger-codespace-main/app/build/reports/tests/test/index.html




package com.clientledger.core.integration.client

import com.clientledger.core.auth.CurrentOwnerResolver
import com.clientledger.core.domain.Client
import com.clientledger.core.repository.client.ClientRepository
import com.fasterxml.jackson.databind.ObjectMapper
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.mock.mockito.MockBean
import org.springframework.http.MediaType
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.status
import org.mockito.Mockito.`when`

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
class ClientControllerIntegrationTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Autowired
    private lateinit var objectMapper: ObjectMapper

    @Autowired
    private lateinit var clientRepository: ClientRepository

    @MockBean
    private lateinit var currentOwnerResolver: CurrentOwnerResolver

    private lateinit var ownerId: String

    @BeforeEach
    fun setUp() {
        ownerId = "integration-owner-${System.nanoTime()}"

        `when`(
            currentOwnerResolver.getOwnerId()
        ).thenReturn(ownerId)
    }

    @AfterEach
    fun tearDown() {
        // Each test uses a unique ownerId.
        // No shared client data remains between tests.
    }

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
                .andExpect(status().isCreated)
                .andReturn()

        val response =
            objectMapper.readValue(
                result.response.contentAsString,
                Client::class.java
            )

        assertTrue(response.id.isNotBlank())
        assertEquals(ownerId, response.ownerId)
        assertEquals("HTTP Client", response.name)
        assertEquals("9876543210", response.phone)
        assertEquals(10000, response.initialOpeningBalance)

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
                .andExpect(status().isOk)
                .andReturn()

        val response =
            objectMapper.readValue(
                result.response.contentAsString,
                Client::class.java
            )

        assertEquals(createdClient.id, response.id)
        assertEquals(ownerId, response.ownerId)
        assertEquals("GET Client", response.name)
    }

    @Test
    fun getUnknownClientThroughHttpReturnsNotFound() {

        mockMvc.perform(
            get("/api/clients/CLI-NOT-FOUND")
        )
            .andExpect(status().isNotFound)
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
                .andExpect(status().isOk)
                .andReturn()

        val json =
            objectMapper.readTree(
                result.response.contentAsString
            )

        val content =
            json.get("content")

        assertEquals(2, content.size())
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
                .andExpect(status().isOk)
                .andReturn()

        val firstPageJson =
            objectMapper.readTree(
                firstPageResult.response.contentAsString
            )

        val cursor =
            firstPageJson
                .get("nextCursor")
                .asText()

        assertTrue(cursor.isNotBlank())

        val secondPageResult =
            mockMvc.perform(
                get("/api/clients")
                    .param("size", "2")
                    .param("cursor", cursor)
            )
                .andExpect(status().isOk)
                .andReturn()

        val secondPageJson =
            objectMapper.readTree(
                secondPageResult.response.contentAsString
            )

        val content =
            secondPageJson.get("content")

        assertEquals(1, content.size())

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
                .andExpect(status().isCreated)
                .andReturn()

        return objectMapper.readValue(
            result.response.contentAsString,
            Client::class.java
        )
    }
}

* Try:
> Run with --scan to get full insights.

BUILD FAILED in 59s
9 actionable tasks: 4 executed, 5 up-to-date
Configuration cache entry reused.
PS D:\New folder\client-ledger-codespace-main> 
