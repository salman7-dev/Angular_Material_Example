package com.clientledger.core.repository.operation

import com.clientledger.core.domain.ClientOperationState
import com.clientledger.core.domain.OperationStatus
import com.clientledger.core.domain.OperationType
import com.google.cloud.firestore.Firestore
import org.springframework.stereotype.Repository
import java.time.Instant

@Repository
class ClientOperationStateRepository(
    private val firestore: Firestore
) {

    companion object {
        private const val COLLECTION = "client_operation_state"
    }

    fun save(state: ClientOperationState) {
        firestore
            .collection(COLLECTION)
            .document(state.clientId)
            .set(state)
            .get()
    }

    fun find(clientId: String): ClientOperationState? {
        val snapshot = firestore
            .collection(COLLECTION)
            .document(clientId)
            .get()
            .get()

        if (!snapshot.exists()) {
            return null
        }

        return snapshot.toObject(ClientOperationState::class.java)
    }

    fun delete(clientId: String) {
        firestore
            .collection(COLLECTION)
            .document(clientId)
            .delete()
            .get()
    }

    fun acquireClientLock(
        clientId: String,
        operationId: String,
        operationType: OperationType,
        leaseUntil: Instant
    ): Boolean {
        val document = firestore
            .collection(COLLECTION)
            .document(clientId)

        return firestore.runTransaction { transaction ->
            val snapshot = transaction.get(document).get()

            if (snapshot.exists()) {
                val currentState = snapshot.toObject(ClientOperationState::class.java)

                if (
                    currentState != null &&
                    currentState.status == OperationStatus.RUNNING &&
                    currentState.leaseUntil != null &&
                    currentState.leaseUntil.isAfter(Instant.now())
                ) {
                    return@runTransaction false
                }
            }

            val now = Instant.now()

            val newState = ClientOperationState(
                clientId = clientId,
                operationId = operationId,
                operationType = operationType,
                status = OperationStatus.RUNNING,
                startedAt = now,
                updatedAt = now,
                leaseUntil = leaseUntil
            )

            transaction.set(document, newState)

            true
        }.get()
    }
}
