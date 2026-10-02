package com.clientledger.core.controller.client

import com.clientledger.core.auth.CurrentOwnerResolver
import com.clientledger.core.domain.Client
import com.clientledger.core.pagination.PageResult
import com.clientledger.core.service.client.ClientService
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController
import java.net.URI

@RestController
@RequestMapping("/api/clients")
class ClientController(
    private val clientService: ClientService,
    private val currentOwnerResolver: CurrentOwnerResolver
) {

    @PostMapping
    fun createClient(
        @RequestBody client: Client
    ): ResponseEntity<Client> {

        val ownerId =
            currentOwnerResolver.getOwnerId()

        val clientWithOwner =
            client.copy(
                ownerId = ownerId
            )

        val createdClient =
            clientService.create(
                clientWithOwner
            )

        return ResponseEntity
            .created(
                URI.create("/api/clients/${createdClient.id}")
            )
            .body(createdClient)
    }

    @GetMapping("/{clientId}")
    fun getClient(
        @PathVariable clientId: String
    ): ResponseEntity<Client> {

        val ownerId =
            currentOwnerResolver.getOwnerId()

        val client =
            clientService.findById(
                ownerId = ownerId,
                clientId = clientId
            )
                ?: return ResponseEntity
                    .notFound()
                    .build()

        return ResponseEntity.ok(client)
    }

    @GetMapping
    fun getClients(
        @RequestParam(defaultValue = "50") size: Int,
        @RequestParam(required = false) cursor: String?
    ): ResponseEntity<PageResult<Client>> {

        val ownerId =
            currentOwnerResolver.getOwnerId()

        val result =
            clientService.findPage(
                ownerId = ownerId,
                size = size,
                cursor = cursor
            )

        return ResponseEntity.ok(result)
    }
}

package com.clientledger.core.service.client

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.*
import com.clientledger.core.pagination.PageResult
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.lock.OwnerOperationLockRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.service.lock.OwnerOperationLockService
import com.clientledger.core.transaction.FirestoreTransactionExecutor
import com.clientledger.core.utils.IdGenerator
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Service
import java.time.Clock
import java.time.YearMonth
import kotlin.math.abs

