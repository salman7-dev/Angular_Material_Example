@Test
fun failsOwnRunningOperation() {
    val clientId = "CLI-FAIL-${java.util.UUID.randomUUID()}"

    val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
        Timestamp.now().seconds + 60,
        Timestamp.now().nanos
    )

    val acquired = repository.acquireClientLock(
        clientId = clientId,
        operationId = "OPR-FAIL-001",
        operationType = OperationType.PAYMENT,
        leaseUntil = leaseUntil
    )

    assertTrue(acquired)

    val failed = repository.failClientOperation(
        clientId = clientId,
        operationId = "OPR-FAIL-001"
    )

    assertTrue(failed)

    val state = repository.find(clientId)

    assertNotNull(state)
    assertEquals("OPR-FAIL-001", state?.operationId)
    assertEquals(OperationStatus.FAILED, state?.status)
    assertNull(state?.leaseUntil)
}
