package com.clientledger.core.service

import com.clientledger.core.domain.Address
import com.clientledger.core.domain.Client
import com.clientledger.core.dto.ApiResponse
import com.clientledger.core.dto.CreateClientRequest
import com.clientledger.core.utils.IdGenerator
import com.google.cloud.firestore.FieldValue
import com.google.cloud.firestore.Firestore
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import org.springframework.stereotype.Service
import java.util.concurrent.TimeUnit

@Service
class ClientService(
    private val firestore: Firestore
) {

    suspend fun createClient(
        ownerId: String,
        request: CreateClientRequest
    ): ApiResponse<ClientResponse> = withContext(Dispatchers.IO) {

        try {
            // Step 1: Generate client ID
            val clientId = IdGenerator.generateClientId()

            // Step 2: Create client document
            val client = Client(
                id = clientId,
                name = request.name,
                phone = request.phone,
                email = request.email,
                gstNumber = request.gstNumber,
                address = request.address?.let {
                    Address(
                        line1 = it.line1,
                        line2 = it.line2,
                        city = it.city,
                        state = it.state,
                        pincode = it.pincode,
                        country = it.country
                    )
                },
                initialBalance = request.initialBalance,
                currentReceivable = request.initialBalance, // Initial balance becomes receivable
                currentAdvance = 0,
                createdAt = System.currentTimeMillis(),
                lastUpdated = System.currentTimeMillis()
            )

            // Step 3: Write to Firestore
            val clientRef = firestore.document("owners/$ownerId/clients/$clientId")
            clientRef.set(client).get(5,TimeUnit.SECONDS)

            // Step 4: Create year_summaries subcollection (empty docs for 4 years)
            val currentYear = java.time.LocalDate.now().year
            for (i in 0 until 4) {
                val year = currentYear + i
                val yearSummaryRef = firestore.document("owners/$ownerId/clients/$clientId/year_summaries/$year")
                yearSummaryRef.set(mapOf(
                    "year" to year,
                    "totalOrders" to 0,
                    "totalPayments" to 0,
                    "totalReceivable" to 0,
                    "totalAdvance" to 0,
                    "monthly" to emptyMap<String, Any>(),
                    "createdAt" to System.currentTimeMillis()
                )).get()
            }

            // Step 5: Update aggregates/metrics/current
            val metricsRef = firestore.document("owners/$ownerId/aggregates/metrics/current")
            metricsRef.update("totalClients", FieldValue.increment(1)).get()

            ApiResponse(
                success = true,
                data = ClientResponse(
                    clientId = clientId,
                    name = client.name,
                    phone = client.phone,
                    currentReceivable = client.currentReceivable,
                    currentAdvance = client.currentAdvance
                ),
                message = "Client created successfully"
            )

        } catch (e: Exception) {
            ApiResponse(
                success = false,
                message = "Failed to create client: ${e.message}"
            )
        }
    }

    data class ClientResponse(
        val clientId: String,
        val name: String,
        val phone: String,
        val currentReceivable: Long,
        val currentAdvance: Long
    )
}
