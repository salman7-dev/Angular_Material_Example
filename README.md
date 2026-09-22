@Test
fun rejectsCompletionWithWrongOperationId() {
    val clientId = "CLI-COMPLETE-${java.util.UUID.randomUUID()}"

    val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
        Timestamp.now().seconds + 60,
        Timestamp.now().nanos
    )

    val acquired = repository.acquireClientLock(
        clientId = clientId,
        operationId = "OPR-COMPLETE-002",
        operationType = OperationType.ORDER,
        leaseUntil = leaseUntil
    )

    assertTrue(acquired)

    val completed = repository.completeClientOperation(
        clientId = clientId,
        operationId = "OPR-WRONG-001"
    )

    assertFalse(completed)

    val state = repository.find(clientId)

    assertNotNull(state)
    assertEquals("OPR-COMPLETE-002", state?.operationId)
    assertEquals(OperationStatus.RUNNING, state?.status)
    assertNotNull(state?.leaseUntil)
}
