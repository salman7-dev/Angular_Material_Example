
package com.clientledger.core.domain

import com.google.cloud.Timestamp

data class OwnerOperationState(
    val ownerId: String = "",
    val operationId: String = "",
    val operationType: OwnerOperationType = OwnerOperationType.MONTH_INITIALIZATION,
    val status: OperationStatus = OperationStatus.RUNNING,
    val startedAt: Timestamp? = null,
    val updatedAt: Timestamp? = null,
    val leaseUntil: Timestamp? = null
)
