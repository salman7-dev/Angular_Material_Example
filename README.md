package com.clientledger.core.service

import com.clientledger.core.domain.ClientOperationState
import com.clientledger.core.domain.OperationStatus
import com.clientledger.core.domain.OperationType
import com.clientledger.core.repository.operation.ClientOperationStateRepository
import com.google.cloud.NoCredentials
import com.google.cloud.Timestamp
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test

class ClientOperationLockServiceTest {

    private lateinit var firestore: Firestore
    private lateinit var repository: ClientOperationStateRepository
    private lateinit var service: ClientOperationLockService

    @BeforeEach
    fun setUp() {
        firestore = FirestoreOptions.newBuilder()
            .setProjectId("client-ledger-codespace")
            .setHost("127.0.0.1:8080")
            .setEmulatorHost("127.0.0.1:8080")
            .setCredentials(NoCredentials.getInstance())
            .build()
            .service

        repository = ClientOperationStateRepository(firestore)
        service = ClientOperationLockService(
            repository = repository,
            leaseSeconds = 120
        )
    }

    @Test
    fun acquiresClientLock() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        val operationId = service.acquire(
            clientId = clientId,
            operationType = OperationType.ORDER,
            operationId = "OPR-SERVICE-001"
        )

        assertEquals("OPR-SERVICE-001", operationId)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals("OPR-SERVICE-001", state?.operationId)
        assertEquals(OperationType.ORDER, state?.operationType)
        assertEquals(OperationStatus.RUNNING, state?.status)
        assertNotNull(state?.leaseUntil)
    }

    @Test
    fun blocksSecondOperationForSameClient() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        val first = service.acquire(
            clientId = clientId,
            operationType = OperationType.ORDER,
            operationId = "OPR-SERVICE-002"
        )

        val second = service.acquire(
            clientId = clientId,
            operationType = OperationType.PAYMENT,
            operationId = "OPR-SERVICE-003"
        )

        assertEquals("OPR-SERVICE-002", first)
        assertNull(second)
    }

    @Test
    fun allowsOperationForDifferentClient() {
        val firstClientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"
        val secondClientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        val first = service.acquire(
            clientId = firstClientId,
            operationType = OperationType.ORDER,
            operationId = "OPR-SERVICE-004"
        )

        val second = service.acquire(
            clientId = secondClientId,
            operationType = OperationType.PAYMENT,
            operationId = "OPR-SERVICE-005"
        )

        assertEquals("OPR-SERVICE-004", first)
        assertEquals("OPR-SERVICE-005", second)
    }

    @Test
    fun completesOwnOperation() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        val operationId = service.acquire(
            clientId = clientId,
            operationType = OperationType.ORDER,
            operationId = "OPR-SERVICE-006"
        )

        assertEquals("OPR-SERVICE-006", operationId)

        val completed = service.complete(
            clientId = clientId,
            operationId = operationId!!
        )

        assertTrue(completed)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals(OperationStatus.COMPLETED, state?.status)
        assertNull(state?.leaseUntil)
    }

    @Test
    fun failsOwnOperation() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        val operationId = service.acquire(
            clientId = clientId,
            operationType = OperationType.PAYMENT,
            operationId = "OPR-SERVICE-007"
        )

        assertEquals("OPR-SERVICE-007", operationId)

        val failed = service.fail(
            clientId = clientId,
            operationId = operationId!!
        )

        assertTrue(failed)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals(OperationStatus.FAILED, state?.status)
        assertNull(state?.leaseUntil)
    }

    @Test
    fun allowsNewOperationAfterLeaseExpires() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        val expiredLease = Timestamp.ofTimeSecondsAndNanos(
            Timestamp.now().seconds - 60,
            Timestamp.now().nanos
        )

        repository.save(
            ClientOperationState(
                clientId = clientId,
                operationId = "OPR-SERVICE-008",
                operationType = OperationType.ORDER,
                status = OperationStatus.RUNNING,
                startedAt = expiredLease,
                updatedAt = expiredLease,
                leaseUntil = expiredLease
            )
        )

        val newOperation = service.acquire(
            clientId = clientId,
            operationType = OperationType.PAYMENT,
            operationId = "OPR-SERVICE-009"
        )

        assertEquals("OPR-SERVICE-009", newOperation)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals("OPR-SERVICE-009", state?.operationId)
        assertEquals(OperationType.PAYMENT, state?.operationType)
        assertEquals(OperationStatus.RUNNING, state?.status)
    }

    @Test
    fun generatesOperationIdWhenNotProvided() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        val operationId = service.acquire(
            clientId = clientId,
            operationType = OperationType.ORDER
        )

        assertNotNull(operationId)
        assertTrue(operationId!!.startsWith("OPR-"))

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals(operationId, state?.operationId)
    }

    @Test
    fun rejectsCompletionWithWrongOperationId() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        service.acquire(
            clientId = clientId,
            operationType = OperationType.ORDER,
            operationId = "OPR-SERVICE-010"
        )

        val completed = service.complete(
            clientId = clientId,
            operationId = "OPR-WRONG-001"
        )

        assertFalse(completed)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals("OPR-SERVICE-010", state?.operationId)
        assertEquals(OperationStatus.RUNNING, state?.status)
    }

    @Test
    fun rejectsFailureWithWrongOperationId() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        service.acquire(
            clientId = clientId,
            operationType = OperationType.PAYMENT,
            operationId = "OPR-SERVICE-011"
        )

        val failed = service.fail(
            clientId = clientId,
            operationId = "OPR-WRONG-002"
        )

        assertFalse(failed)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals("OPR-SERVICE-011", state?.operationId)
        assertEquals(OperationStatus.RUNNING, state?.status)
    }

    @Test
    fun renewsOwnOperation() {
        val clientId = "CLI-RENEW-${java.util.UUID.randomUUID()}"

        val acquired = service.acquire(
            clientId = clientId,
            operationId = "OPR-RENEW-001",
            operationType = OperationType.ORDER
        )

        assertEquals("OPR-RENEW-001",acquired)

        val renewed = service.renew(
            clientId = clientId,
            operationId = "OPR-RENEW-001"
        )

        assertTrue(renewed)
    }


    @Test
    fun rejectsRenewForWrongOperation() {
        val clientId = "CLI-RENEW-${java.util.UUID.randomUUID()}"

        val acquired = service.acquire(
            clientId = clientId,
            operationId = "OPR-RENEW-002",
            operationType = OperationType.ORDER
        )

        assertEquals("OPR-RENEW-002",acquired)

        val renewed = service.renew(
            clientId = clientId,
            operationId = "OPR-WRONG-001"
        )

        assertFalse(renewed)
    }
}



