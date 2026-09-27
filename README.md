package com.clientledger.core.repository.client

import com.clientledger.core.domain.Address
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientType
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test

class ClientRepositoryTest {

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
            if (::firestore.isInitialized) {
                firestore.close()
            }
        }
    }

    @Test
    fun createAndFindClient() {

        val repository = ClientRepository(firestore)

        val client = Client(
            id = "client-001",
            ownerId = "owner-001",
            name = "ABC Traders",
            phone = "9876543210",
            email = "abc@example.com",
            gstNumber = "27ABCDE1234F1Z5",
            address = Address(
                line1 = "Main Road",
                city = "Mumbai",
                state = "Maharashtra",
                pinCode = "400001"
            ),
            initialOpeningBalance = 5000,
            latestAmount = 5000,
            type = ClientType.RECEIVABLE,
            bucketId = "bucket_000"
        )

        repository.create(client)

        val savedClient = repository.findById(
            ownerId = "owner-001",
            clientId = "client-001"
        )

        assertNotNull(savedClient)

        assertEquals("client-001", savedClient!!.id)
        assertEquals("owner-001", savedClient.ownerId)
        assertEquals("ABC Traders", savedClient.name)
        assertEquals(5000, savedClient.initialOpeningBalance)
        assertEquals(5000, savedClient.latestAmount)
        assertEquals(ClientType.RECEIVABLE, savedClient.type)
        assertEquals("bucket_000", savedClient.bucketId)
    }

    @Test
    fun clientExists() {

        val repository = ClientRepository(firestore)

        val client = Client(
            id = "client-002",
            ownerId = "owner-001",
            name = "XYZ Store",
            phone = "9999999999"
        )

        repository.create(client)

        assertTrue(
            repository.exists(
                ownerId = "owner-001",
                clientId = "client-002"
            )
        )
    }

    @Test
    fun existsInTransactionReturnsFalseForMissingClient() {

        val repository = ClientRepository(firestore)

        val ownerId = "owner-${System.nanoTime()}"
        val clientId = "client-${System.nanoTime()}"

        val result = firestore.runTransaction { transaction ->

            repository.existsInTransaction(
                transaction = transaction,
                ownerId = ownerId,
                clientId = clientId
            )
        }.get()

        assertEquals(false, result)
    }

    @Test
    fun createInTransactionCreatesClient() {

        val repository = ClientRepository(firestore)

        val ownerId = "owner-${System.nanoTime()}"
        val clientId = "client-${System.nanoTime()}"

        val client = Client(
            id = clientId,
            ownerId = ownerId,
            name = "Transaction Client",
            phone = "9999999999",
            initialOpeningBalance = 10000,
            latestAmount = 10000,
            type = ClientType.RECEIVABLE,
            bucketId = "bucket_000"
        )

        firestore.runTransaction { transaction ->

            repository.createInTransaction(
                transaction = transaction,
                client = client
            )

            null
        }.get()

        val result = repository.findById(
            ownerId = ownerId,
            clientId = clientId
        )

        assertNotNull(result)
        assertEquals(clientId, result!!.id)
        assertEquals("Transaction Client", result.name)
        assertEquals(10000, result.latestAmount)
        assertEquals(ClientType.RECEIVABLE, result.type)
    }

    @Test
    fun findPageReturnsFirstPageWithNextCursor() {

        val repository = ClientRepository(firestore)

        val ownerId = "owner-${System.nanoTime()}"

        repository.create(
            Client(
                id = "client-001",
                ownerId = ownerId,
                name = "Client 001",
                phone = "9000000001"
            )
        )

        repository.create(
            Client(
                id = "client-002",
                ownerId = ownerId,
                name = "Client 002",
                phone = "9000000002"
            )
        )

        repository.create(
            Client(
                id = "client-003",
                ownerId = ownerId,
                name = "Client 003",
                phone = "9000000003"
            )
        )

        val page =
            repository.findPage(
                ownerId = ownerId,
                size = 2,
                cursor = null
            )

        assertEquals(
            2,
            page.content.size
        )

        assertEquals(
            "client-001",
            page.content[0].id
        )

        assertEquals(
            "client-002",
            page.content[1].id
        )

        assertTrue(page.hasNext)
        assertNotNull(page.nextCursor)
    }

    @Test
    fun findPageReturnsNextPageUsingCursor() {

        val repository = ClientRepository(firestore)

        val ownerId = "owner-${System.nanoTime()}"

        repository.create(
            Client(
                id = "client-001",
                ownerId = ownerId,
                name = "Client 001",
                phone = "9000000001"
            )
        )

        repository.create(
            Client(
                id = "client-002",
                ownerId = ownerId,
                name = "Client 002",
                phone = "9000000002"
            )
        )

        repository.create(
            Client(
                id = "client-003",
                ownerId = ownerId,
                name = "Client 003",
                phone = "9000000003"
            )
        )

        val firstPage =
            repository.findPage(
                ownerId = ownerId,
                size = 2,
                cursor = null
            )

        val secondPage =
            repository.findPage(
                ownerId = ownerId,
                size = 2,
                cursor = firstPage.nextCursor
            )

        assertEquals(
            1,
            secondPage.content.size
        )

        assertEquals(
            "client-003",
            secondPage.content[0].id
        )

        assertEquals(
            false,
            secondPage.hasNext
        )

        assertEquals(
            null,
            secondPage.nextCursor
        )
    }

    @Test
    fun findPageRejectsInvalidCursor() {

        val repository = ClientRepository(firestore)

        val ownerId = "owner-${System.nanoTime()}"

        assertThrows(
            IllegalArgumentException::class.java
        ) {
            repository.findPage(
                ownerId = ownerId,
                size = 10,
                cursor = "invalid-cursor"
            )
        }
    }
}
