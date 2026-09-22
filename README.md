fun failClientOperation(
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

        val failedState = currentState.copy(
            status = OperationStatus.FAILED,
            updatedAt = com.google.cloud.Timestamp.now(),
            leaseUntil = null
        )

        transaction.set(document, failedState)

        true
    }.get()
}
