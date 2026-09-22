@Test
fun rejectsFailureWithWrongOperationId() {
    val clientId = "CLI-FAIL-${java.util.UUID.randomUUID()}"

    val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
        Timestamp.now().seconds + 60,
        Timestamp.now().nanos
    )

    val acquired = repository.acquireClientLock(
        clientId = clientId,
        operationId = "OPR-FAIL-002",
        operationType = OperationType.PAYMENT,
        leaseUntil = leaseUntil
    )

    assertTrue(acquired)

    val failed = repository.failClientOperation(
        clientId = clientId,
        operationId = "OPR-WRONG-002"
    )

    assertFalse(failed)

    val state = repository.find(clientId)

    assertNotNull(state)
    assertEquals("OPR-FAIL-002", state?.operationId)
    assertEquals(OperationStatus.RUNNING, state?.status)
    assertNotNull(state?.leaseUntil)
}
