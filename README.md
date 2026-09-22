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


package com.clientledger.core.service

import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientMonthlyHistory
import com.clientledger.core.repository.history.ClientMonthlyHistoryRepository
import org.springframework.beans.factory.annotation.Value
import org.springframework.stereotype.Service
import java.time.Clock
import java.time.YearMonth

@Service
class ClientMonthlyHistoryMaterializationService(
    private val historyRepository: ClientMonthlyHistoryRepository,
    private val clock: Clock,
    @Value("\${ledger.editable-months:8}")
    private val editableMonths: Int
) {

    fun materializeForNewClient(client: Client) {
        val currentMonth = YearMonth.now(clock)
        val startMonth = currentMonth.minusMonths(editableMonths.toLong() - 1)

        var openingBalance = client.initialOpeningBalance

        var month = startMonth
        while (!month.isAfter(currentMonth)) {
            val yearMonth = month.toString()

            val history = ClientMonthlyHistory(
                clientId = client.id,
                ownerId = client.ownerId,
                yearMonth = yearMonth,
                openingBalance = openingBalance,
                closingBalance = openingBalance,
                receivable = if (openingBalance > 0) openingBalance else 0,
                advance = if (openingBalance < 0) -openingBalance else 0,
                status = getClientType(openingBalance)
            )

            historyRepository.save(
                history = history,
                bucketId = client.bucketId
            )

            month = month.plusMonths(1)
        }
    }

    private fun getClientType(amount: Long) =
        when {
            amount > 0 -> com.clientledger.core.domain.ClientType.RECEIVABLE
            amount < 0 -> com.clientledger.core.domain.ClientType.ADVANCE
            else -> com.clientledger.core.domain.ClientType.SETTLED
        }
}


package com.clientledger.core.repository.history

import com.clientledger.core.domain.ClientHistoryBucket
import com.clientledger.core.domain.ClientMonthlyHistory
import com.google.cloud.firestore.Firestore
import org.springframework.stereotype.Repository

@Repository
class ClientMonthlyHistoryRepository(
    private val firestore: Firestore
) {

    companion object {
        private const val COLLECTION = "master_history_client_data"
        private const val CAPACITY = 50
    }

    fun save(
        history: ClientMonthlyHistory,
        bucketId: String
    ) {
        val document = getBucketDocument(history.yearMonth, bucketId)
        val snapshot = document.get().get()

        if (!snapshot.exists()) {
            val bucket = ClientHistoryBucket(
                capacity = CAPACITY,
                size = 1,
                clients = mapOf(history.clientId to history)
            )

            document.set(bucket).get()
            return
        }

        val bucket = snapshot.toObject(ClientHistoryBucket::class.java)
            ?: ClientHistoryBucket(capacity = CAPACITY)

        val isNewClient = !bucket.clients.containsKey(history.clientId)

        val updatedClients = bucket.clients.toMutableMap()
        updatedClients[history.clientId] = history

        val updatedBucket = bucket.copy(
            size = if (isNewClient) bucket.size + 1 else bucket.size,
            clients = updatedClients
        )

        document.set(updatedBucket).get()
    }

    fun find(
        clientId: String,
        yearMonth: String,
        bucketId: String
    ): ClientMonthlyHistory? {

        val snapshot = getBucketDocument(yearMonth, bucketId)
            .get()
            .get()

        if (!snapshot.exists()) {
            return null
        }

        val bucket = snapshot.toObject(ClientHistoryBucket::class.java)
            ?: return null

        return bucket.clients[clientId]
    }

    fun getBucketSize(
        yearMonth: String,
        bucketId: String
    ): Int {

        val snapshot = getBucketDocument(yearMonth, bucketId)
            .get()
            .get()

        if (!snapshot.exists()) {
            return 0
        }

        val bucket = snapshot.toObject(ClientHistoryBucket::class.java)
            ?: return 0

        return bucket.size
    }

    private fun getBucketDocument(
        yearMonth: String,
        bucketId: String
    ) = firestore
        .collection(COLLECTION)
        .document(yearMonth.substring(0, 4))
        .collection(yearMonth.substring(5, 7))
        .document(bucketId)
}
