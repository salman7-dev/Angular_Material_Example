package com.clientledger.core.repository.operation

import com.clientledger.core.domain.ClientOperationState
import com.clientledger.core.domain.OperationStatus
import com.clientledger.core.domain.OperationType
import com.google.cloud.firestore.Firestore
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import java.time.Instant

class ClientOperationStateRepositoryTest {

    private lateinit var repository: ClientOperationStateRepository

    private lateinit var firestore: Firestore

    @BeforeEach
    fun setUp() {
        firestore = com.google.cloud.firestore.FirestoreOptions.newBuilder()
            .setProjectId("client-ledger-codespace")
            .setHost("127.0.0.1:8080")
            .setEmulatorHost("127.0.0.1:8080")
            .setCredentials(com.google.auth.oauth2.NoCredentials.getInstance())
            .build()
            .service

        repository = ClientOperationStateRepository(firestore)
    }

    @Test
    fun savesAndFindsOperationState() {
        val state = ClientOperationState(
            clientId = "CLI-TEST-001",
            operationId = "OPR-TEST-001",
            operationType = OperationType.ORDER,
            status = OperationStatus.RUNNING,
            startedAt = Instant.now(),
            updatedAt = Instant.now(),
            leaseUntil = Instant.now().plusSeconds(60)
        )

        repository.save(state)

        val result = repository.find("CLI-TEST-001")

        assertNotNull(result)
        assertEquals("CLI-TEST-001", result?.clientId)
        assertEquals("OPR-TEST-001", result?.operationId)
        assertEquals(OperationType.ORDER, result?.operationType)
        assertEquals(OperationStatus.RUNNING, result?.status)
    }

    @Test
    fun returnsNullWhenStateDoesNotExist() {
        val result = repository.find("CLI-DOES-NOT-EXIST")

        assertNull(result)
    }

    @Test
    fun deletesOperationState() {
        val state = ClientOperationState(
            clientId = "CLI-TEST-002",
            operationId = "OPR-TEST-002",
            operationType = OperationType.PAYMENT,
            status = OperationStatus.COMPLETED,
            startedAt = Instant.now(),
            updatedAt = Instant.now(),
            leaseUntil = null
        )

        repository.save(state)

        assertNotNull(repository.find("CLI-TEST-002"))

        repository.delete("CLI-TEST-002")

        assertNull(repository.find("CLI-TEST-002"))
    }
}
