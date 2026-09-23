package com.clientledger.core.repository.client

import com.clientledger.core.domain.Address
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientType
import com.clientledger.core.service.CurrentOwnerProvider
import com.google.cloud.firestore.Firestore
import org.springframework.stereotype.Repository
import java.time.Instant
import java.util.Date

@Repository
class ClientRepository(
    private val firestore: Firestore,
    private val currentOwnerProvider: CurrentOwnerProvider
) {

    private val ownersCollection = firestore.collection("owners")

    private fun clientsCollection() =
        ownersCollection
            .document(currentOwnerProvider.getOwnerId())
            .collection("clients")

    fun save(client: Client) {
        clientsCollection()
            .document(client.id)
            .set(client)
            .get()
    }

    fun findById(clientId: String): Client? {
        val snapshot = clientsCollection()
            .document(clientId)
            .get()
            .get()

        if (!snapshot.exists()) {
            return null
        }

        return fromFirestore(snapshot.data ?: return null)
    }

    fun delete(clientId: String) {
        clientsCollection()
            .document(clientId)
            .delete()
            .get()
    }

    fun count(): Long {
        return clientsCollection()
            .get()
            .get()
            .size()
            .toLong()
    }

    private fun fromFirestore(data: Map<String, Any?>): Client {

        val addressData = data["address"] as? Map<*, *>

        return Client(
            id = data["id"] as? String ?: "",
            ownerId = data["ownerId"] as? String ?: "",
            name = data["name"] as? String ?: "",
            phone = data["phone"] as? String ?: "",
            email = data["email"] as? String ?: "",
            gstNumber = data["gstNumber"] as? String,
            address = addressData?.let {
                Address(
                    line1 = it["line1"] as? String ?: "",
                    line2 = it["line2"] as? String,
                    city = it["city"] as? String ?: "",
                    state = it["state"] as? String ?: "",
                    pinCode = it["pinCode"] as? String ?: "",
                    country = it["country"] as? String ?: "India"
                )
            },
            initialOpeningBalance =
            (data["initialOpeningBalance"] as? Number)?.toLong() ?: 0,
            latestAmount =
            (data["latestAmount"] as? Number)?.toLong() ?: 0,
            type =
            (data["type"] as? String)
                ?.let { runCatching { ClientType.valueOf(it) }.getOrNull() }
                ?: ClientType.SETTLED,
            bucketId = data["bucketId"] as? String ?: "",
            createdAt =
            (data["createdAt"] as? Date)?.toInstant() ?: Instant.now(),
            updatedAt =
            (data["updatedAt"] as? Date)?.toInstant() ?: Instant.now()
        )
    }
}


package com.clientledger.core.repository.order

import com.clientledger.core.domain.Order
import com.clientledger.core.service.CurrentOwnerProvider
import com.google.cloud.firestore.Firestore
import org.springframework.stereotype.Repository

@Repository
class OrderRepository(
    private val firestore: Firestore,
    private val currentOwnerProvider: CurrentOwnerProvider
) {

    private val transactionsCollection = "transactions"
    private val itemsCollection = "items"

    private fun itemsCollection(year: String, month: String) =
        firestore
            .collection("owners")
            .document(currentOwnerProvider.getOwnerId())
            .collection(transactionsCollection)
            .document(year)
            .collection(month)
            .document(itemsCollection)

    fun save(order: Order) {
        val year = order.orderDate.substring(0, 4)
        val month = order.orderDate.substring(5, 7)

        itemsCollection(year, month)
            .document(order.id)
            .set(order)
            .get()
    }

    fun findById(
        orderId: String,
        year: String,
        month: String
    ): Order? {
        val snapshot = itemsCollection(year, month)
            .document(orderId)
            .get()
            .get()

        if (!snapshot.exists()) {
            return null
        }

        return snapshot.toObject(Order::class.java)
    }

    fun findAllForMonth(
        year: String,
        month: String,
        clientId: String
    ): List<Order> {
        val querySnapshot = itemsCollection(year, month)
            .whereEqualTo("clientId", clientId)
            .get()
            .get()

        return querySnapshot.getDocuments()
            .mapNotNull { document ->
                document.toObject(Order::class.java)
            }
    }

    fun delete(
        orderId: String,
        year: String,
        month: String
    ) {
        itemsCollection(year, month)
            .document(orderId)
            .delete()
            .get()
    }
}

