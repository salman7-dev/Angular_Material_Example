package com.clientledger.core.repository.summary

import com.clientledger.core.config.ClientLedgerProperties
import com.google.cloud.firestore.CollectionReference
import com.google.cloud.firestore.DocumentReference
import com.google.cloud.firestore.DocumentSnapshot
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Repository

data class SummaryClientIndexBucketAllocationPlan(
    val bucketId: String,
    val bucketReference: DocumentReference,
    val currentSize: Long,
    val isNewBucket: Boolean
)

@Repository
class SummaryClientIndexBucketRepository(
    private val firestore: Firestore,
    private val properties: ClientLedgerProperties
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

    fun allocateBucket(
        ownerId: String,
        yearMonth: String
    ): String {

        val capacity =
            properties.summary.indexBucketCapacity

        require(capacity > 0) {
            "summary.index-bucket-capacity must be greater than zero"
        }

        val collection =
            bucketCollection(
                ownerId = ownerId,
                yearMonth = yearMonth
            )

        return firestore.runTransaction { transaction ->

            val buckets = transaction
                .get(
                    collection.orderBy("__name__")
                )
                .get()
                .documents

            allocateFromSnapshots(
                transaction = transaction,
                collection = collection,
                snapshots = buckets,
                capacity = capacity
            )
        }.get()
    }

    fun allocateBucketInTransaction(
        transaction: Transaction,
        ownerId: String,
        yearMonth: String
    ): String {

        val capacity =
            properties.summary.indexBucketCapacity

        require(capacity > 0) {
            "summary.index-bucket-capacity must be greater than zero"
        }

        val collection =
            bucketCollection(
                ownerId = ownerId,
                yearMonth = yearMonth
            )

        val buckets = transaction
            .get(
                collection.orderBy("__name__")
            )
            .get()
            .documents

        return allocateFromSnapshots(
            transaction = transaction,
            collection = collection,
            snapshots = buckets,
            capacity = capacity
        )
    }

    private fun allocateFromSnapshots(
        transaction: Transaction,
        collection: CollectionReference,
        snapshots: List<DocumentSnapshot>,
        capacity: Int
    ): String {

        val availableBucket = snapshots.firstOrNull { snapshot ->
            val size = snapshot.getLong("size") ?: 0
            size < capacity
        }

        if (availableBucket != null) {

            val currentSize =
                availableBucket.getLong("size") ?: 0

            transaction.update(
                availableBucket.reference,
                "size",
                currentSize + 1
            )

            return availableBucket.id
        }

        val bucketId =
            nextBucketId(snapshots)

        val bucketReference =
            collection.document(bucketId)

        transaction.set(
            bucketReference,
            mapOf(
                "capacity" to capacity,
                "size" to 1L
            )
        )

        return bucketId
    }

    fun find(
        ownerId: String,
        yearMonth: String,
        bucketId: String
    ): Map<String, Long>? {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        val reference =
            bucketCollection(
                ownerId = ownerId,
                yearMonth = yearMonth
            ).document(bucketId)

        val snapshot =
            reference
                .get()
                .get()

        if (!snapshot.exists()) {
            return null
        }

        return mapOf(
            "capacity" to (snapshot.getLong("capacity") ?: 0),
            "size" to (snapshot.getLong("size") ?: 0)
        )
    }

    fun planBucketAllocationInTransaction(
        transaction: Transaction,
        ownerId: String,
        yearMonth: String
    ): SummaryClientIndexBucketAllocationPlan {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }

        val capacity =
            properties.summary.indexBucketCapacity

        require(capacity > 0) {
            "summary.index-bucket-capacity must be greater than zero"
        }

        val collection =
            bucketCollection(
                ownerId = ownerId,
                yearMonth = yearMonth
            )

        val snapshots = transaction
            .get(
                collection.orderBy("__name__")
            )
            .get()
            .documents

        val availableBucket = snapshots.firstOrNull { snapshot ->
            val size = snapshot.getLong("size") ?: 0
            size < capacity
        }

        if (availableBucket != null) {

            return SummaryClientIndexBucketAllocationPlan(
                bucketId = availableBucket.id,
                bucketReference = availableBucket.reference,
                currentSize =
                availableBucket.getLong("size") ?: 0,
                isNewBucket = false
            )
        }

        val bucketId =
            nextBucketId(snapshots)

        return SummaryClientIndexBucketAllocationPlan(
            bucketId = bucketId,
            bucketReference = collection.document(bucketId),
            currentSize = 0,
            isNewBucket = true
        )
    }

    fun applyBucketAllocationInTransaction(
        transaction: Transaction,
        plan: SummaryClientIndexBucketAllocationPlan
    ) {

        val newSize =
            plan.currentSize + 1

        if (plan.isNewBucket) {

            transaction.set(
                plan.bucketReference,
                mapOf(
                    "capacity" to properties.summary.indexBucketCapacity,
                    "size" to newSize
                )
            )

            return
        }

        transaction.update(
            plan.bucketReference,
            "size",
            newSize
        )
    }

    fun removeClientFromBucketInTransaction(
        transaction: Transaction,
        ownerId: String,
        yearMonth: String,
        bucketId: String
    ) {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        val bucketReference =
            bucketCollection(
                ownerId = ownerId,
                yearMonth = yearMonth
            ).document(bucketId)

        val snapshot =
            transaction
                .get(bucketReference)
                .get()

        require(snapshot.exists()) {
            "Summary index bucket does not exist: " +
                    "$ownerId/$yearMonth/$bucketId"
        }

        val currentSize =
            snapshot.getLong("size") ?: 0

        require(currentSize > 0) {
            "Summary index bucket size cannot be negative: " +
                    "$ownerId/$yearMonth/$bucketId"
        }

        transaction.update(
            bucketReference,
            "size",
            currentSize - 1
        )
    }

    private fun nextBucketId(
        documents: List<DocumentSnapshot>
    ): String {

        val nextNumber = documents
            .mapNotNull { snapshot ->
                snapshot.id
                    .removePrefix("bucket_")
                    .toIntOrNull()
            }
            .maxOrNull()
            ?.plus(1)
            ?: 0

        return "bucket_${nextNumber.toString().padStart(3, '0')}"
    }

    fun findBucketIds(
        ownerId: String,
        yearMonth: String
    ): List<String> {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(yearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "yearMonth must be in yyyy-MM format"
        }

        return bucketCollection(
            ownerId = ownerId,
            yearMonth = yearMonth
        )
            .orderBy("__name__")
            .get()
            .get()
            .documents
            .map { it.id }
    }

    fun copyBucketInTransaction(
        transaction: Transaction,
        ownerId: String,
        previousYearMonth: String,
        newYearMonth: String,
        bucketId: String
    ) {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(previousYearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "previousYearMonth must be in yyyy-MM format"
        }

        require(newYearMonth.matches(Regex("\\d{4}-\\d{2}"))) {
            "newYearMonth must be in yyyy-MM format"
        }

        require(bucketId.isNotBlank()) {
            "bucketId must not be blank"
        }

        val sourceReference =
            bucketCollection(
                ownerId = ownerId,
                yearMonth = previousYearMonth
            ).document(bucketId)

        val targetReference =
            bucketCollection(
                ownerId = ownerId,
                yearMonth = newYearMonth
            ).document(bucketId)

        val sourceSnapshot =
            transaction
                .get(sourceReference)
                .get()

        require(sourceSnapshot.exists()) {
            "Summary index bucket does not exist: " +
                    "$ownerId/$previousYearMonth/$bucketId"
        }

        val capacity =
            sourceSnapshot.getLong("capacity") ?: 0

        val size =
            sourceSnapshot.getLong("size") ?: 0

        transaction.set(
            targetReference,
            mapOf(
                "capacity" to capacity,
                "size" to size
            )
        )
    }
}

