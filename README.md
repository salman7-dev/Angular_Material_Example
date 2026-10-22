
fun create(client: Client): Client {

    require(client.ownerId.isNotBlank()) {
        "ownerId must not be blank"
    }
    require(client.companyName.isNotBlank()) {
        "client name must not be blank"
    }
    require(client.phone.isNotBlank()) {
        "client phone must not be blank"
    }

    val clientId = if (client.id.isBlank()) {
        IdGenerator.generateClientId()
    } else {
        client.id
    }

    val currentYearMonth = YearMonth.now(clock)

    val clientWithId = client.copy(
        id = clientId,
        latestAmount = client.initialOpeningBalance
    )

    return ownerOperationLockService.executeBusiness(
        ownerId = clientWithId.ownerId
    ) { context, fence ->

        transactionExecutor.execute(context) { transaction, operationContext ->

            require(
                ownerOperationLockRepository.isFenceValidInTransaction(
                    transaction = transaction,
                    ownerId = clientWithId.ownerId,
                    expectedFence = fence
                )
            ) {
                "Owner operation lock fence changed: ${clientWithId.ownerId}"
            }

            operationContext.checkDeadline()

            require(
                !clientRepository.existsInTransaction(
                    transaction = transaction,
                    ownerId = clientWithId.ownerId,
                    clientId = clientId
                )
            ) {
                "Client already exists: $clientId"
            }

            val materializedMonths =
                materializedMonths(currentYearMonth)

            require(materializedMonths.isNotEmpty()) {
                "Materialized months cannot be empty"
            }

            val summaryStatus =
                statusFromOpeningBalance(
                    clientWithId.initialOpeningBalance
                )

            /*
             * Plan the master-history bucket for EVERY materialized
             * month, not only the current month.
             */
            val historyBucketPlans =
                materializedMonths.map { yearMonth ->
                    yearMonth to
                        historyBucketRepository
                            .planBucketAllocationInTransaction(
                                transaction = transaction,
                                ownerId = clientWithId.ownerId,
                                yearMonth = yearMonth.toString()
                            )
                }

            /*
             * A Client stores one bucketId, and the existing update
             * and deletion flows use it across months.
             *
             * Therefore, do not silently accept different bucket IDs.
             */
            val sharedBucketId =
                historyBucketPlans.first().second.bucketId

            require(
                historyBucketPlans.all { (_, plan) ->
                    plan.bucketId == sharedBucketId
                }
            ) {
                "Master-history bucket IDs differ across materialized months. " +
                    "A shared-bucket allocation plan is required."
            }

            val globalSummaryPlans =
                materializedMonths.map { yearMonth ->
                    globalSummaryRepository.planDeltaInTransaction(
                        transaction = transaction,
                        ownerId = clientWithId.ownerId,
                        yearMonth = yearMonth.toString(),
                        delta = globalSummaryDelta(
                            client = clientWithId,
                            yearMonth = yearMonth.toString()
                        )
                    )
                }

            /*
             * Plan summary-index bucket updates for EVERY month.
             * These plans must use sharedBucketId as well.
             */
            val summaryIndexPlans =
                materializedMonths.map { yearMonth ->
                    yearMonth to
                        summaryIndexBucketRepository
                            .planBucketAllocationInTransaction(
                                transaction = transaction,
                                ownerId = clientWithId.ownerId,
                                yearMonth = yearMonth.toString()
                            )
                }

            require(
                summaryIndexPlans.all { (_, plan) ->
                    plan.bucketId == sharedBucketId
                }
            ) {
                "Summary-index bucket IDs do not match master-history " +
                    "bucket ID $sharedBucketId. A shared-bucket allocation " +
                    "plan is required."
            }

            /*
             * Apply master-history bucket metadata for EVERY month.
             */
            historyBucketPlans.forEach { (_, plan) ->
                historyBucketRepository.applyBucketAllocationInTransaction(
                    transaction = transaction,
                    plan = plan
                )
            }

            /*
             * Save all monthly histories under the shared bucket ID.
             */
            createMonthlyHistories(
                transaction = transaction,
                client = clientWithId,
                materializedMonths = materializedMonths,
                bucketId = sharedBucketId
            )

            globalSummaryPlans.forEach { plan ->
                globalSummaryRepository.applyDeltaPlanInTransaction(
                    transaction = transaction,
                    plan = plan
                )
            }

            /*
             * Apply summary-index metadata and insert each index row.
             */
            summaryIndexPlans.forEach { (yearMonth, plan) ->

                summaryIndexBucketRepository
                    .applyBucketAllocationInTransaction(
                        transaction = transaction,
                        plan = plan
                    )

                summaryIndexRepository.insertInTransaction(
                    transaction = transaction,
                    ownerId = clientWithId.ownerId,
                    yearMonth = yearMonth.toString(),
                    bucketId = sharedBucketId,
                    index = SummaryClientIndex(
                        clientName = clientWithId.companyName,
                        clientId = clientWithId.id,
                        amount = clientWithId.initialOpeningBalance.abs(),
                        status = summaryStatus
                    )
                )
            }

            val finalClient = clientWithId.copy(
                bucketId = sharedBucketId,
                type = summaryStatus
            )

            operationContext.checkDeadline()

            clientRepository.createInTransaction(
                transaction = transaction,
                client = finalClient
            )
        }
    }
}
