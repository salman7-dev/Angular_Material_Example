package com.clientledger.core.repository.owner

import com.clientledger.core.domain.Address
import com.clientledger.core.domain.owner.Owner
import com.google.cloud.firestore.DocumentReference
import com.google.cloud.firestore.DocumentSnapshot
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Repository
import java.time.Instant

@Repository
class OwnerRepository(
    private val firestore: Firestore
) {

    private fun ownerReference(
        ownerId: String
    ): DocumentReference {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        return firestore
            .collection("owners")
            .document(ownerId)
    }

    fun exists(
        ownerId: String
    ): Boolean {

        return ownerReference(ownerId)
            .get()
            .get()
            .exists()
    }

    fun existsInTransaction(
        transaction: Transaction,
        ownerId: String
    ): Boolean {

        return transaction
            .get(
                ownerReference(ownerId)
            )
            .get()
            .exists()
    }

    fun create(
        owner: Owner
    ): Owner {

        require(owner.ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        ownerReference(owner.ownerId)
            .set(toFirestoreMap(owner))
            .get()

        return owner
    }

    fun createInTransaction(
        transaction: Transaction,
        owner: Owner
    ): Owner {

        require(owner.ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        transaction.set(
            ownerReference(owner.ownerId),
            toFirestoreMap(owner)
        )

        return owner
    }

    fun find(
        ownerId: String
    ): Owner? {

        val snapshot =
            ownerReference(ownerId)
                .get()
                .get()

        if (!snapshot.exists()) {
            return null
        }

        return toOwner(snapshot)
    }

    private fun toFirestoreMap(
        owner: Owner
    ): Map<String, Any?> {

        return mapOf(
            "ownerId" to owner.ownerId,
            "name" to owner.name,
            "businessName" to owner.businessName,
            "phone" to owner.phone,
            "email" to owner.email,
            "gstNumber" to owner.gstNumber,
            "address" to mapOf(
                "line1" to owner.address.line1,
                "line2" to owner.address.line2,
                "city" to owner.address.city,
                "state" to owner.address.state,
                "pinCode" to owner.address.pinCode,
                "country" to owner.address.country
            ),
            "createdAt" to owner.createdAt.toString(),
            "updatedAt" to owner.updatedAt.toString()
        )
    }

    private fun toOwner(
        snapshot: DocumentSnapshot
    ): Owner {

        val addressMap =
            snapshot.get("address") as? Map<*, *>

        val address =
            Address(
                line1 =
                addressMap?.get("line1") as? String ?: "",

                line2 =
                addressMap?.get("line2") as? String,

                city =
                addressMap?.get("city") as? String ?: "",

                state =
                addressMap?.get("state") as? String ?: "",

                pinCode =
                addressMap?.get("pinCode") as? String ?: "",

                country =
                addressMap?.get("country") as? String ?: "India"
            )

        return Owner(
            ownerId =
            snapshot.getString("ownerId")
                ?: snapshot.id,

            name =
            snapshot.getString("name")
                ?: "",

            businessName =
            snapshot.getString("businessName")
                ?: "",

            phone =
            snapshot.getString("phone")
                ?: "",

            email =
            snapshot.getString("email")
                ?: "",

            gstNumber =
            snapshot.getString("gstNumber")
                ?: "",

            address = address,

            createdAt =
            snapshot.getString("createdAt")
                ?.let(Instant::parse)
                ?: Instant.EPOCH,

            updatedAt =
            snapshot.getString("updatedAt")
                ?.let(Instant::parse)
                ?: Instant.EPOCH
        )
    }

}
