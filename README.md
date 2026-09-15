package com.clientledger.core.repository.client

import com.clientledger.core.domain.client.Address
import com.clientledger.core.domain.client.Client
import com.google.cloud.firestore.DocumentSnapshot
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Repository
import java.time.Instant
import java.util.Date

@Repository
class FirestoreClientRepository(
    private val db: Firestore
) : ClientRepository {

    private val collection = db.collection("clients")

    private fun mapDocToClient(doc: DocumentSnapshot): Client {
        val addressMap = doc.get("address") as? Map<String, Any?>
        val address = Address(
            line1 = addressMap?.get("line1") as? String ?: "",
            line2 = addressMap?.get("line2") as? String,
            city = addressMap?.get("city") as? String ?: "",
            state = addressMap?.get("state") as? String ?: "",
            pincode = addressMap?.get("pincode") as? String ?: "",
            country = addressMap?.get("country") as? String ?: "India"
        )

        return Client(
            id = doc.getString("id") ?: doc.id,
            name = doc.getString("name") ?: "",
            phone = doc.getString("phone") ?: "",
            email = doc.getString("email") ?: "",
            initialOpeningBalance = doc.getLong("initialOpeningBalance") ?: 0L,
            lastClosingBalance = doc.getLong("lastClosingBalance") ?: 0L,
            gstNumber = doc.getString("gstNumber") ?: "",
            address = address,
            createdAt = doc.getDate("createdAt")?.toInstant() ?: Instant.now(),
            updatedAt = doc.getDate("updatedAt")?.toInstant() ?: Instant.now()
        )
    }

    private fun clientToMap(client: Client): Map<String, Any?> {
        return mapOf(
            "id" to client.id,
            "name" to client.name,
            "phone" to client.phone,
            "email" to client.email,
            "initialOpeningBalance" to client.initialOpeningBalance,
            "lastClosingBalance" to client.lastClosingBalance,
            "gstNumber" to client.gstNumber,
            "address" to mapOf(
                "line1" to client.address.line1,
                "line2" to client.address.line2,
                "city" to client.address.city,
                "state" to client.address.state,
                "pincode" to client.address.pincode,
                "country" to client.address.country
            ),
            "createdAt" to Date.from(client.createdAt),
            "updatedAt" to Date.from(client.updatedAt)
        )
    }

    override fun save(client: Client): Client {
        val docRef = collection.document(client.id)
        val existingDoc = docRef.get().get()

        // 🔥 Backend Intelligence: Check if order already exists
        val finalCreatedAt = if (existingDoc.exists()) {
            // Safely read createdAt as a Date/Timestamp, or fallback to Number/default
            val existingDate = existingDoc.getDate("createdAt")
            if (existingDate != null) {
                existingDate.toInstant()
            } else {
                val rawNumber = existingDoc.get("createdAt") as? Number
                if (rawNumber != null) {
                    Instant.ofEpochMilli(rawNumber.toLong())
                } else {
                    client.createdAt
                }
            }
        } else {
            // Agar naya order hai, toh current time do
            client.createdAt
        }


        // Updated client object with secured timestamps
        val securedClient = client.copy(
            createdAt = finalCreatedAt,
            updatedAt = Instant.now() // Update karte waqt updatedAt hamesha naya ho jayega
        )

        // Save to Firestore
        docRef.set(clientToMap(securedClient)).get()
        return securedClient
    }

    override fun findById(id: String): Client? {
        val documentSnapshot = collection.document(id).get().get()
        return if (documentSnapshot.exists()) {
            mapDocToClient(documentSnapshot)
        } else null
    }

    override fun findAll(): List<Client> {
        val querySnapshot = collection.get().get()
        return querySnapshot.documents.map { mapDocToClient(it) }
    }

    override fun deleteById(id: String) {
        collection.document(id).delete().get()
    }

    override fun findById(
        transaction: Transaction,
        clientId: String
    ): Client? {

        val docRef = collection.document(clientId)

        val documentSnapshot = transaction.get(docRef).get()

        return if (documentSnapshot.exists()) {
            mapDocToClient(documentSnapshot)
        } else {
            null
        }
    }

    override fun save(
        transaction: Transaction,
        client: Client,
        existingClient: Client?
    ) {

        val finalCreatedAt =
            existingClient?.createdAt
                ?: client.createdAt

        val securedClient =
            client.copy(
                createdAt = finalCreatedAt,
                updatedAt = Instant.now()
            )

        val docRef =
            collection.document(client.id)

        transaction.set(
            docRef,
            clientToMap(securedClient)
        )
    }
}



