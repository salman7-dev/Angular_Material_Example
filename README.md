@Test
fun allowsNewOperationAfterCompletion() {
    val clientId = "CLI-LIFECYCLE-${java.util.UUID.randomUUID()}"

    val firstOperation = service.acquire(
        clientId = clientId,
        operationType = OperationType.ORDER,
        operationId = "OPR-LIFECYCLE-001"
    )

    assertEquals("OPR-LIFECYCLE-001", firstOperation)

    val completed = service.complete(
        clientId = clientId,
        operationId = "OPR-LIFECYCLE-001"
    )

    assertTrue(completed)

    val secondOperation = service.acquire(
        clientId = clientId,
        operationType = OperationType.PAYMENT,
        operationId = "OPR-LIFECYCLE-002"
    )

    assertEquals("OPR-LIFECYCLE-002", secondOperation)
}

@Test
fun allowsNewOperationAfterFailure() {
    val clientId = "CLI-LIFECYCLE-${java.util.UUID.randomUUID()}"

    val firstOperation = service.acquire(
        clientId = clientId,
        operationType = OperationType.ORDER,
        operationId = "OPR-LIFECYCLE-003"
    )

    assertEquals("OPR-LIFECYCLE-003", firstOperation)

    val failed = service.fail(
        clientId = clientId,
        operationId = "OPR-LIFECYCLE-003"
    )

    assertTrue(failed)

    val secondOperation = service.acquire(
        clientId = clientId,
        operationType = OperationType.PAYMENT,
        operationId = "OPR-LIFECYCLE-004"
    )

    assertEquals("OPR-LIFECYCLE-004", secondOperation)
}

@Test
fun allowsOnlyOneConcurrentOperationForSameClient() {
    val clientId = "CLI-CONCURRENT-${java.util.UUID.randomUUID()}"

    val executor = java.util.concurrent.Executors.newFixedThreadPool(2)

    try {
        val first = executor.submit<String?> {
            service.acquire(
                clientId = clientId,
                operationType = OperationType.ORDER,
                operationId = "OPR-CONCURRENT-001"
            )
        }

        val second = executor.submit<String?> {
            service.acquire(
                clientId = clientId,
                operationType = OperationType.PAYMENT,
                operationId = "OPR-CONCURRENT-002"
            )
        }

        val firstResult = first.get()
        val secondResult = second.get()

        assertNotEquals(firstResult != null, secondResult != null)

        assertTrue(
            firstResult == "OPR-CONCURRENT-001" ||
            secondResult == "OPR-CONCURRENT-002"
        )

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals(OperationStatus.RUNNING, state?.status)

        assertTrue(
            state?.operationId == "OPR-CONCURRENT-001" ||
            state?.operationId == "OPR-CONCURRENT-002"
        )
    } finally {
        executor.shutdown()
    }
}
