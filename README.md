package com.clientledger.core.repository.summary

import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.SummaryClientIndex
import com.clientledger.core.pagination.PageCursor
import com.clientledger.core.pagination.PageResult
import com.google.cloud.firestore.CollectionReference
import com.google.cloud.firestore.DocumentReference
import com.google.cloud.firestore.DocumentSnapshot
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Repository
import java.util.Base64

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
    val indexExists: Boolean,
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

    private fun bucketCollection(
        ownerId: String,
        yearMonth: String
    ): CollectionReference {

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
            .collection("buckets")
    }

    private fun bucketReference(
        ownerId: String,
        yearMonth: String,
        bucketId: String
    ): DocumentReference {

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth
        ).document(bucketId)
    }

    private fun clientReference(
        ownerId: String,
        yearMonth: String,
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
        bucketId: String,
        clientId: String
    ): SummaryClientIndex? {

        val reference =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
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
     * Finds a client index directly in the client's physical bucket.
     *
     * No status bucket scan is required.
     */
    fun findInTransaction(
        transaction: Transaction,
        ownerId: String,
        yearMonth: String,
        bucketId: String,
        clientId: String
    ): SummaryClientIndexLocation? {

        val clientRef =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                bucketId = bucketId,
                clientId = clientId
            )

        val bucketRef =
            bucketReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
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

        return SummaryClientIndexLocation(
            bucketId = bucketId,
            index = toIndex(clientSnapshot),
            bucketExists = bucketSnapshot.exists(),
            bucketSize =
            bucketSnapshot.getLong("size") ?: 0L
        )
    }

    /**
     * READ / PLAN only.
     *
     * Determines whether the target index already exists and reads
     * the current bucket metadata.
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
                bucketId = bucketId,
                clientId = index.clientId
            )

        val bucketRef =
            bucketReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
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

        return SummaryClientIndexUpsertPlan(
            clientReference = clientRef,
            bucketReference = bucketRef,
            index = index,
            indexExists = clientSnapshot.exists(),
            bucketExists = bucketSnapshot.exists(),
            currentBucketSize =
            bucketSnapshot.getLong("size") ?: 0L,
            capacity = capacity
        )
    }

    /**
     * WRITE / APPLY.
     *
     * Existing index:
     *   update index only.
     *   Bucket size remains unchanged.
     *
     * Missing index:
     *   create index.
     *   Bucket size increases by one.
     */
    fun applyUpsertPlanInTransaction(
        transaction: Transaction,
        plan: SummaryClientIndexUpsertPlan
    ): SummaryClientIndex {

        if (plan.indexExists) {

            transaction.set(
                plan.clientReference,
                toFirestoreMap(plan.index)
            )

            return plan.index
        }

        require(
            !plan.bucketExists ||
                    plan.currentBucketSize < plan.capacity
        ) {
            "Summary index bucket is full: ${plan.bucketReference.path}"
        }

        if (!plan.bucketExists) {

            transaction.set(
                plan.bucketReference,
                mapOf(
                    "capacity" to plan.capacity,
                    "size" to 1L
                )
            )

        } else {

            transaction.update(
                plan.bucketReference,
                "size",
                plan.currentBucketSize + 1L
            )
        }

        transaction.set(
            plan.clientReference,
            toFirestoreMap(plan.index)
        )

        return plan.index
    }

    /**
     * READ / PLAN only.
     */
    fun planDeleteInTransaction(
        transaction: Transaction,
        location: SummaryClientIndexLocation,
        ownerId: String,
        yearMonth: String
    ): SummaryClientIndexDeletePlan {

        val clientRef =
            clientReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                bucketId = location.bucketId,
                clientId = location.index.clientId
            )

        val bucketRef =
            bucketReference(
                ownerId = ownerId,
                yearMonth = yearMonth,
                bucketId = location.bucketId
            )

        return SummaryClientIndexDeletePlan(
            clientReference = clientRef,
            bucketReference = bucketRef,
            currentBucketSize = location.bucketSize
        )
    }

    /**
     * WRITE / APPLY.
     *
     * Removes the client index and releases one bucket slot.
     */
    fun applyDeletePlanInTransaction(
        transaction: Transaction,
        plan: SummaryClientIndexDeletePlan
    ) {

        transaction.delete(
            plan.clientReference
        )

        require(plan.currentBucketSize > 0) {
            "Summary index bucket size cannot be negative: " +
                    plan.bucketReference.path
        }

        transaction.update(
            plan.bucketReference,
            "size",
            plan.currentBucketSize - 1L
        )
    }

    fun findPage(
        ownerId: String,
        yearMonth: String,
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
                        yearMonth = yearMonth
                    )

            if (bucketId == null) {
                break
            }

            val remaining =
                size - result.size

            val clientCollection =
                bucketCollection(
                    ownerId = ownerId,
                    yearMonth = yearMonth
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

            val snapshots =
                clientQuery
                    .get()
                    .get()
                    .documents

            val hasMoreInCurrentBucket =
                snapshots.size > remaining

            val documentsToReturn =
                if (hasMoreInCurrentBucket) {
                    snapshots.take(remaining)
                } else {
                    snapshots
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

            currentBucketId = nextBucketId
            currentClientId = null
        }

        return PageResult(
            content = result,
            hasNext = false,
            nextCursor = null
        )
    }

    private fun findFirstBucketId(
        ownerId: String,
        yearMonth: String
    ): String? {

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth
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
        bucketId: String
    ): String? {

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth
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

    fun findAllInBucket(
        ownerId: String,
        yearMonth: String,
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

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth
        )
            .document(bucketId)
            .collection("clients")
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

        val targetReference =
            clientReference(
                ownerId = ownerId,
                yearMonth = newYearMonth,
                bucketId = bucketId,
                clientId = index.clientId
            )

        transaction.set(
            targetReference,
            toFirestoreMap(index)
        )

        return index
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
                ?: 0L,

            status =
            snapshot.getString("status")
                ?.let(ClientType::valueOf)
                ?: ClientType.SETTLED
        )
    }
}
