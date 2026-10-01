package com.clientledger.core.repository.client

import com.clientledger.core.domain.Address
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientType
import com.clientledger.core.pagination.PageResult
import com.google.cloud.Timestamp
import com.google.cloud.firestore.DocumentSnapshot
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Repository
import java.time.Instant
import java.util.Base64
import java.util.Date

@Repository
class ClientRepository(
    private val firestore: Firestore
) {

    private fun clientCollection(ownerId: String) =
        firestore
            .collection("owners")
            .document(ownerId)
            .collection("clients")

    fun create(client: Client): Client {
        require(client.ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(client.id.isNotBlank()) {
            "client id must not be blank"
        }

        clientCollection(client.ownerId)
            .document(client.id)
            .set(
                client.copy(
                    createdAt = client.createdAt,
                    updatedAt = client.updatedAt
                )
            )
            .get()

        return client
    }

    fun findById(
        ownerId: String,
        clientId: String
    ): Client? {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        val snapshot =
            clientCollection(ownerId)
                .document(clientId)
                .get()
                .get()

        if (!snapshot.exists()) {
            return null
        }

        return toClient(snapshot)
    }

    fun findPage(
        ownerId: String,
        size: Int,
        cursor: String?
    ): PageResult<Client> {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(size > 0) {
            "size must be greater than zero"
        }

        val clientCollection =
            clientCollection(ownerId)

        val query =
            clientCollection
                .orderBy("__name__")
                .let { baseQuery ->

                    val clientId =
                        decodeCursor(cursor)

                    if (clientId != null) {
                        baseQuery.startAfter(clientId)
                    } else {
                        baseQuery
                    }
                }
                .limit(size + 1)

        val documents =
            query
                .get()
                .get()
                .documents

        val hasNext =
            documents.size > size

        val clients =
            if (hasNext) {
                documents
                    .take(size)
                    .map(::toClient)
            } else {
                documents
                    .map(::toClient)
            }

        if (!hasNext) {
            return PageResult(
                content = clients,
                hasNext = false,
                nextCursor = null
            )
        }

        val lastClient =
            clients.last()

        return PageResult(
            content = clients,
            hasNext = true,
            nextCursor = encodeCursor(lastClient.id)
        )
    }

    fun exists(
        ownerId: String,
        clientId: String
    ): Boolean {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        return clientCollection(ownerId)
            .document(clientId)
            .get()
            .get()
            .exists()
    }

    private fun encodeCursor(
        clientId: String
    ): String {

        require(clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        val raw =
            "client:$clientId"

        return Base64
            .getUrlEncoder()
            .withoutPadding()
            .encodeToString(
                raw.toByteArray()
            )
    }

    private fun decodeCursor(
        cursor: String?
    ): String? {

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

            require(
                decoded.startsWith("client:")
            )

            val clientId =
                decoded.removePrefix("client:")

            require(
                clientId.isNotBlank()
            )

            clientId

        } catch (exception: Exception) {

            throw IllegalArgumentException(
                "Invalid cursor",
                exception
            )
        }
    }



    private fun toClient(
        snapshot: DocumentSnapshot
    ): Client {

        return Client(
            id =
            snapshot.getString("id")
                ?: "",

            ownerId =
            snapshot.getString("ownerId")
                ?: "",

            name =
            snapshot.getString("name")
                ?: "",

            phone =
            snapshot.getString("phone")
                ?: "",

            email =
            snapshot.getString("email")
                ?: "",

            gstNumber =
            snapshot.getString("gstNumber"),

            address =
            toAddress(
                snapshot.get("address")
            ),

            initialOpeningBalance =
            snapshot.getLong("initialOpeningBalance")
                ?: 0,

            latestAmount =
            snapshot.getLong("latestAmount")
                ?: 0,

            type =
            snapshot.getString("type")
                ?.let { ClientType.valueOf(it) }
                ?: ClientType.SETTLED,

            bucketId =
            snapshot.getString("bucketId")
                ?: "",

            createdAt =
            toInstant(
                snapshot.get("createdAt")
            ),

            updatedAt =
            toInstant(
                snapshot.get("updatedAt")
            )
        )
    }

    private fun toAddress(
        value: Any?
    ): Address? {

        val data =
            value as? Map<*, *>
                ?: return null

        return Address(
            line1 =
            data["line1"] as? String
                ?: "",

            line2 =
            data["line2"] as? String,

            city =
            data["city"] as? String
                ?: "",

            state =
            data["state"] as? String
                ?: "",

            pinCode =
            data["pinCode"] as? String
                ?: "",

            country =
            data["country"] as? String
                ?: "India"
        )
    }

    private fun toInstant(
        value: Any?
    ): Instant {

        return when (value) {

            is Instant ->
                value

            is Date ->
                value.toInstant()

            is Timestamp ->
                value.toDate().toInstant()

            is Map<*, *> -> {

                val seconds =
                    (value["seconds"] as? Number)
                        ?.toLong()
                        ?: return Instant.EPOCH

                val nanos =
                    (value["nanos"] as? Number)
                        ?.toInt()
                        ?: 0

                Instant.ofEpochSecond(
                    seconds,
                    nanos.toLong()
                )
            }

            else ->
                Instant.EPOCH
        }
    }

    fun existsInTransaction(
        transaction: Transaction,
        ownerId: String,
        clientId: String
    ): Boolean {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        return transaction
            .get(
                clientCollection(ownerId)
                    .document(clientId)
            )
            .get()
            .exists()
    }

    fun createInTransaction(
        transaction: Transaction,
        client: Client
    ): Client {

        require(client.ownerId.isNotBlank()) {
            "client ownerId must not be blank"
        }

        require(client.id.isNotBlank()) {
            "client id must not be blank"
        }

        transaction.set(
            clientCollection(client.ownerId)
                .document(client.id),
            client.copy(
                createdAt = client.createdAt,
                updatedAt = client.updatedAt
            )
        )

        return client
    }



}
