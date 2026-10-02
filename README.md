package com.clientledger.core.repository.summary

import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.SummaryClientIndex
import com.clientledger.core.pagination.PageCursor
import com.clientledger.core.pagination.PageResult
import com.google.cloud.firestore.*
import org.springframework.stereotype.Repository
import java.util.*

data class SummaryClientIndexLocation(
    val bucketId: String,
    val index: SummaryClientIndex,
    val bucketExists: Boolean,
    val bucketSize: Long
)

data class SummaryClientIndexUpsertPlan(
    val clientReference: DocumentReference,
    val bucketReference: DocumentReference,
    val index: SummaryClientIndex,
    val bucketExists: Boolean,
    val currentBucketSize: Long,
    val capacity: Long
)

data class SummaryClientIndexDeletePlan(
    val clientReference: DocumentReference,
    val bucketReference: DocumentReference,
    val currentBucketSize: Long
)

@Repository
class SummaryClientIndexRepository(
    private val firestore: Firestore
) {

    private fun statusReference(
        ownerId: String,
        yearMonth: String,
        status: ClientType
    ): DocumentReference {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }

        return firestore
            .collection("owners")
            .document(ownerId)
            .collection("summary_client_index")
            .document(yearMonth)
            .collection("status")
            .document(status.name.lowercase())
    }

    private fun bucketCollection(
        ownerId: String,
        yearMonth: String,
        status: ClientType
    ): CollectionReference {

        return statusReference(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status
        )
            .collection("buckets")
    }

    private fun bucketReference(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String
    ): DocumentReference {

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status
        )
            .document(bucketId)
    }

    private fun clientReference(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String,
        clientId: String
    ): DocumentReference {

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        require(clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        return bucketReference(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status,
            bucketId = bucketId
        )
            .collection("clients")
            .document(clientId)
    }

    fun insert(
        ownerId: String,
        yearMonth: String,
        bucketId: String,
        index: SummaryClientIndex
    ): SummaryClientIndex {

        val reference =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = index.status,
                bucketId = bucketId,
                clientId = index.clientId
            )

        reference
            .set(toFirestoreMap(index))
            .get()

        return index
    }

    fun insertInTransaction(
        transaction: Transaction,
        ownerId: String,
        yearMonth: String,
        bucketId: String,
        index: SummaryClientIndex
    ): SummaryClientIndex {

        val reference =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = index.status,
                bucketId = bucketId,
                clientId = index.clientId
            )

        transaction.set(
            reference,
            toFirestoreMap(index)
        )

        return index
    }

    fun find(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String,
        clientId: String
    ): SummaryClientIndex? {

        val reference =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status,
                bucketId = bucketId,
                clientId = clientId
            )

        val snapshot =
            reference
                .get()
                .get()

        if (!snapshot.exists()) {
            return null
        }

        return toIndex(snapshot)
    }

    /**
     * Finds the index in the exact bucket supplied by the client.
     *
     * No bucket scan is required because Client.bucketId is the
     * source of truth for the current architecture.
     */
    fun findInTransaction(
        transaction: Transaction,
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String,
        clientId: String
    ): SummaryClientIndexLocation? {

        val clientRef =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status,
                bucketId = bucketId,
                clientId = clientId
            )

        val bucketRef =
            bucketReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status,
                bucketId = bucketId
            )

        val clientSnapshot =
            transaction
                .get(clientRef)
                .get()

        if (!clientSnapshot.exists()) {
            return null
        }

        val bucketSnapshot =
            transaction
                .get(bucketRef)
                .get()

        val bucketSize =
            bucketSnapshot
                .getLong("size")
                ?: 0L

        return SummaryClientIndexLocation(
            bucketId = bucketId,
            index = toIndex(clientSnapshot),
            bucketExists = bucketSnapshot.exists(),
            bucketSize = bucketSize
        )
    }

    /**
     * Reads the target bucket and target client index.
     *
     * This method is READ / PLAN only.
     * It performs no writes.
     */
    fun planUpsertInTransaction(
        transaction: Transaction,
        ownerId: String,
        yearMonth: String,
        bucketId: String,
        index: SummaryClientIndex,
        capacity: Long
    ): SummaryClientIndexUpsertPlan {

        require(capacity > 0) {
            "capacity must be greater than zero"
        }

        val clientRef =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = index.status,
                bucketId = bucketId,
                clientId = index.clientId
            )

        val bucketRef =
            bucketReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = index.status,
                bucketId = bucketId
            )

        val clientSnapshot =
            transaction
                .get(clientRef)
                .get()

        val bucketSnapshot =
            transaction
                .get(bucketRef)
                .get()

        val currentBucketSize =
            bucketSnapshot
                .getLong("size")
                ?: 0L

        return SummaryClientIndexUpsertPlan(
            clientReference = clientRef,
            bucketReference = bucketRef,
            index = index,
            bucketExists = bucketSnapshot.exists(),
            currentBucketSize = currentBucketSize,
            capacity = capacity
        )
    }

    /**
     * WRITE / APPLY for a planned upsert.
     *
     * Existing index:
     *   update only the index document.
     *
     * Missing index:
     *   create the index and increment/create bucket metadata.
     */
    fun applyUpsertPlanInTransaction(
        transaction: Transaction,
        plan: SummaryClientIndexUpsertPlan
    ): SummaryClientIndex {

        if (!plan.bucketExists) {

            transaction.set(
                plan.bucketReference,
                mapOf(
                    "capacity" to plan.capacity,
                    "size" to 1L
                )
            )

        } else if (plan.currentBucketSize <= plan.capacity) {

            /*
             * If the index already exists, the bucket size must not
             * increase. The caller uses the existence information from
             * the READ / PLAN phase.
             *
             * This method is intended for the create/missing-index case.
             */
        }

        transaction.set(
            plan.clientReference,
            toFirestoreMap(plan.index)
        )

        return plan.index
    }

    /**
     * Creates a plan for removing an index.
     *
     * READ / PLAN only.
     */
    fun planDeleteInTransaction(
        transaction: Transaction,
        location: SummaryClientIndexLocation,
        ownerId: String,
        yearMonth: String,
        status: ClientType
    ): SummaryClientIndexDeletePlan {

        val clientRef =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status,
                bucketId = location.bucketId,
                clientId = location.index.clientId
            )

        val bucketRef =
            bucketReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status,
                bucketId = location.bucketId
            )

        return SummaryClientIndexDeletePlan(
            clientReference = clientRef,
            bucketReference = bucketRef,
            currentBucketSize = location.bucketSize
        )
    }

    /**
     * WRITE / APPLY for a planned deletion.
     *
     * Removes the client index and releases its bucket capacity.
     */
    fun applyDeletePlanInTransaction(
        transaction: Transaction,
        plan: SummaryClientIndexDeletePlan
    ) {

        transaction.delete(
            plan.clientReference
        )

        val newSize =
            (plan.currentBucketSize - 1L)
                .coerceAtLeast(0L)

        transaction.update(
            plan.bucketReference,
            "size",
            newSize
        )
    }

    fun findPage(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        size: Int,
        cursor: String?
    ): PageResult<SummaryClientIndex> {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }

        require(size > 0) {
            "size must be greater than zero"
        }

        val pageCursor =
            decodeCursor(cursor)

        var currentBucketId =
            pageCursor?.bucketId

        var currentClientId =
            pageCursor?.clientId

        val result =
            mutableListOf<SummaryClientIndex>()

        while (result.size < size) {

            val bucketId =
                currentBucketId
                    ?: findFirstBucketId(
                        ownerId = ownerId,
                        yearMonth = yearMonth,
                        status = status
                    )

            if (bucketId == null) {
                break
            }

            val remaining =
                size - result.size

            val clientCollection =
                bucketCollection(
                    ownerId = ownerId,
                    yearMonth = yearMonth,
                    status = status
                )
                    .document(bucketId)
                    .collection("clients")

            val clientQuery =
                clientCollection
                    .orderBy("__name__")
                    .let { query ->

                        if (
                            currentBucketId == bucketId &&
                            currentClientId != null
                        ) {
                            query.startAfter(currentClientId)
                        } else {
                            query
                        }
                    }
                    .limit(remaining + 1)

            val clientSnapshots =
                clientQuery
                    .get()
                    .get()
                    .documents

            val hasMoreInCurrentBucket =
                clientSnapshots.size > remaining

            val documentsToReturn =
                if (hasMoreInCurrentBucket) {
                    clientSnapshots.take(remaining)
                } else {
                    clientSnapshots
                }

            result.addAll(
                documentsToReturn.map(::toIndex)
            )

            if (hasMoreInCurrentBucket) {

                val lastDocument =
                    documentsToReturn.last()

                return PageResult(
                    content = result,
                    hasNext = true,
                    nextCursor =
                    encodeCursor(
                        PageCursor(
                            bucketId = bucketId,
                            clientId = lastDocument.id
                        )
                    )
                )
            }

            val nextBucketId =
                findNextBucketId(
                    ownerId = ownerId,
                    yearMonth = yearMonth,
                    status = status,
                    bucketId = bucketId
                )

            if (result.size == size) {

                val lastDocument =
                    documentsToReturn.lastOrNull()

                if (
                    nextBucketId != null &&
                    lastDocument != null
                ) {

                    return PageResult(
                        content = result,
                        hasNext = true,
                        nextCursor =
                        encodeCursor(
                            PageCursor(
                                bucketId = bucketId,
                                clientId = lastDocument.id
                            )
                        )
                    )
                }

                return PageResult(
                    content = result,
                    hasNext = false,
                    nextCursor = null
                )
            }

            if (nextBucketId == null) {
                break
            }

            currentBucketId =
                nextBucketId

            currentClientId =
                null
        }

        return PageResult(
            content = result,
            hasNext = false,
            nextCursor = null
        )
    }

    private fun findFirstBucketId(
        ownerId: String,
        yearMonth: String,
        status: ClientType
    ): String? {

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status
        )
            .orderBy("__name__")
            .limit(1)
            .get()
            .get()
            .documents
            .firstOrNull()
            ?.id
    }

    private fun findNextBucketId(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String
    ): String? {

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth,
            status = status
        )
            .orderBy("__name__")
            .startAfter(bucketId)
            .limit(1)
            .get()
            .get()
            .documents
            .firstOrNull()
            ?.id
    }

    private fun encodeCursor(
        cursor: PageCursor
    ): String {

        val raw =
            "${cursor.bucketId}|${cursor.clientId}"

        return Base64
            .getUrlEncoder()
            .withoutPadding()
            .encodeToString(
                raw.toByteArray()
            )
    }

    private fun decodeCursor(
        cursor: String?
    ): PageCursor? {

        if (cursor.isNullOrBlank()) {
            return null
        }

        return try {

            val decoded =
                String(
                    Base64
                        .getUrlDecoder()
                        .decode(cursor)
                )

            val parts =
                decoded.split("|")

            require(parts.size == 2)
            require(parts[0].isNotBlank())
            require(parts[1].isNotBlank())

            PageCursor(
                bucketId = parts[0],
                clientId = parts[1]
            )

        } catch (exception: Exception) {

            throw IllegalArgumentException(
                "Invalid cursor",
                exception
            )
        }
    }

    private fun toFirestoreMap(
        index: SummaryClientIndex
    ): Map<String, Any> {

        return mapOf(
            "clientId" to index.clientId,
            "amount" to index.amount,
            "status" to index.status.name
        )
    }

    private fun toIndex(
        snapshot: DocumentSnapshot
    ): SummaryClientIndex {

        return SummaryClientIndex(
            clientId =
            snapshot.getString("clientId")
                ?: snapshot.id,

            amount =
            snapshot.getLong("amount")
                ?: 0,

            status =
            snapshot.getString("status")
                ?.let {
                    ClientType.valueOf(it)
                }
                ?: ClientType.SETTLED
        )
    }

    fun findAllInBucket(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String
    ): List<SummaryClientIndex> {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        val clientCollection =
            bucketCollection(
                ownerId = ownerId,
                yearMonth = yearMonth,
                status = status
            )
                .document(bucketId)
                .collection("clients")

        return clientCollection
            .orderBy("__name__")
            .get()
            .get()
            .documents
            .map(::toIndex)
    }

    fun copyInTransaction(
        transaction: Transaction,
        ownerId: String,
        newYearMonth: String,
        status: ClientType,
        bucketId: String,
        index: SummaryClientIndex
    ): SummaryClientIndex {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(newYearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "newYearMonth must be in yyyy-MM format"
        }

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        require(index.clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        require(index.status == status) {
            "Index status does not match bucket status"
        }

        val targetReference =
            bucketCollection(
                ownerId = ownerId,
                yearMonth = newYearMonth,
                status = status
            )
                .document(bucketId)
                .collection("clients")
                .document(index.clientId)

        transaction.set(
            targetReference,
            toFirestoreMap(index)
        )

        return index
    }
}
