package com.clientledger.core.service

import com.clientledger.core.domain.Order
import com.clientledger.core.domain.OrderItem
import com.clientledger.core.dto.ApiResponse
import com.clientledger.core.dto.CreateOrderRequest
import com.clientledger.core.utils.IdGenerator
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.SetOptions
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import org.springframework.stereotype.Service
import java.time.LocalDate
import java.util.concurrent.TimeUnit

@Service
class OrderService(
    private val firestore: Firestore
) {

    suspend fun createOrder(
        ownerId: String,
        request: CreateOrderRequest
    ): ApiResponse<OrderResponse> = withContext(Dispatchers.IO) {

        try {
            // Step 1: Generate order ID
            val orderId = IdGenerator.generateOrderId()

            // Step 2: Get client details
            val clientRef = firestore.document("owners/$ownerId/clients/${request.clientId}")
            val clientDoc = clientRef.get().get(5, TimeUnit.SECONDS)

            if (!clientDoc.exists()) {
                return@withContext ApiResponse(
                    success = false,
                    message = "Client not found"
                )
            }

            val currentReceivable = clientDoc.getDouble("currentReceivable") ?: 0.0
            val currentAdvance = clientDoc.getDouble("currentAdvance") ?: 0.0

            // Step 3: Calculate order totals
            var subtotal = 0.0
            var totalGstAmount = 0.0
            var totalExpense = 0.0

            request.items.forEach { item ->
                val gstAmount = (item.baseAmount * item.gstRate) / 100.0
                val expense = gstAmount * 0.20

                subtotal += item.baseAmount
                totalGstAmount += gstAmount
                totalExpense += expense
            }

            val totalAmount = subtotal + totalGstAmount

            // Step 4: Determine if this order creates receivable or advance
            val isReceivable = currentAdvance >= totalAmount

            if (isReceivable) {
                clientRef.update("currentAdvance", currentAdvance - totalAmount).get(5, TimeUnit.SECONDS)
            } else {
                val newReceivable = currentReceivable + totalAmount
                clientRef.update("currentReceivable", newReceivable).get(5, TimeUnit.SECONDS)
            }

            // Step 5: Create order document
            val order = Order(
                id = orderId,
                clientId = request.clientId,
                orderDate = request.orderDate,
                items = request.items.map { item ->
                    OrderItem(
                        name = item.name,
                        quantity = item.quantity,
                        unit = item.unit,
                        baseAmount = item.baseAmount,
                        gstRate = item.gstRate
                    )
                },
                subtotalAmount = subtotal,
                gstAmount = totalGstAmount,
                totalAmount = totalAmount,
                expense = totalExpense,
                isAdvanceOrder = isReceivable,
                createdAt = System.currentTimeMillis(),
                lastUpdated = System.currentTimeMillis()
            )

            val orderRef = firestore.document("owners/$ownerId/clients/${request.clientId}/orders/$orderId")
            orderRef.set(order).get(5, TimeUnit.SECONDS)

            // Step 6: Update year_summary
            updateYearSummary(ownerId, request.clientId, request.orderDate, totalAmount, isReceivable)

            // Step 7: Update aggregates
            updateAggregates(ownerId, totalAmount, totalExpense, isReceivable)

            ApiResponse(
                success = true,
                data = OrderResponse(
                    orderId = orderId,
                    clientId = request.clientId,
                    totalAmount = totalAmount,
                    gstAmount = totalGstAmount,
                    expense = totalExpense,
                    isAdvanceOrder = isReceivable
                ),
                message = "Order created successfully"
            )

        } catch (e: Exception) {
            ApiResponse(
                success = false,
                message = "Failed to create order: ${e.message}"
            )
        }
    }

    private suspend fun updateYearSummary(
        ownerId: String,
        clientId: String,
        orderDate: String,
        totalAmount: Double,
        isReceivable: Boolean
    ) {
        val orderYear = LocalDate.parse(orderDate).year
        val orderMonth = LocalDate.parse(orderDate).monthValue
        val monthKey = "$orderYear-${String.format("%02d", orderMonth)}"

        val yearSummaryRef = firestore.document("owners/$ownerId/clients/$clientId/year_summaries/$orderYear")
        val yearDoc = yearSummaryRef.get().get(5, TimeUnit.SECONDS)

        if (!yearDoc.exists()) {
            val yearData = mapOf<String, Any>(
                "openingBalance" to 0.0,
                "totalOrders" to totalAmount,
                "totalPayments" to 0.0,
                "totalReceivable" to if (isReceivable) totalAmount else 0.0,
                "totalAdvance" to if (!isReceivable) totalAmount else 0.0,
                "monthly" to mapOf(
                    monthKey to mapOf<String, Any>(
                        "openingBalance" to 0.0,
                        "orders" to totalAmount,
                        "payments" to 0.0,
                        "closing" to totalAmount
                    )
                ),
                "createdAt" to System.currentTimeMillis()
            )
            yearSummaryRef.set(yearData).get(5, TimeUnit.SECONDS)
        } else {
            val currentTotalOrders = yearDoc.getDouble("totalOrders") ?: 0.0
            val currentTotalReceivable = yearDoc.getDouble("totalReceivable") ?: 0.0
            val currentTotalAdvance = yearDoc.getDouble("totalAdvance") ?: 0.0
            val monthlyData = yearDoc.get("monthly") as? Map<String, Map<String, Any>> ?: emptyMap()

            val updates = mutableMapOf<String, Any>(
                "totalOrders" to (currentTotalOrders + totalAmount)
            )

            if (isReceivable) {
                updates["totalReceivable"] = currentTotalReceivable + totalAmount
            } else {
                updates["totalAdvance"] = currentTotalAdvance + totalAmount
            }

            val existingMonthData = monthlyData[monthKey]
            val updatedMonthData = if (existingMonthData != null) {
                mapOf<String, Any>(
                    "openingBalance" to (existingMonthData["openingBalance"] ?: 0.0),
                    "orders" to ((existingMonthData["orders"] ?: 0.0) as Double + totalAmount),
                    "payments" to (existingMonthData["payments"] ?: 0.0),
                    "closing" to ((existingMonthData["closing"] ?: 0.0) as Double + totalAmount)
                )
            } else {
                mapOf<String, Any>(
                    "openingBalance" to 0.0,
                    "orders" to totalAmount,
                    "payments" to 0.0,
                    "closing" to totalAmount
                )
            }

            updates["monthly"] = mapOf(monthKey to updatedMonthData)

            yearSummaryRef.set(updates, SetOptions.merge()).get(5, TimeUnit.SECONDS)
        }
    }

    private suspend fun updateAggregates(
        ownerId: String,
        totalAmount: Double,
        totalExpense: Double,
        isReceivable: Boolean
    ) {
        val metricsRef = firestore.collection("owners")
            .document(ownerId)
            .collection("aggregates")
            .document("metrics")
            .collection("current")
            .document("order")

        metricsRef.set(
            mapOf(
                "totalOrders" to com.google.cloud.firestore.FieldValue.increment(1),
                "totalRevenue" to com.google.cloud.firestore.FieldValue.increment(totalAmount),
                "totalExpense" to com.google.cloud.firestore.FieldValue.increment(totalExpense)
            ),
            SetOptions.merge()
        ).get(5, TimeUnit.SECONDS)
    }

    data class OrderResponse(
        val orderId: String,
        val clientId: String,
        val totalAmount: Double,
        val gstAmount: Double,
        val expense: Double,
        val isAdvanceOrder: Boolean
    )
}




package com.clientledger.core.domain


data class Order(
    val id: String = "",
    val clientId: String,
    val orderDate: String, // "YYYY-MM-DD"
    val items: List<OrderItem>,
    val totalInvoiceAmount: Long,
    val totalGstAmount: Long,
    val totalExpense: Long,
    val month: String, // "YYYY-MM"
    val fiscalYear: String, // "2026_27"
    val createdAt: Long = System.currentTimeMillis()
)

data class OrderItem(
    val name: String,
    val quantity: Int,
    val unit: String,
    val baseAmount: Long,
    val gstRate: Double,
    val gstAmount: Long,
    val lineTotal: Long,
    val expense: Long
)


package com.clientledger.core.dto

data class CreateOrderRequest(
    val clientId: String,
    val orderDate: String,
    val items: List<OrderItemRequest>
)

data class OrderItemRequest(
    val name: String,
    val quantity: Int,
    val unit: String,
    val baseAmount: Double,
    val gstRate: Double
)