@Service
class ClientService(
    private val firestore: Firestore,
    private val transactionExecutor: FirestoreTransactionExecutor,
    private val clientRepository: ClientRepository,
    private val historyBucketRepository: ClientHistoryBucketRepository,
    private val historyRepository: ClientHistoryRepository,
    private val globalSummaryRepository: GlobalSummaryRepository,
    private val summaryIndexBucketRepository: SummaryClientIndexBucketRepository,
    private val summaryIndexRepository: SummaryClientIndexRepository,
    private val properties: ClientLedgerProperties,
    private val clock: Clock,
    private val ownerOperationLockService: OwnerOperationLockService,
    private val ownerOperationLockRepository: OwnerOperationLockRepository,
) {

    fun create(client: Client): Client {

        require(client.ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(client.name.isNotBlank()) {
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
            ownerId = clientWithId.ownerId,
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

                val summaryStatus =
                    statusFromOpeningBalance(
                        clientWithId.initialOpeningBalance
                    )

                /*
             * READ / PLAN PHASE
             *
             * All transaction reads happen before any writes.
             */

                val historyBucketPlan =
                    historyBucketRepository.planBucketAllocationInTransaction(
                        transaction = transaction,
                        ownerId = clientWithId.ownerId,
                        yearMonth = currentYearMonth.toString()
                    )

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
             * Summary index bucket planning is done for every
             * materialized month.
             */
                val summaryIndexPlans =
                    materializedMonths.map { yearMonth ->

                        summaryIndexBucketRepository
                            .planBucketAllocationInTransaction(
                                transaction = transaction,
                                ownerId = clientWithId.ownerId,
                                yearMonth = yearMonth.toString()
                            )
                    }

                /*
             * WRITE / APPLY PHASE
             */

                historyBucketRepository.applyBucketAllocationInTransaction(
                    transaction = transaction,
                    plan = historyBucketPlan
                )

                createMonthlyHistories(
                    transaction = transaction,
                    client = clientWithId,
                    materializedMonths = materializedMonths,
                    bucketId = historyBucketPlan.bucketId
                )

                globalSummaryPlans.forEach { plan ->

                    globalSummaryRepository.applyDeltaPlanInTransaction(
                        transaction = transaction,
                        plan = plan
                    )
                }

                summaryIndexPlans.forEachIndexed { index, plan ->

                    val yearMonth = materializedMonths[index]

                    summaryIndexBucketRepository
                        .applyBucketAllocationInTransaction(
                            transaction = transaction,
                            plan = plan
                        )

                    summaryIndexRepository.insertInTransaction(
                        transaction = transaction,
                        ownerId = clientWithId.ownerId,
                        yearMonth = yearMonth.toString(),
                        bucketId = plan.bucketId,
                        index = SummaryClientIndex(
                            clientId = clientWithId.id,
                            amount = abs(
                                clientWithId.initialOpeningBalance
                            ),
                            status = summaryStatus
                        )
                    )
                }

                val finalClient =
                    clientWithId.copy(
                        bucketId = historyBucketPlan.bucketId,
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

    private fun materializedMonths(
        currentMonth: YearMonth
    ): List<YearMonth> {

        val editableMonths =
            properties.history.editableMonths

        require(editableMonths > 0) {
            "history.editable-months must be greater than zero"
        }

        return (editableMonths - 1 downTo 0)
            .map { offset ->
                currentMonth.minusMonths(offset.toLong())
            }
    }

    private fun createMonthlyHistories(
        transaction: Transaction,
        client: Client,
        materializedMonths: List<YearMonth>,
        bucketId: String
    ) {

        materializedMonths.forEach { yearMonth ->

            historyRepository.saveInTransaction(
                transaction = transaction,
                history = ClientMonthlyHistory(
                    clientId = client.id,
                    ownerId = client.ownerId,
                    yearMonth = yearMonth.toString(),
                    openingBalance =
                    client.initialOpeningBalance,
                    closingBalance =
                    client.initialOpeningBalance,
                    receivable =
                    if (client.initialOpeningBalance > 0) {
                        client.initialOpeningBalance
                    } else {
                        0
                    },
                    advance =
                    if (client.initialOpeningBalance < 0) {
                        abs(client.initialOpeningBalance)
                    } else {
                        0
                    },
                    status =
                    statusFromOpeningBalance(
                        client.initialOpeningBalance
                    )
                ),
                bucketId = bucketId
            )
        }
    }

    private fun globalSummaryDelta(
        client: Client,
        yearMonth: String
    ): GlobalSummary {

        val openingBalance =
            client.initialOpeningBalance

        return GlobalSummary(
            ownerId = client.ownerId,
            yearMonth = yearMonth,

            receivableClientCount =
            if (openingBalance > 0) 1 else 0,

            advanceClientCount =
            if (openingBalance < 0) 1 else 0,

            settledClientCount =
            if (openingBalance == 0L) 1 else 0,

            totalReceivableAmount =
            if (openingBalance > 0) {
                openingBalance
            } else {
                0
            },

            totalAdvanceAmount =
            if (openingBalance < 0) {
                abs(openingBalance)
            } else {
                0
            }
        )
    }

    private fun statusFromOpeningBalance(
        openingBalance: Long
    ): ClientType {

        return when {

            openingBalance > 0 ->
                ClientType.RECEIVABLE

            openingBalance < 0 ->
                ClientType.ADVANCE

            else ->
                ClientType.SETTLED
        }
    }

    fun findById(
        ownerId: String,
        clientId: String
    ): Client? {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(clientId.isNotBlank()) {
            "clientId must not be blank"
        }

        return clientRepository.findById(
            ownerId = ownerId,
            clientId = clientId
        )
    }

    fun findPage(
        ownerId: String,
        size: Int,
        cursor: String?
    ): PageResult<Client> {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        return clientRepository.findPage(
            ownerId = ownerId,
            size = size,
            cursor = cursor
        )
    }
}
