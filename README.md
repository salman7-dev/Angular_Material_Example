package com.clientledger.core.service

import com.clientledger.core.repository.client.ClientRepository
import org.springframework.stereotype.Service

@Service
class ClientHistoryBucketService(
    private val clientRepository: ClientRepository
) {

    companion object {
        private const val BUCKET_CAPACITY = 50
    }

    fun allocateBucket(): String {
        val clientCount = clientRepository.count()
        val bucketNumber = clientCount / BUCKET_CAPACITY

        return "bucket_${bucketNumber.toString().padStart(3, '0')}"
    }
}package com.clientledger.core.repository.history

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
        private const val CLIENT_HISTORIES = "client_histories"
        private const val CAPACITY = 50
    }

    fun save(
        history: ClientMonthlyHistory,
        bucketId: String
    ) {
        val bucketDocument = getBucketDocument(
            history.yearMonth,
            bucketId
        )

        val clientDocument = getClientHistoryDocument(
            history.yearMonth,
            bucketId,
            history.clientId
        )

        firestore.runTransaction { transaction ->

            val bucketSnapshot = transaction
                .get(bucketDocument)
                .get()

            if (!bucketSnapshot.exists()) {

                val bucket = ClientHistoryBucket(
                    capacity = CAPACITY,
                    size = 1
                )

                transaction.set(bucketDocument, bucket)
                transaction.set(clientDocument, history)

                return@runTransaction
            }

            val bucket = bucketSnapshot
                .toObject(ClientHistoryBucket::class.java)
                ?: ClientHistoryBucket(capacity = CAPACITY)

            val clientSnapshot = transaction
                .get(clientDocument)
                .get()

            if (!clientSnapshot.exists()) {

                if (bucket.size >= bucket.capacity) {
                    throw IllegalStateException(
                        "Bucket $bucketId is full"
                    )
                }

                transaction.update(
                    bucketDocument,
                    "size",
                    bucket.size + 1
                )
            }

            transaction.set(clientDocument, history)
        }.get()
    }

    fun find(
        clientId: String,
        yearMonth: String,
        bucketId: String
    ): ClientMonthlyHistory? {

        val snapshot = getClientHistoryDocument(
            yearMonth,
            bucketId,
            clientId
        )
            .get()
            .get()

        if (!snapshot.exists()) {
            return null
        }

        return snapshot.toObject(ClientMonthlyHistory::class.java)
    }

    fun getBucketSize(
        yearMonth: String,
        bucketId: String
    ): Int {

        val snapshot = getBucketDocument(
            yearMonth,
            bucketId
        )
            .get()
            .get()

        if (!snapshot.exists()) {
            return 0
        }

        return snapshot
            .toObject(ClientHistoryBucket::class.java)
            ?.size
            ?: 0
    }

    private fun getBucketDocument(
        yearMonth: String,
        bucketId: String
    ) = firestore
        .collection(COLLECTION)
        .document(yearMonth.substring(0, 4))
        .collection(yearMonth.substring(5, 7))
        .document(bucketId)

    private fun getClientHistoryDocument(
        yearMonth: String,
        bucketId: String,
        clientId: String
    ) = getBucketDocument(
        yearMonth,
        bucketId
    )
        .collection(CLIENT_HISTORIES)
        .document(clientId)

    fun existsForOwner(
        ownerId: String,
        yearMonth: String
    ): Boolean {

        val bucketsSnapshot = firestore
            .collection(COLLECTION)
            .document(yearMonth.substring(0, 4))
            .collection(yearMonth.substring(5, 7))
            .get()
            .get()

        if (bucketsSnapshot.isEmpty) {
            return false
        }

        return bucketsSnapshot.documents.any { bucketDocument ->

            val clientHistories = bucketDocument
                .reference
                .collection(CLIENT_HISTORIES)
                .get()
                .get()

            clientHistories.documents.any { clientDocument ->

                val history = clientDocument
                    .toObject(ClientMonthlyHistory::class.java)

                history?.ownerId == ownerId
            }
        }
    }

    fun findAllForOwner(
        ownerId: String,
        yearMonth: String
    ): List<Pair<String, ClientMonthlyHistory>> {

        val bucketsSnapshot = firestore
            .collection(COLLECTION)
            .document(yearMonth.substring(0, 4))
            .collection(yearMonth.substring(5, 7))
            .get()
            .get()

        if (bucketsSnapshot.isEmpty) {
            return emptyList()
        }

        return bucketsSnapshot.documents.flatMap { bucketDocument ->

            val clientHistories = bucketDocument
                .reference
                .collection(CLIENT_HISTORIES)
                .get()
                .get()

            clientHistories.documents.mapNotNull { clientDocument ->

                val history = clientDocument
                    .toObject(ClientMonthlyHistory::class.java)
                    ?: return@mapNotNull null

                if (history.ownerId != ownerId) {
                    return@mapNotNull null
                }

                bucketDocument.id to history
            }
        }
    }
}package com.clientledger.core.service

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
