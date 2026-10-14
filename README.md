
package com.clientledger.core.service.order

import com.clientledger.core.model.order.Order
import com.clientledger.core.model.order.OrderItem
import java.math.BigDecimal
import java.math.RoundingMode

object OrderDiscountCalculator {

    private val ZERO = BigDecimal("0.00")
    private val HUNDRED = BigDecimal("100")
    private const val MONEY_SCALE = 2

    fun recalculate(
        order: Order,
        discountAmount: BigDecimal
    ): Order {
        val discount = money(discountAmount)

        require(discount >= ZERO) {
            "Discount cannot be negative"
        }

        val originalItemsTotal = order.items.fold(ZERO) { sum, item ->
            sum.add(money(item.originalAmount))
        }

        require(discount <= originalItemsTotal) {
            "Discount cannot exceed original taxable amount"
        }

        require(
            money(order.originalTotalInvoiceAmount)
                .compareTo(originalItemsTotal) == 0
        ) {
            "Original order total does not match original item amounts"
        }

        val updatedItems = allocateDiscount(
            items = order.items,
            discount = discount,
            originalItemsTotal = originalItemsTotal
        )

        val totalTaxableAmount = updatedItems.fold(ZERO) { sum, item ->
            sum.add(item.amount)
        }

        val totalGstAmount = updatedItems.fold(ZERO) { sum, item ->
            sum.add(item.gstAmount)
        }

        val calculatedGrossAmount =
            totalTaxableAmount.add(totalGstAmount)

        // Round the GST-inclusive total to the nearest whole rupee.
        val roundedGrossAmount = calculatedGrossAmount
            .setScale(0, RoundingMode.HALF_UP)
            .setScale(MONEY_SCALE)

        // Can be positive, negative, or zero.
        val roundOffAmount = roundedGrossAmount
            .subtract(calculatedGrossAmount)
            .setScale(MONEY_SCALE, RoundingMode.UNNECESSARY)

        val totalExpense = money(order.totalExpense)

        // Retains the existing calculator's profit formula.
        val profitAmount = totalTaxableAmount.subtract(totalExpense)

        return order.copy(
            // Current values are updated.
            items = updatedItems,
            totalInvoiceAmount = totalTaxableAmount,
            totalGstAmount = totalGstAmount,
            totalInvoiceAmountWithGst = roundedGrossAmount,
            orderDiscount = discount,
            roundOffAmount = roundOffAmount,
            totalExpense = totalExpense,
            profitAmount = money(profitAmount)

            // Original order totals are intentionally not changed.
        )
    }

    private fun allocateDiscount(
        items: List<OrderItem>,
        discount: BigDecimal,
        originalItemsTotal: BigDecimal
    ): List<OrderItem> {
        if (items.isEmpty()) {
            require(discount.compareTo(ZERO) == 0) {
                "Cannot apply a discount to an order without items"
            }

            return emptyList()
        }

        if (originalItemsTotal.compareTo(ZERO) == 0) {
            require(discount.compareTo(ZERO) == 0) {
                "Cannot apply a discount when original total is zero"
            }

            return items.map { item ->
                item.copy(
                    amount = ZERO,
                    gstAmount = ZERO
                )
            }
        }

        // Allocate any paise-level remainder to the largest item.
        val remainderIndex = items.indices.maxBy {
            items[it].originalAmount
        }

        var allocatedSoFar = ZERO

        return items.mapIndexed { index, item ->
            val originalAmount = money(item.originalAmount)
            val remainingDiscount = discount.subtract(allocatedSoFar)

            val allocatedDiscount =
                if (index == remainderIndex) {
                    remainingDiscount
                } else {
                    discount
                        .multiply(originalAmount)
                        .divide(
                            originalItemsTotal,
                            MONEY_SCALE,
                            RoundingMode.HALF_UP
                        )
                        .min(originalAmount)
                        .min(remainingDiscount)
                }.setScale(MONEY_SCALE, RoundingMode.UNNECESSARY)

            allocatedSoFar = allocatedSoFar.add(allocatedDiscount)

            val discountedAmount = originalAmount
                .subtract(allocatedDiscount)
                .setScale(MONEY_SCALE, RoundingMode.UNNECESSARY)

            // GST is calculated only from the current amount and GST rate.
            val recalculatedGst = discountedAmount
                .multiply(BigDecimal.valueOf(item.gstRate.toLong()))
                .divide(
                    HUNDRED,
                    MONEY_SCALE,
                    RoundingMode.HALF_UP
                )

            item.copy(
                // originalAmount, gstRate, and other unspecified
                // properties remain unchanged.
                amount = discountedAmount,
                gstAmount = recalculatedGst
            )
        }
    }

    private fun money(value: BigDecimal): BigDecimal =
        value.setScale(MONEY_SCALE, RoundingMode.HALF_UP)
}
