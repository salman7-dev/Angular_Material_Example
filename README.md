fun renewClientOperation(
    clientId: String,
    operationId: String,
    leaseUntil: Timestamp
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

        val renewedState = currentState.copy(
            updatedAt = Timestamp.now(),
            leaseUntil = leaseUntil
        )

        transaction.set(document, renewedState)

        true
    }.get()
}



fun renew(
    clientId: String,
    operationId: String
): Boolean {
    val now = Timestamp.now()

    val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
        now.seconds + leaseSeconds,
        now.nanos
    )

    return repository.renewClientOperation(
        clientId = clientId,
        operationId = operationId,
        leaseUntil = leaseUntil
    )
}
