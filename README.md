package com.clientledger.core.repository.month

import com.clientledger.core.domain.MonthMetadata
import com.clientledger.core.domain.MonthInitializationStatus
import com.google.cloud.firestore.Firestore
import org.springframework.stereotype.Repository

@Repository
class MonthMetadataRepository(
    private val firestore: Firestore
) {

    companion object {
        private const val COLLECTION = "month_metadata"
    }

    fun save(metadata: MonthMetadata) {
        firestore
            .collection(COLLECTION)
            .document(metadata.yearMonth)
            .set(metadata)
            .get()
    }

    fun find(yearMonth: String): MonthMetadata? {
        val snapshot = firestore
            .collection(COLLECTION)
            .document(yearMonth)
            .get()
            .get()

        if (!snapshot.exists()) {
            return null
        }

        return snapshot.toObject(MonthMetadata::class.java)
    }

    fun updateStatus(
        yearMonth: String,
        status: MonthInitializationStatus
    ) {
        val document = firestore
            .collection(COLLECTION)
            .document(yearMonth)

        document.update("status", status).get()
    }
}
