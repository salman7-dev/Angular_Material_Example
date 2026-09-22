@Test
fun blocksSecondOperationForSameClient() {
    val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
        Timestamp.now().seconds + 60,
        Timestamp.now().nanos
    )

    val firstAcquired = repository.acquireClientLock(
        clientId = "CLI-LOCK-001",
        operationId = "OPR-001",
        operationType = OperationType.ORDER,
        leaseUntil = leaseUntil
    )

    val secondAcquired = repository.acquireClientLock(
        clientId = "CLI-LOCK-001",
        operationId = "OPR-002",
        operationType = OperationType.PAYMENT,
        leaseUntil = leaseUntil
    )

    assertTrue(firstAcquired)
    assertFalse(secondAcquired)

    val state = repository.find("CLI-LOCK-001")

    assertNotNull(state)
    assertEquals("OPR-001", state?.operationId)
    assertEquals(OperationType.ORDER, state?.operationType)
    assertEquals(OperationStatus.RUNNING, state?.status)
}
