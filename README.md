@Test
fun allowsOperationsForDifferentClients() {
    val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
        Timestamp.now().seconds + 60,
        Timestamp.now().nanos
    )

    val firstAcquired = repository.acquireClientLock(
        clientId = "CLI-LOCK-002",
        operationId = "OPR-003",
        operationType = OperationType.ORDER,
        leaseUntil = leaseUntil
    )

    val secondAcquired = repository.acquireClientLock(
        clientId = "CLI-LOCK-003",
        operationId = "OPR-004",
        operationType = OperationType.PAYMENT,
        leaseUntil = leaseUntil
    )

    assertTrue(firstAcquired)
    assertTrue(secondAcquired)

    val firstState = repository.find("CLI-LOCK-002")
    val secondState = repository.find("CLI-LOCK-003")

    assertEquals("OPR-003", firstState?.operationId)
    assertEquals("OPR-004", secondState?.operationId)
}
