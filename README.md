@Test
fun renewsOwnRunningOperation() {
    val clientId = "CLI-RENEW-${java.util.UUID.randomUUID()}"

    val initialLease = Timestamp.ofTimeSecondsAndNanos(
        Timestamp.now().seconds + 30,
        Timestamp.now().nanos
    )

    val acquired = repository.acquireClientLock(
        clientId = clientId,
        operationId = "OPR-RENEW-001",
        operationType = OperationType.ORDER,
        leaseUntil = initialLease
    )

    assertTrue(acquired)

    val renewedLease = Timestamp.ofTimeSecondsAndNanos(
        Timestamp.now().seconds + 120,
        Timestamp.now().nanos
    )

    val renewed = repository.renewClientOperation(
        clientId = clientId,
        operationId = "OPR-RENEW-001",
        leaseUntil = renewedLease
    )

    assertTrue(renewed)

    val state = repository.find(clientId)

    assertNotNull(state)
    assertEquals("OPR-RENEW-001", state?.operationId)
    assertEquals(OperationStatus.RUNNING, state?.status)
    assertEquals(renewedLease, state?.leaseUntil)
}
