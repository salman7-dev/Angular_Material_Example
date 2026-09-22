@Test
fun renewsOwnOperation() {
    val clientId = "CLI-RENEW-${java.util.UUID.randomUUID()}"

    val acquired = service.acquire(
        clientId = clientId,
        operationId = "OPR-RENEW-001",
        operationType = OperationType.ORDER
    )

    assertTrue(acquired)

    val renewed = service.renew(
        clientId = clientId,
        operationId = "OPR-RENEW-001"
    )

    assertTrue(renewed)
}


@Test
fun rejectsRenewForWrongOperation() {
    val clientId = "CLI-RENEW-${java.util.UUID.randomUUID()}"

    val acquired = service.acquire(
        clientId = clientId,
        operationId = "OPR-RENEW-002",
        operationType = OperationType.ORDER
    )

    assertTrue(acquired)

    val renewed = service.renew(
        clientId = clientId,
        operationId = "OPR-WRONG-001"
    )

    assertFalse(renewed)
}
