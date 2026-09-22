fun completeClientOperation(
    clientId: String,
    operationId: String
): Boolean {
    val document = firestore
        .collection(COLLECTION)
        .document(clientId)

    return firestore.runTransaction { transaction ->
        val snapshot = transaction.get(document).get()

        if (!snapshot.exists()) {
            return@runTransaction false
        }

        val currentState = snapshot.toObject(ClientOperationState::class.java)
            ?: return@runTransaction false

        if (
            currentState.operationId != operationId ||
            currentState.status != OperationStatus.RUNNING
        ) {
            return@runTransaction false
        }

        val now = com.google.cloud.Timestamp.now()

        val completedState = currentState.copy(
            status = OperationStatus.COMPLETED,
            updatedAt = now,
            leaseUntil = null
        )

        transaction.set(document, completedState)

        true
    }.get()
}
