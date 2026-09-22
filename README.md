
package com.clientledger.core.service

import com.clientledger.core.domain.OwnerOperationType
import com.clientledger.core.repository.operation.OwnerOperationStateRepository
import com.clientledger.core.utils.IdGenerator
import com.google.cloud.Timestamp
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Service

@Service
class OwnerOperationLockService(
    private val repository: OwnerOperationStateRepository,
    @Value("\${ledger.operation-lock-lease-seconds:120}")
    private val leaseSeconds: Long
) {

    fun acquire(
        ownerId: String,
        operationType: OwnerOperationType,
        operationId: String = IdGenerator.generateOperationId()
    ): String? {
        val now = Timestamp.now()

        val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
            now.seconds + leaseSeconds,
            now.nanos
        )

        val acquired = repository.acquireOwnerLock(
            ownerId = ownerId,
            operationId = operationId,
            operationType = operationType,
            leaseUntil = leaseUntil
        )

        return if (acquired) operationId else null
    }

    fun complete(
        ownerId: String,
        operationId: String
    ): Boolean {
        return repository.completeOwnerOperation(
            ownerId = ownerId,
            operationId = operationId
        )
    }

    fun fail(
        ownerId: String,
        operationId: String
    ): Boolean {
        return repository.failOwnerOperation(
            ownerId = ownerId,
            operationId = operationId
        )
    }

    fun renew(
        ownerId: String,
        operationId: String
    ): Boolean {
        val now = Timestamp.now()

        val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
            now.seconds + leaseSeconds,
            now.nanos
        )

        return repository.renewOwnerOperation(
            ownerId = ownerId,
            operationId = operationId,
            leaseUntil = leaseUntil
        )
    }
}
