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
}
