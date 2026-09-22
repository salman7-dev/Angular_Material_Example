@Test
fun allowsNewOperationAfterLeaseExpires() {
    val expiredLease = Timestamp.ofTimeSecondsAndNanos(
        Timestamp.now().seconds - 60,
        Timestamp.now().nanos
    )

    repository.save(
        ClientOperationState(
            clientId = "CLI-LOCK-004",
            operationId = "OPR-005",
            operationType = OperationType.ORDER,
            status = OperationStatus.RUNNING,
            startedAt = expiredLease,
            updatedAt = expiredLease,
            leaseUntil = expiredLease
        )
    )

    val newLease = Timestamp.ofTimeSecondsAndNanos(
        Timestamp.now().seconds + 60,
        Timestamp.now().nanos
    )

    val acquired = repository.acquireClientLock(
        clientId = "CLI-LOCK-004",
        operationId = "OPR-006",
        operationType = OperationType.PAYMENT,
        leaseUntil = newLease
    )

    assertTrue(acquired)

    val state = repository.find("CLI-LOCK-004")

    assertNotNull(state)
    assertEquals("OPR-006", state?.operationId)
    assertEquals(OperationType.PAYMENT, state?.operationType)
    assertEquals(OperationStatus.RUNNING, state?.status)
}