package com.clientledger.core.service

import com.clientledger.core.domain.OperationType
import com.clientledger.core.repository.operation.ClientOperationStateRepository
import com.clientledger.core.utils.IdGenerator
import com.google.cloud.Timestamp
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Service

@Service
class ClientOperationLockService(
    private val repository: ClientOperationStateRepository,
    @Value("\${ledger.operation-lock-lease-seconds:120}")
    private val leaseSeconds: Long
) {

    fun acquire(
        clientId: String,
        operationType: OperationType,
        operationId: String = IdGenerator.generateOperationId()
    ): String? {
        val now = Timestamp.now()

        val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
            now.seconds + leaseSeconds,
            now.nanos
        )

        val acquired = repository.acquireClientLock(
            clientId = clientId,
            operationId = operationId,
            operationType = operationType,
            leaseUntil = leaseUntil
        )

        return if (acquired) operationId else null
    }

    fun complete(
        clientId: String,
        operationId: String
    ): Boolean {
        return repository.completeClientOperation(
            clientId = clientId,
            operationId = operationId
        )
    }

    fun fail(
        clientId: String,
        operationId: String
    ): Boolean {
        return repository.failClientOperation(
            clientId = clientId,
            operationId = operationId
        )
    }

    fun renew(
        clientId: String,
        operationId: String
    ): Boolean {
        val now = Timestamp.now()

        val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
            now.seconds + leaseSeconds,
            now.nanos
        )

        return repository.renewClientOperation(
            clientId = clientId,
            operationId = operationId,
            leaseUntil = leaseUntil
        )
    }
}
