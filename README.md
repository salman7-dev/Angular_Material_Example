        )

        assertTrue(firstAcquired)
        assertFalse(secondAcquired)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals("OPR-001", state?.operationId)
        assertEquals(OperationType.ORDER, state?.operationType)
        assertEquals(OperationStatus.RUNNING, state?.status)
    }

    @Test
    fun allowsOperationsForDifferentClients() {
        val firstClientId = "CLI-LOCK-${java.util.UUID.randomUUID()}"
        val secondClientId = "CLI-LOCK-${java.util.UUID.randomUUID()}"

        val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
            Timestamp.now().seconds + 60,
            Timestamp.now().nanos
        )

        val firstAcquired = repository.acquireClientLock(
            clientId = firstClientId,
            operationId = "OPR-003",
            operationType = OperationType.ORDER,
            leaseUntil = leaseUntil
        )

        val secondAcquired = repository.acquireClientLock(
            clientId = secondClientId,
            operationId = "OPR-004",
            operationType = OperationType.PAYMENT,
            leaseUntil = leaseUntil
        )

        assertTrue(firstAcquired)
        assertTrue(secondAcquired)

        val firstState = repository.find(firstClientId)
        val secondState = repository.find(secondClientId)

        assertEquals("OPR-003", firstState?.operationId)
        assertEquals("OPR-004", secondState?.operationId)
    }

    @Test
    fun allowsNewOperationAfterLeaseExpires() {
        val clientId = "CLI-LOCK-${java.util.UUID.randomUUID()}"

        val expiredLease = Timestamp.ofTimeSecondsAndNanos(
            Timestamp.now().seconds - 60,
            Timestamp.now().nanos
        )

        repository.save(
            ClientOperationState(
                clientId = clientId,
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
            clientId = clientId,
            operationId = "OPR-006",
            operationType = OperationType.PAYMENT,
            leaseUntil = newLease
        )

        assertTrue(acquired)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals("OPR-006", state?.operationId)
        assertEquals(OperationType.PAYMENT, state?.operationType)
        assertEquals(OperationStatus.RUNNING, state?.status)
    }

    @Test
    fun allowsOnlyOneConcurrentOperationForSameClient() {
        val clientId = "CLI-LOCK-${java.util.UUID.randomUUID()}"

        val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
            Timestamp.now().seconds + 60,
            Timestamp.now().nanos
        )

        val executor = java.util.concurrent.Executors.newFixedThreadPool(2)

        try {
            val first = executor.submit<Boolean> {
                repository.acquireClientLock(
                    clientId = clientId,
                    operationId = "OPR-007",
                    operationType = OperationType.ORDER,
                    leaseUntil = leaseUntil
                )
            }

            val second = executor.submit<Boolean> {
                repository.acquireClientLock(
                    clientId = clientId,
                    operationId = "OPR-008",
                    operationType = OperationType.PAYMENT,
                    leaseUntil = leaseUntil
                )
            }

            val firstResult = first.get()
            val secondResult = second.get()

            assertNotEquals(firstResult, secondResult)

            val state = repository.find(clientId)

            assertNotNull(state)
            assertEquals(OperationStatus.RUNNING, state?.status)

            assertTrue(
                state?.operationId == "OPR-007" ||
                state?.operationId == "OPR-008"
            )
        } finally {
            executor.shutdown()
        }
    }

    @Test
    fun completesOwnRunningOperation() {
        val clientId = "CLI-COMPLETE-${java.util.UUID.randomUUID()}"

        val leaseUntil = Timestamp.ofTimeSecondsAndNanos(
            Timestamp.now().seconds + 60,
            Timestamp.now().nanos
        )

        val acquired = repository.acquireClientLock(
            clientId = clientId,
            operationId = "OPR-COMPLETE-001",
            operationType = OperationType.ORDER,
            leaseUntil = leaseUntil
        )

        assertTrue(acquired)

        val completed = repository.completeClientOperation(
            clientId = clientId,
            operationId = "OPR-COMPLETE-001"
        )

        assertTrue(completed)

        val state = repository.find(clientId)

        assertNotNull(state)
        assertEquals("OPR-COMPLETE-001", state?.operationId)
        assertEquals(OperationStatus.COMPLETED, state?.status)
        assertNull(state?.leaseUntil)
    }
}
