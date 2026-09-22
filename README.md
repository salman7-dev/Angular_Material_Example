@Test
fun allowsOnlyOneConcurrentOperationForSameClient() {
    val clientId = "CLI-LOCK-005"

    val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
        Timestamp.now().seconds + 60,
        Timestamp.now().nanos
    )

    val executor = java.util.concurrent.Executors.newFixedThreadPool(2)

    try {
        val first = executor.submit<Boolean> {
            repository.acquireClientLock(
                clientId = clientId,
                operationId = "OPR-007",
                operationType = OperationType.ORDER,
                leaseUntil = leaseUntil
            )
        }

        val second = executor.submit<Boolean> {
            repository.acquireClientLock(
                clientId = clientId,
                operationId = "OPR-008",
                operationType = OperationType.PAYMENT,
                leaseUntil = leaseUntil
            )
        }

        val firstResult = first.get()
        val secondResult = second.get()

        assertNotEquals(firstResult, secondResult)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals(OperationStatus.RUNNING, state?.status)

        assertTrue(
            state?.operationId == "OPR-007" ||
            state?.operationId == "OPR-008"
        )
    } finally {
        executor.shutdown()
    }
}
