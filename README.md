package com.clientledger.core.domain

import java.time.Instant

data class Order(
    val id: String = "",
	val ownerId: String = "",
    val clientId: String,
    val orderDate: String,
    val deliveryDate: String? = null,
    val items: List<OrderItem> = emptyList(),
    val totalInvoiceAmount: Long = 0,
    val totalGstAmount: Long = 0,
    val totalExpense: Long = 0,
    val profitAmount: Long = 0,
    val status: OrderStatus = OrderStatus.CREATED,
    val createdAt: Instant = Instant.now(),
    val updatedAt: Instant = Instant.now()
)

data class OrderItem(
    val name: String,
    val quantity: Int = 0,
    val unit: OrderUnit = OrderUnit.SINGLE,
    val amount: Long = 0,
    val gstRate: Int = 0,
    val gstAmount: Long = 0,
    val expense: Long = 0
)

enum class OrderUnit {
    SINGLE,
    DOZEN,
    BOX
}

enum class OrderStatus {
    CREATED,
    PENDING,
    IN_PROGRESS,
    READY,
    DELIVERED,
}


structure:

|--owners --own-uuid
			├──owner	
			├── transactions
			│   └── {year}
			│       └── items
			│           └── {ORD}
						└── {PAY}
