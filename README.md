
package com.clientledger.core.model.order

import java.math.BigDecimal

data class OrderItem(
    val name: String,
    val quantity: Int,
    val unit: String,

    val originalAmount: BigDecimal,
    val amount: BigDecimal,

    val gstRate: BigDecimal,
    val originalGstAmount: BigDecimal,
    val gstAmount: BigDecimal,

    val expense: BigDecimal
)



package com.clientledger.core.model.order

import java.math.BigDecimal
import java.time.Instant
import java.time.LocalDate

data class Order(
    val id: String,
    val ownerId: String,
    val clientId: String,

    val orderDate: LocalDate,
    val deliveryDate: LocalDate,

    val items: List<OrderItem>,

    val originalTotalInvoiceAmount: BigDecimal,
    val totalInvoiceAmount: BigDecimal,

    val originalInvoiceAmountWithGst: BigDecimal,
    val totalInvoiceAmountWithGst: BigDecimal,

    val originalTotalGstAmount: BigDecimal,
    val totalGstAmount: BigDecimal,

    val totalExpense: BigDecimal,
    val profitAmount: BigDecimal,

    val orderDiscount: BigDecimal,
    val roundOffAmount: BigDecimal,

    val status: String,
    val createdAt: Instant,
    val updatedAt: Instant,
    val invoiceId: String?
)


package com.clientledger.core.service.order

import com.clientledger.core.model.order.Order
import java.math.BigDecimal
import java.math.RoundingMode

object OrderAmountCalculator {

    private val ZERO = BigDecimal("0.00")
    private val HUNDRED = BigDecimal("100")
    private const val MONEY_SCALE = 2

    fun calculateGst(
        taxableAmount: BigDecimal,
        gstRate: BigDecimal
    ): BigDecimal {
        return taxableAmount
            .multiply(gstRate)
            .divide(HUNDRED, 2, RoundingMode.HALF_UP)
    }

    fun calculateDiscountedAmounts(
        order: Order,
        discountAmount: BigDecimal
    ): Order {
        require(discountAmount >= ZERO) {
            "Discount cannot be negative"
        }

        require(
            discountAmount <= order.originalTotalInvoiceAmount
        ) {
            "Discount cannot exceed the original taxable amount"
        }

        val originalTotal = order.originalTotalInvoiceAmount

        val updatedItems = order.items.map { item ->
            val allocatedDiscount =
                if (originalTotal.compareTo(ZERO) == 0) {
                    ZERO
                } else {
                    discountAmount
                        .multiply(item.originalAmount)
                        .divide(
                            originalTotal,
                            MONEY_SCALE,
                            RoundingMode.HALF_UP
                        )
                }

            val discountedAmount = item.originalAmount
                .subtract(allocatedDiscount)
                .setScale(MONEY_SCALE, RoundingMode.HALF_UP)

            val recalculatedGst = calculateGst(
                discountedAmount,
                item.gstRate
            )

            item.copy(
                amount = discountedAmount,
                gstAmount = recalculatedGst
            )
        }

        val totalAmount = updatedItems.fold(ZERO) { sum, item ->
            sum.add(item.amount)
        }

        val totalGst = updatedItems.fold(ZERO) { sum, item ->
            sum.add(item.gstAmount)
        }

        val totalWithGst = totalAmount.add(totalGst)

        return order.copy(
            items = updatedItems,
            totalInvoiceAmount = totalAmount,
            totalGstAmount = totalGst,
            totalInvoiceAmountWithGst = totalWithGst,
            orderDiscount = discountAmount,
            roundOffAmount = ZERO,
            profitAmount = totalAmount.subtract(order.totalExpense)
        )
    }
}
