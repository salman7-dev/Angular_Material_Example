
package com.clientledger.core.repository.operation

import com.clientledger.core.domain.OperationStatus
import com.clientledger.core.domain.OwnerOperationState
import com.clientledger.core.domain.OwnerOperationType
import com.google.cloud.Timestamp
import com.google.cloud.firestore.Firestore
import org.springframework.stereotype.Repository

@Repository
class OwnerOperationStateRepository(
    private val firestore: Firestore
) {

    companion object {
        private const val COLLECTION = "owner_operation_state"
    }

    fun save(state: OwnerOperationState) {
        firestore
            .collection(COLLECTION)
            .document(state.ownerId)
            .set(state)
            .get()
    }

    fun find(ownerId: String): OwnerOperationState? {
        val snapshot = firestore
            .collection(COLLECTION)
            .document(ownerId)
            .get()
            .get()

        if (!snapshot.exists()) {
            return null
        }

        return snapshot.toObject(OwnerOperationState::class.java)
    }

    fun delete(ownerId: String) {
        firestore
            .collection(COLLECTION)
            .document(ownerId)
            .delete()
            .get()
    }

    fun acquireOwnerLock(
        ownerId: String,
        operationId: String,
        operationType: OwnerOperationType,
        leaseUntil: Timestamp
    ): Boolean {
        val document = firestore
            .collection(COLLECTION)
            .document(ownerId)

        return firestore.runTransaction { transaction ->
            val snapshot = transaction.get(document).get()

            if (snapshot.exists()) {
                val currentState = snapshot.toObject(OwnerOperationState::class.java)

                if (
                    currentState != null &&
                    currentState.status == OperationStatus.RUNNING &&
                    currentState.leaseUntil != null &&
                    currentState.leaseUntil.compareTo(Timestamp.now()) > 0
                ) {
                    return@runTransaction false
                }
            }

            val now = Timestamp.now()

            val newState = OwnerOperationState(
                ownerId = ownerId,
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

    fun completeOwnerOperation(
        ownerId: String,
        operationId: String
    ): Boolean {
        val document = firestore
            .collection(COLLECTION)
            .document(ownerId)

        return firestore.runTransaction { transaction ->
            val snapshot = transaction.get(document).get()

            if (!snapshot.exists()) {
                return@runTransaction false
            }

            val currentState = snapshot.toObject(OwnerOperationState::class.java)
                ?: return@runTransaction false

            if (
                currentState.operationId != operationId ||
                currentState.status != OperationStatus.RUNNING
            ) {
                return@runTransaction false
            }

            val completedState = currentState.copy(
                status = OperationStatus.COMPLETED,
                updatedAt = Timestamp.now(),
                leaseUntil = null
            )

            transaction.set(document, completedState)

            true
        }.get()
    }

    fun failOwnerOperation(
        ownerId: String,
        operationId: String
    ): Boolean {
        val document = firestore
            .collection(COLLECTION)
            .document(ownerId)

        return firestore.runTransaction { transaction ->
            val snapshot = transaction.get(document).get()

            if (!snapshot.exists()) {
                return@runTransaction false
            }

            val currentState = snapshot.toObject(OwnerOperationState::class.java)
                ?: return@runTransaction false

            if (
                currentState.operationId != operationId ||
                currentState.status != OperationStatus.RUNNING
            ) {
                return@runTransaction false
            }

            val failedState = currentState.copy(
                status = OperationStatus.FAILED,
                updatedAt = Timestamp.now(),
                leaseUntil = null
            )

            transaction.set(document, failedState)

            true
        }.get()
    }

    fun renewOwnerOperation(
        ownerId: String,
        operationId: String,
        leaseUntil: Timestamp
    ): Boolean {
        val document = firestore
            .collection(COLLECTION)
            .document(ownerId)

        return firestore.runTransaction { transaction ->
            val snapshot = transaction.get(document).get()

            if (!snapshot.exists()) {
                return@runTransaction false
            }

            val currentState = snapshot.toObject(OwnerOperationState::class.java)
                ?: return@runTransaction false

            if (
                currentState.operationId != operationId ||
                currentState.status != OperationStatus.RUNNING
            ) {
                return@runTransaction false
            }

            val renewedState = currentState.copy(
                updatedAt = Timestamp.now(),
                leaseUntil = leaseUntil
            )

            transaction.set(document, renewedState)

            true
        }.get()
    }
}