package com.clientledger.core.repository.snapshot

import com.clientledger.core.domain.snapshot.ClientBalanceSnapshot
import com.clientledger.core.utils.CommonUtils.MONTHS_TO_CHECK_IN_LEDGER_ENGINE
import com.google.cloud.firestore.DocumentSnapshot
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Query
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Repository
import java.time.Instant
import java.time.YearMonth
import java.time.temporal.ChronoUnit

@Repository
class FirestoreSnapshotRepository(
    private val db: Firestore
) : SnapshotRepository {


    override fun getInitialClientOpeningBalance(clientId: String): Long {
        val doc = db.collection("clients").document(clientId).get().get()
        if (!doc.exists()) return 0L
        return doc.getLong("initialOpeningBalance") ?: 0L
    }

    override fun getAllSnapshotsForClient(clientId: String): List<ClientBalanceSnapshot> {
        val snapshotsList = mutableListOf<ClientBalanceSnapshot>()
        val currentYearMonth = YearMonth.now()

        val cutoffYearMonth = currentYearMonth.minusMonths(MONTHS_TO_CHECK_IN_LEDGER_ENGINE)
//        println("CLIENT ID = $clientId")
//        println("CURRENT MONTH = $currentYearMonth")
//        println("CUTOFF MONTH = $cutoffYearMonth")
        try {

            val snapshotsRef = db.collection("clients")
                .document(clientId)
                .collection("snapshots")

            // Get all years covered by the date range
            val startYear = cutoffYearMonth.year
            val endYear = currentYearMonth.year

            for (year in startYear..endYear) {
                val monthsRef = snapshotsRef
                    .document(year.toString())
                    .collection("months")

                println("READING PATH = ${monthsRef.path}")
                val monthDocs = monthsRef.get().get()
                println("YEAR $year MONTHS = " + monthDocs.documents.map { it.id })

                for (monthDoc in monthDocs.documents) {

                    try {
                        val yearMonth = YearMonth.parse(monthDoc.id)

                        // Apply exact 24-month range filter
                        if (
                            !yearMonth.isBefore(cutoffYearMonth) &&
                            !yearMonth.isAfter(currentYearMonth)
                        ) {

                            val snapshot =
                                ClientBalanceSnapshot(
                                    clientId = clientId,
                                    yearMonth = yearMonth,
                                    openingBalance = monthDoc.getLong("openingBalance") ?: 0L,
                                    totalOrdersCount = monthDoc.getLong("totalOrdersCount")?.toInt() ?: 0,
                                    totalInvoiceAmount = monthDoc.getLong("totalInvoiceAmount") ?: 0L,
                                    totalInvoiceAmountWithGst = monthDoc.getLong("totalInvoiceAmountWithGst") ?: 0L,
                                    totalPayments = monthDoc.getLong("totalPayments") ?: 0L,
                                    totalExpenses = monthDoc.getLong("totalExpenses") ?: 0L,
                                    closingBalance = monthDoc.getLong("closingBalance") ?: 0L,
                                    totalGstAmounts = monthDoc.getLong("totalGstAmounts") ?: 0L,
                                    totalDiscount = monthDoc.getLong("totalDiscount") ?: 0L,
                                    updatedAt = monthDoc.getDate("updatedAt")?.toInstant() ?: Instant.now()
                                )

                            snapshotsList.add(snapshot)
                        }

                    } catch (e: Exception) {
                        println("Error parsing month ${monthDoc.id}: " + e.message)
                    }
                }
            }

        } catch (e: Exception) {
            println("Error loading snapshots for client=$clientId: " + e.message)
            e.printStackTrace()
        }

        return snapshotsList.sortedBy { it.yearMonth }
    }

    override fun getSnapshot(clientId: String, yearMonth: YearMonth): ClientBalanceSnapshot? {

        try {
            val year = yearMonth.year.toString()
            val monthsRef = db.collection("clients")
                .document(clientId)
                .collection("snapshots")
                .document(year)
                .collection("months")

            println("CLIENT ID = $clientId")
            println("REQUESTED YEAR-MONTH = $yearMonth")
            println("READING PATH = ${monthsRef.path}")
            println("DOCUMENT ID = ${yearMonth}")

            val monthDoc = monthsRef.document(yearMonth.toString()).get().get()
            println("SNAPSHOT EXISTS = ${monthDoc.exists()}")
            println("SNAPSHOT DATA = ${monthDoc.data}")
            if (!monthDoc.exists()) {
                return null
            }

            return ClientBalanceSnapshot(
                clientId = clientId,
                yearMonth = YearMonth.parse(monthDoc.id),
                openingBalance = monthDoc.getLong("openingBalance") ?: 0L,
                totalOrdersCount = monthDoc.getLong("totalOrdersCount")?.toInt() ?: 0,
                totalInvoiceAmount = monthDoc.getLong("totalInvoiceAmount") ?: 0L,
                totalInvoiceAmountWithGst = monthDoc.getLong("totalInvoiceAmountWithGst") ?: 0L,
                totalPayments = monthDoc.getLong("totalPayments") ?: 0L,
                totalExpenses = monthDoc.getLong("totalExpenses") ?: 0L,
                closingBalance = monthDoc.getLong("closingBalance") ?: 0L,
                totalGstAmounts = monthDoc.getLong("totalGstAmounts") ?: 0L,
                totalDiscount = monthDoc.getLong("totalDiscount") ?: 0L,
                updatedAt = monthDoc.getDate("updatedAt")?.toInstant() ?: Instant.now()
            )

        } catch (e: Exception) {
            println("Error loading snapshot " + "client=$clientId, yearMonth=$yearMonth: ${e.message}")
            e.printStackTrace()
            return null
        }
    }


    override fun saveSnapshot(snapshot: ClientBalanceSnapshot) {
        val year = snapshot.yearMonth.year.toString()
        val docRef = db.collection("clients").document(snapshot.clientId)
            .collection("snapshots").document(year)
            .collection("months").document(snapshot.yearMonth.toString())

        val data = mapOf(
            "clientId" to snapshot.clientId,
            "yearMonth" to snapshot.yearMonth.toString(),
            "openingBalance" to snapshot.openingBalance,
            "totalOrdersCount" to snapshot.totalOrdersCount,
            "totalInvoiceAmount" to snapshot.totalInvoiceAmount,
            "totalInvoiceAmountWithGst" to snapshot.totalInvoiceAmountWithGst,
            "totalPayments" to snapshot.totalPayments,
            "totalExpenses" to snapshot.totalExpenses,
            "closingBalance" to snapshot.closingBalance,
            "totalGstAmounts" to snapshot.totalGstAmounts,
            "totalDiscount" to snapshot.totalDiscount,
            "updatedAt" to java.util.Date.from(snapshot.updatedAt)
        )

        docRef.set(data).get()
    }

    override fun getSnapshot(transaction: Transaction, clientId: String, yearMonth: YearMonth): ClientBalanceSnapshot? {
        val docRef = db.collection("clients")
            .document(clientId)
            .collection("snapshots")
            .document(yearMonth.year.toString())
            .collection("months")
            .document(yearMonth.toString())
        val document = transaction.get(docRef).get()
        if (!document.exists()) {
            return null
        }
        return ClientBalanceSnapshot(
            clientId = clientId,
            yearMonth = YearMonth.parse(document.id),
            openingBalance = document.getLong("openingBalance") ?: 0L,
            totalOrdersCount = document.getLong("totalOrdersCount")?.toInt() ?: 0,
            totalInvoiceAmount = document.getLong("totalInvoiceAmount") ?: 0L,
            totalInvoiceAmountWithGst = document.getLong("totalInvoiceAmountWithGst") ?: 0L,
            totalPayments = document.getLong("totalPayments") ?: 0L,
            totalExpenses = document.getLong("totalExpenses") ?: 0L,
            closingBalance = document.getLong("closingBalance") ?: 0L,
            totalGstAmounts = document.getLong("totalGstAmounts") ?: 0L,
            totalDiscount = document.getLong("totalDiscount") ?: 0L,
            updatedAt = document.getDate("updatedAt")?.toInstant() ?: Instant.now()
        )
    }

    override fun saveSnapshot(
        transaction: Transaction,
        snapshot: ClientBalanceSnapshot
    ) {

        val docRef = db.collection("clients")
            .document(snapshot.clientId)
            .collection("snapshots")
            .document(snapshot.yearMonth.year.toString())
            .collection("months")
            .document(snapshot.yearMonth.toString())

        val data = mapOf(
            "clientId" to snapshot.clientId,
            "yearMonth" to snapshot.yearMonth.toString(),
            "openingBalance" to snapshot.openingBalance,
            "totalOrdersCount" to snapshot.totalOrdersCount,
            "totalInvoiceAmount" to snapshot.totalInvoiceAmount,
            "totalInvoiceAmountWithGst" to snapshot.totalInvoiceAmountWithGst,
            "totalPayments" to snapshot.totalPayments,
            "totalExpenses" to snapshot.totalExpenses,
            "closingBalance" to snapshot.closingBalance,
            "totalGstAmounts" to snapshot.totalGstAmounts,
            "totalDiscount" to snapshot.totalDiscount,
            "updatedAt" to java.util.Date.from(snapshot.updatedAt)
        )

        transaction.set(docRef, data)
    }

    override fun getAllSnapshotsForClient(
        transaction: Transaction,
        clientId: String
    ): List<ClientBalanceSnapshot> {

        val query = db.collectionGroup("months")
            .whereEqualTo("clientId", clientId)

        return transaction.get(query).get().documents.map { document ->

            ClientBalanceSnapshot(
                clientId = clientId,
                yearMonth = YearMonth.parse(
                    document.getString("yearMonth")
                        ?: document.id
                ),
                openingBalance = document.getLong("openingBalance") ?: 0L,
                totalOrdersCount =
                document.getLong("totalOrdersCount")?.toInt() ?: 0,
                totalInvoiceAmount =
                document.getLong("totalInvoiceAmount") ?: 0L,
                totalInvoiceAmountWithGst =
                document.getLong("totalInvoiceAmountWithGst") ?: 0L,
                totalPayments =
                document.getLong("totalPayments") ?: 0L,
                totalExpenses =
                document.getLong("totalExpenses") ?: 0L,
                closingBalance =
                document.getLong("closingBalance") ?: 0L,
                totalGstAmounts =
                document.getLong("totalGstAmounts") ?: 0L,
                totalDiscount =
                document.getLong("totalDiscount") ?: 0L,
                updatedAt =
                document.getDate("updatedAt")?.toInstant()
                    ?: Instant.now()
            )
        }.sortedBy { it.yearMonth }
    }

    override fun findLatestSnapshotForClient(transaction: Transaction, clientId: String): ClientBalanceSnapshot? {
        val query = db.collectionGroup("months")
                .whereEqualTo("clientId", clientId)
                .orderBy("yearMonth", Query.Direction.DESCENDING)
                .limit(1)

        val document = transaction.get(query)
                .get()
                .documents
                .firstOrNull()
                ?: return null

        return ClientBalanceSnapshot(
            clientId = clientId,
            yearMonth = YearMonth.parse(document.getString("yearMonth") ?: document.id),
            openingBalance = document.getLong("openingBalance") ?: 0L,
            totalOrdersCount = document.getLong("totalOrdersCount")?.toInt() ?: 0,
            totalInvoiceAmount = document.getLong("totalInvoiceAmount") ?: 0L,
            totalInvoiceAmountWithGst = document.getLong("totalInvoiceAmountWithGst") ?: 0L,
            totalPayments = document.getLong("totalPayments") ?: 0L,
            totalExpenses = document.getLong("totalExpenses") ?: 0L,
            closingBalance = document.getLong("closingBalance") ?: 0L,
            totalGstAmounts = document.getLong("totalGstAmounts") ?: 0L,
            totalDiscount = document.getLong("totalDiscount") ?: 0L,
            updatedAt = document.getDate("updatedAt")?.toInstant() ?: Instant.now()
        )
    }

    override fun getSnapshotsForClientInRange(transaction: Transaction,
        clientId: String,
        startMonth: YearMonth,
        endMonth: YearMonth
    ): List<ClientBalanceSnapshot> {
        val query =
            db.collectionGroup("months")
                .whereEqualTo("clientId", clientId)
                .whereGreaterThanOrEqualTo(
                    "yearMonth",
                    startMonth.toString()
                )
                .whereLessThanOrEqualTo(
                    "yearMonth",
                    endMonth.toString()
                )
                .orderBy("yearMonth")

        return transaction.get(query)
            .get()
            .documents
            .map { document ->
                ClientBalanceSnapshot(
                    clientId = clientId,
                    yearMonth = YearMonth.parse(document.getString("yearMonth") ?: document.id),
                    openingBalance = document.getLong("openingBalance") ?: 0L,
                    totalOrdersCount = document.getLong("totalOrdersCount")?.toInt() ?: 0,
                    totalInvoiceAmount = document.getLong("totalInvoiceAmount") ?: 0L,
                    totalInvoiceAmountWithGst = document.getLong("totalInvoiceAmountWithGst") ?: 0L,
                    totalPayments = document.getLong("totalPayments") ?: 0L,
                    totalExpenses = document.getLong("totalExpenses") ?: 0L,
                    closingBalance = document.getLong("closingBalance") ?: 0L,
                    totalGstAmounts = document.getLong("totalGstAmounts") ?: 0L,
                    totalDiscount = document.getLong("totalDiscount") ?: 0L,
                    updatedAt = document.getDate("updatedAt")?.toInstant() ?: Instant.now()
                )
            }
            .sortedBy { it.yearMonth }
    }


    override fun getOpeningBalanceSeed(
        transaction: Transaction,
        clientId: String,
        startMonth: YearMonth,
        fallbackOpeningBalance: Long
    ): Long {
        // Prefer the exact month's opening balance.
        getSnapshot(transaction, clientId, startMonth)?.let { return it.openingBalance }

        val searchRadiusMonths = 12L
        val windowStart = startMonth.minusMonths(searchRadiusMonths)
        val windowEnd = startMonth.plusMonths(searchRadiusMonths)

        val snapshotQuery = db.collectionGroup("months")
            .whereEqualTo("clientId", clientId)
            .whereGreaterThanOrEqualTo(
                "yearMonth",
                windowStart.toString()
            )
            .whereLessThanOrEqualTo(
                "yearMonth",
                windowEnd.toString()
            )

        val nearestSnapshot = transaction.get(snapshotQuery)
            .get()
            .documents
            .mapNotNull { document ->
                val yearMonthValue =
                    document.getString("yearMonth") ?: return@mapNotNull null

                val snapshotMonth = runCatching {
                    YearMonth.parse(yearMonthValue)
                }.getOrNull() ?: return@mapNotNull null

                SnapshotCandidate(
                    document = document,
                    month = snapshotMonth,
                    distance = kotlin.math.abs(
                        ChronoUnit.MONTHS.between(
                            startMonth,
                            snapshotMonth
                        )
                    )
                )
            }
            .minWithOrNull(
                compareBy<SnapshotCandidate> { it.distance }
                    // On equal distance, prefer the past snapshot.
                    .thenBy { it.month.isAfter(startMonth) }
            )
            ?: return fallbackOpeningBalance

        return if (nearestSnapshot.month.isBefore(startMonth)) {
            nearestSnapshot.document.getLong("closingBalance")
        } else {
            nearestSnapshot.document.getLong("openingBalance")
        } ?: fallbackOpeningBalance
    }

    private data class SnapshotCandidate(
        val document: DocumentSnapshot,
        val month: YearMonth,
        val distance: Long
    )



    override fun findLatestSnapshotBefore(
        clientId: String,
        yearMonth: YearMonth
    ): ClientBalanceSnapshot? {

        val query =
            db.collectionGroup("months")
                .whereEqualTo("clientId", clientId)
                .whereLessThan(
                    "yearMonth",
                    yearMonth.toString())
                .orderBy(
                    "yearMonth",
                    Query.Direction.DESCENDING)
                .limit(1)

        val document =
            query.get()
                .get()
                .documents
                .firstOrNull()
                ?: return null

        return ClientBalanceSnapshot(
            clientId = clientId,
            yearMonth = YearMonth.parse(
                document.getString("yearMonth")
                    ?: document.id
            ),
            openingBalance =
            document.getLong("openingBalance") ?: 0L,
            totalOrdersCount =
            document.getLong("totalOrdersCount")?.toInt() ?: 0,
            totalInvoiceAmount =
            document.getLong("totalInvoiceAmount") ?: 0L,
            totalInvoiceAmountWithGst =
            document.getLong("totalInvoiceAmountWithGst") ?: 0L,
            totalPayments =
            document.getLong("totalPayments") ?: 0L,
            totalExpenses =
            document.getLong("totalExpenses") ?: 0L,
            closingBalance =
            document.getLong("closingBalance") ?: 0L,
            totalGstAmounts =
            document.getLong("totalGstAmounts") ?: 0L,
            totalDiscount =
            document.getLong("totalDiscount") ?: 0L,
            updatedAt =
            document.getDate("updatedAt")?.toInstant()
                ?: Instant.now()
        )
    }
}
