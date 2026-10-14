
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

        require(
            money(order.originalTotalInvoiceAmount)
                .compareTo(originalItemsTotal) == 0
        ) {
            "Original order total does not match original item amounts"
        }

        val currentGrossTotal = order.items.fold(ZERO) { sum, item ->
            sum.add(money(item.amount).add(money(item.gstAmount)))
        }

        require(discount <= currentGrossTotal) {
            "Discount cannot exceed the current GST-inclusive total"
        }

        val updatedItems = allocateGrossDiscount(
            items = order.items,
            discount = discount,
            currentGrossTotal = currentGrossTotal
        )

        val totalTaxableAmount = updatedItems.fold(ZERO) { sum, item ->
            sum.add(item.amount)
        }

        val totalGstAmount = updatedItems.fold(ZERO) { sum, item ->
            sum.add(item.gstAmount)
        }

        val calculatedGrossTotal = totalTaxableAmount.add(totalGstAmount)

        val roundedGrossTotal = calculatedGrossTotal
            .setScale(0, RoundingMode.HALF_UP)
            .setScale(MONEY_SCALE)

        val roundOffAmount = roundedGrossTotal
            .subtract(calculatedGrossTotal)
            .setScale(MONEY_SCALE, RoundingMode.UNNECESSARY)

        val totalExpense = money(order.totalExpense)
        val profitAmount = totalTaxableAmount.subtract(totalExpense)

        return order.copy(
            items = updatedItems,
            totalInvoiceAmount = totalTaxableAmount,
            totalGstAmount = totalGstAmount,
            totalInvoiceAmountWithGst = roundedGrossTotal,
            orderDiscount = discount,
            roundOffAmount = roundOffAmount,
            totalExpense = totalExpense,
            profitAmount = money(profitAmount)
        )
    }

    private fun allocateGrossDiscount(
        items: List<OrderItem>,
        discount: BigDecimal,
        currentGrossTotal: BigDecimal
    ): List<OrderItem> {
        if (items.isEmpty()) {
            require(discount.compareTo(ZERO) == 0) {
                "Cannot apply a discount to an order without items"
            }
            return emptyList()
        }

        if (currentGrossTotal.compareTo(ZERO) == 0) {
            require(discount.compareTo(ZERO) == 0) {
                "Cannot apply a discount when the current total is zero"
            }
            return items.map { item ->
                item.copy(
                    amount = ZERO,
                    gstAmount = ZERO
                )
            }
        }

        val remainderIndex = items.indices.maxBy { index ->
            money(items[index].amount)
                .add(money(items[index].gstAmount))
        }

        var allocatedDiscount = ZERO

        return items.mapIndexed { index, item ->
            val currentAmount = money(item.amount)
            val currentGst = money(item.gstAmount)
            val itemGross = currentAmount.add(currentGst)

            val remainingDiscount = discount.subtract(allocatedDiscount)

            val itemDiscount =
                if (index == remainderIndex) {
                    remainingDiscount
                } else {
                    discount
                        .multiply(itemGross)
                        .divide(
                            currentGrossTotal,
                            MONEY_SCALE,
                            RoundingMode.HALF_UP
                        )
                        .min(itemGross)
                        .min(remainingDiscount)
                }.setScale(MONEY_SCALE, RoundingMode.UNNECESSARY)

            allocatedDiscount = allocatedDiscount.add(itemDiscount)

            val discountedGross = itemGross
                .subtract(itemDiscount)
                .setScale(MONEY_SCALE, RoundingMode.UNNECESSARY)

            // Keep the GST rate unchanged.
            // Split discounted GST-inclusive value into taxable amount and GST.
            val divisor = HUNDRED.add(
                BigDecimal.valueOf(item.gstRate.toLong())
            )

            val recalculatedAmount = discountedGross
                .multiply(HUNDRED)
                .divide(
                    divisor,
                    MONEY_SCALE,
                    RoundingMode.HALF_UP
                )

            val recalculatedGst = discountedGross
                .subtract(recalculatedAmount)
                .setScale(MONEY_SCALE, RoundingMode.HALF_UP)

            item.copy(
                // originalAmount and gstRate remain unchanged.
                amount = recalculatedAmount,
                gstAmount = recalculatedGst
            )
        }
    }

    private fun money(value: BigDecimal): BigDecimal =
        value.setScale(MONEY_SCALE, RoundingMode.HALF_UP)
}
