@Test
fun completesOwnRunningOperation() {
    val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
        Timestamp.now().seconds + 60,
        Timestamp.now().nanos
    )

    val acquired = repository.acquireClientLock(
        clientId = "CLI-COMPLETE-001",
        operationId = "OPR-COMPLETE-001",
        operationType = OperationType.ORDER,
        leaseUntil = leaseUntil
    )

    assertTrue(acquired)

    val completed = repository.completeClientOperation(
        clientId = "CLI-COMPLETE-001",
        operationId = "OPR-COMPLETE-001"
    )

    assertTrue(completed)

    val state = repository.find("CLI-COMPLETE-001")

    assertNotNull(state)
    assertEquals("OPR-COMPLETE-001", state?.operationId)
    assertEquals(OperationStatus.COMPLETED, state?.status)
    assertNull(state?.leaseUntil)
}
