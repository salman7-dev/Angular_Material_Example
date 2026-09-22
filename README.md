
        assertEquals(OperationStatus.COMPLETED, state?.status)
        assertNull(state?.leaseUntil)
    }

    @Test
    fun failsOwnOperation() {
        val ownerId = "OWN-${UUID.randomUUID()}"

        service.acquire(
            ownerId = ownerId,
            operationType = OwnerOperationType.MONTH_INITIALIZATION,
            operationId = "OPR-OWNER-007"
        )

        assertTrue(
            service.fail(
                ownerId = ownerId,
                operationId = "OPR-OWNER-007"
            )
        )

        val state = repository.find(ownerId)

        assertEquals(OperationStatus.FAILED, state?.status)
        assertNull(state?.leaseUntil)
    }

    @Test
    fun allowsNewOperationAfterCompletion() {
        val ownerId = "OWN-${UUID.randomUUID()}"

        service.acquire(
            ownerId = ownerId,
            operationType = OwnerOperationType.MONTH_INITIALIZATION,
            operationId = "OPR-OWNER-008"
        )

        service.complete(
            ownerId = ownerId,
            operationId = "OPR-OWNER-008"
        )

        val second = service.acquire(
            ownerId = ownerId,
            operationType = OwnerOperationType.REBUILD,
            operationId = "OPR-OWNER-009"
        )

        assertEquals("OPR-OWNER-009", second)
    }

    @Test
    fun allowsNewOperationAfterFailure() {
        val ownerId = "OWN-${UUID.randomUUID()}"

        service.acquire(
            ownerId = ownerId,
            operationType = OwnerOperationType.MONTH_INITIALIZATION,
            operationId = "OPR-OWNER-010"
        )

        service.fail(
            ownerId = ownerId,
            operationId = "OPR-OWNER-010"
        )

        val second = service.acquire(
            ownerId = ownerId,
            operationType = OwnerOperationType.RECOVERY,
            operationId = "OPR-OWNER-011"
        )

        assertEquals("OPR-OWNER-011", second)
    }

    @Test
    fun allowsNewOperationAfterLeaseExpires() {
        val ownerId = "OWN-${UUID.randomUUID()}"

        val expiredLease = Timestamp.ofTimeSecondsAndNanos(
            Timestamp.now().seconds - 10,
            Timestamp.now().nanos
        )

        assertEquals(
            "OPR-OWNER-012",
            repository.acquireOwnerLock(
                ownerId = ownerId,
                operationId = "OPR-OWNER-012",
                operationType = OwnerOperationType.MONTH_INITIALIZATION,
                leaseUntil = expiredLease
            )
                .let { if (it) "OPR-OWNER-012" else null }
        )

        val second = service.acquire(
            ownerId = ownerId,
            operationType = OwnerOperationType.RECOVERY,
            operationId = "OPR-OWNER-013"
        )

        assertEquals("OPR-OWNER-013", second)
    }

    @Test
    fun rejectsCompletionWithWrongOperationId() {
        val ownerId = "OWN-${UUID.randomUUID()}"

        service.acquire(
            ownerId = ownerId,
            operationType = OwnerOperationType.MONTH_INITIALIZATION,
            operationId = "OPR-OWNER-014"
        )

        assertFalse(
            service.complete(
                ownerId = ownerId,
                operationId = "OPR-WRONG"
            )
        )

        val state = repository.find(ownerId)

        assertEquals(OperationStatus.RUNNING, state?.status)
    }

    @Test
    fun rejectsFailureWithWrongOperationId() {
        val ownerId = "OWN-${UUID.randomUUID()}"

        service.acquire(
            ownerId = ownerId,
            operationType = OwnerOperationType.MONTH_INITIALIZATION,
            operationId = "OPR-OWNER-015"
        )

        assertFalse(
            service.fail(
                ownerId = ownerId,
                operationId = "OPR-WRONG"
            )
        )

        val state = repository.find(ownerId)

        assertEquals(OperationStatus.RUNNING, state?.status)
    }

    @Test
    fun renewsOwnOperation() {
        val ownerId = "OWN-${UUID.randomUUID()}"

        service.acquire(
            ownerId = ownerId,
            operationType = OwnerOperationType.MONTH_INITIALIZATION,
            operationId = "OPR-OWNER-016"
        )

        assertTrue(
            service.renew(
                ownerId = ownerId,
                operationId = "OPR-OWNER-016"
            )
        )

        val state = repository.find(ownerId)

        assertNotNull(state?.leaseUntil)
        assertEquals(OperationStatus.RUNNING, state?.status)
    }

    @Test
    fun rejectsRenewForWrongOperation() {
        val ownerId = "OWN-${UUID.randomUUID()}"

        service.acquire(
            ownerId = ownerId,
            operationType = OwnerOperationType.MONTH_INITIALIZATION,
            operationId = "OPR-OWNER-017"
        )

        assertFalse(
            service.renew(
                ownerId = ownerId,
                operationId = "OPR-WRONG"
            )
        )
    }

    @Test
    fun allowsOnlyOneConcurrentOperationForSameOwner() {
        val ownerId = "OWN-${UUID.randomUUID()}"

        val executor = Executors.newFixedThreadPool(2)

        try {
            val first = executor.submit<String?> {
                service.acquire(
                    ownerId = ownerId,
                    operationType = OwnerOperationType.MONTH_INITIALIZATION,
                    operationId = "OPR-OWNER-CONCURRENT-001"
                )
            }

            val second = executor.submit<String?> {
                service.acquire(
                    ownerId = ownerId,
                    operationType = OwnerOperationType.RECOVERY,
                    operationId = "OPR-OWNER-CONCURRENT-002"
                )
            }

            val firstResult = first.get()
            val secondResult = second.get()

            assertNotEquals(
               
