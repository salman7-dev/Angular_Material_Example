package com.clientledger.core.repository.client

import com.clientledger.core.domain.Address
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientType
import com.clientledger.core.service.CurrentOwnerProvider
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Assertions.assertNull
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.mockito.kotlin.mock
import org.mockito.kotlin.whenever

class ClientRepositoryTest {

    private lateinit var firestore: Firestore
    private lateinit var currentOwnerProvider: CurrentOwnerProvider
    private lateinit var repository: ClientRepository

    @BeforeEach
    fun setUp() {

        firestore = FirestoreOptions.newBuilder()
            .setProjectId("client-ledger-codespace")
            .setHost("127.0.0.1:8080")
            .setEmulatorHost("127.0.0.1:8080")
            .setCredentials(NoCredentials.getInstance())
            .build()
            .service

        currentOwnerProvider = mock()

        repository = ClientRepository(
            firestore,
            currentOwnerProvider
        )

        firestore.collectionGroup("clients")
            .get()
            .get()
            .documents
            .forEach {
                it.reference.delete().get()
            }
    }

    @AfterEach
    fun tearDown() {
        firestore.close()
    }

    @Test
    fun savesAndFindsClientForOwner() {

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER001")

        val client = Client(
            id = "CLI-001",
            ownerId = "OWNER001",
            name = "ABC Traders",
            phone = "9876543210",
            email = "abc@test.com",
            gstNumber = "GST123",
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

        repository.save(client)

        val result = repository.findById("CLI-001")

        assertNotNull(result)
        assertEquals("CLI-001", result?.id)
        assertEquals("OWNER001", result?.ownerId)
        assertEquals("ABC Traders", result?.name)
        assertEquals(5000, result?.latestAmount)
        assertEquals(ClientType.RECEIVABLE, result?.type)
    }

    @Test
    fun differentOwnersCanHaveSameClientId() {

        val client1 = Client(
            id = "CLI-001",
            ownerId = "OWNER001",
            name = "Owner One Client",
            phone = "1111111111"
        )

        val client2 = Client(
            id = "CLI-001",
            ownerId = "OWNER002",
            name = "Owner Two Client",
            phone = "2222222222"
        )

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER001")

        repository.save(client1)

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER002")

        repository.save(client2)

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER001")

        val ownerOneClient = repository.findById("CLI-001")

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER002")

        val ownerTwoClient = repository.findById("CLI-001")

        assertEquals("Owner One Client", ownerOneClient?.name)
        assertEquals("Owner Two Client", ownerTwoClient?.name)
        assertEquals("OWNER001", ownerOneClient?.ownerId)
        assertEquals("OWNER002", ownerTwoClient?.ownerId)
    }

    @Test
    fun findByIdDoesNotCrossOwnerBoundary() {

        val client = Client(
            id = "CLI-001",
            ownerId = "OWNER001",
            name = "Owner One Client",
            phone = "1111111111"
        )

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER001")

        repository.save(client)

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER002")

        val result = repository.findById("CLI-001")

        assertNull(result)
    }

    @Test
    fun countIsScopedToCurrentOwner() {

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER001")

        repository.save(
            Client(
                id = "CLI-001",
                ownerId = "OWNER001",
                name = "Client One",
                phone = "1111111111"
            )
        )

        repository.save(
            Client(
                id = "CLI-002",
                ownerId = "OWNER001",
                name = "Client Two",
                phone = "2222222222"
            )
        )

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER002")

        repository.save(
            Client(
                id = "CLI-003",
                ownerId = "OWNER002",
                name = "Client Three",
                phone = "3333333333"
            )
        )

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER001")

        assertEquals(2, repository.count())

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER002")

        assertEquals(1, repository.count())
    }

    @Test
    fun deleteRemovesOnlyCurrentOwnersClient() {

        val client = Client(
            id = "CLI-001",
            ownerId = "OWNER001",
            name = "Owner One Client",
            phone = "1111111111"
        )

        whenever(currentOwnerProvider.getOwnerId())
            .thenReturn("OWNER001")

        repository.save(client)
        repository.delete("CLI-001")

        assertNull(repository.findById("CLI-001"))
    }
}
