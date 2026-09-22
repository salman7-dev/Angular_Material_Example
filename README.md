            clientId = clientId,
            operationType = OperationType.ORDER,
            operationId = "OPR-SERVICE-006"
        )

        assertEquals("OPR-SERVICE-006", operationId)

        val completed = service.complete(
            clientId = clientId,
            operationId = operationId!!
        )

        assertTrue(completed)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals(OperationStatus.COMPLETED, state?.status)
        assertNull(state?.leaseUntil)
    }

    @Test
    fun failsOwnOperation() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        val operationId = service.acquire(
            clientId = clientId,
            operationType = OperationType.PAYMENT,
            operationId = "OPR-SERVICE-007"
        )

        assertEquals("OPR-SERVICE-007", operationId)

        val failed = service.fail(
            clientId = clientId,
            operationId = operationId!!
        )

        assertTrue(failed)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals(OperationStatus.FAILED, state?.status)
        assertNull(state?.leaseUntil)
    }

    @Test
    fun allowsNewOperationAfterLeaseExpires() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        val expiredLease = Timestamp.ofTimeSecondsAndNanos(
            Timestamp.now().seconds - 60,
            Timestamp.now().nanos
        )

        repository.save(
            ClientOperationState(
                clientId = clientId,
                operationId = "OPR-SERVICE-008",
                operationType = OperationType.ORDER,
                status = OperationStatus.RUNNING,
                startedAt = expiredLease,
                updatedAt = expiredLease,
                leaseUntil = expiredLease
            )
        )

        val newOperation = service.acquire(
            clientId = clientId,
            operationType = OperationType.PAYMENT,
            operationId = "OPR-SERVICE-009"
        )

        assertEquals("OPR-SERVICE-009", newOperation)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals("OPR-SERVICE-009", state?.operationId)
        assertEquals(OperationType.PAYMENT, state?.operationType)
        assertEquals(OperationStatus.RUNNING, state?.status)
    }

    @Test
    fun generatesOperationIdWhenNotProvided() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        val operationId = service.acquire(
            clientId = clientId,
            operationType = OperationType.ORDER
        )

        assertNotNull(operationId)
        assertTrue(operationId!!.startsWith("OPR-"))

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals(operationId, state?.operationId)
    }

    @Test
    fun rejectsCompletionWithWrongOperationId() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        service.acquire(
            clientId = clientId,
            operationType = OperationType.ORDER,
            operationId = "OPR-SERVICE-010"
        )

        val completed = service.complete(
            clientId = clientId,
            operationId = "OPR-WRONG-001"
        )

        assertFalse(completed)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals("OPR-SERVICE-010", state?.operationId)
        assertEquals(OperationStatus.RUNNING, state?.status)
    }

    @Test
    fun rejectsFailureWithWrongOperationId() {
        val clientId = "CLI-SERVICE-${java.util.UUID.randomUUID()}"

        service.acquire(
            clientId = clientId,
            operationType = OperationType.PAYMENT,
            operationId = "OPR-SERVICE-011"
        )

        val failed = service.fail(
            clientId = clientId,
            operationId = "OPR-WRONG-002"
        )

        assertFalse(failed)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals("OPR-SERVICE-011", state?.operationId)
        assertEquals(OperationStatus.RUNNING, state?.status)
    }
}
