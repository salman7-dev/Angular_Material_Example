
package com.clientledger.core.integration.client

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientDeletionJob
import com.clientledger.core.repository.client.ClientDeletionJobRepository
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.lock.OwnerOperationLockRepository
import com.clientledger.core.repository.maintenance.MaintenanceRepository
import com.clientledger.core.repository.order.OrderRepository
import com.clientledger.core.repository.payment.PaymentRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.service.client.ClientDeletionPhaseOneService
import com.clientledger.core.service.client.ClientDeletionPhaseTwoService
import com.clientledger.core.service.client.ClientService
import com.clientledger.core.service.lock.OwnerOperationLockService
import com.clientledger.core.transaction.FirestoreTransactionExecutor
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles
import java.math.BigDecimal
import java.time.Clock
import java.time.Instant
import java.time.YearMonth
import java.time.ZoneId

@ActiveProfiles("test")
@SpringBootTest
class ClientDeletionIntegrationTest {

    @Autowired
    private lateinit var properties: ClientLedgerProperties

    @Autowired
    private lateinit var ownerOperationLockService: OwnerOperationLockService

    companion object {
        private val testClock = Clock.fixed(
            Instant.parse("2026-09-24T10:00:00Z"),
            ZoneId.of("UTC")
        )

        private lateinit var firestore: Firestore

        @JvmStatic
        @BeforeAll
        fun setup() {
            firestore = FirestoreOptions.newBuilder()
                .setProjectId("client-ledger-dashboard")
                .setHost("127.0.0.1:8080")
                .setEmulatorHost("127.0.0.1:8080")
                .setCredentials(NoCredentials.getInstance())
                .build()
                .service
        }

        @JvmStatic
        @AfterAll
        fun cleanup() {
            firestore.close()
        }
    }

    /**
     * Exercises Phase 1 and Phase 2 using a real client created through
     * ClientService. This scenario has no orders, payments, or invoices.
     */
    @Test
    fun phaseOneRemovesClientHistoryAndPhaseTwoCompletesDeletion() {
        val ownerId = uniqueOwnerId()
        val clientService = createClientService()
        val phaseOne = createPhaseOneService()
        val phaseTwo = createPhaseTwoService()

        val client = clientService.create(
            Client(
                ownerId = ownerId,
                companyName = "Client To Delete",
                phone = "9876543210",
                initialOpeningBalance = BigDecimal("10000")
            )
        )

        val months = expectedMonths()

        // Confirm that the client and its history exist before deletion.
        assertNotNull(
            ClientRepository(firestore).findById(ownerId, client.id)
        )

        val historyRepository = ClientHistoryRepository(firestore)
        months.forEach { month ->
            assertNotNull(
                historyRepository.find(
                    ownerId = ownerId,
                    yearMonth = month,
                    bucketId = client.bucketId,
                    clientId = client.id
                ),
                "Expected history for $month before deletion"
            )
        }

        // PHASE 1
        val job = phaseOne.execute(ownerId, client.id)

        assertTrue(job.phase1Completed)
        assertFalse(job.phase2Completed)
        assertEquals(ownerId, job.ownerId)
        assertEquals(client.id, job.clientId)
        assertEquals(client.bucketId, job.clientBucketId)
        assertEquals("PHASE_1_COMPLETED", job.status)
        assertEquals(months, job.historyMonths)

        // Client master document must be gone.
        assertNull(
            ClientRepository(firestore).findById(ownerId, client.id)
        )

        // Phase 1 must remove history and the client's summary-index entries.
        val summaryIndexRepository = SummaryClientIndexRepository(firestore)

        months.forEach { month ->
            assertNull(
                historyRepository.find(
                    ownerId = ownerId,
                    yearMonth = month,
                    bucketId = client.bucketId,
                    clientId = client.id
                ),
                "History must be deleted for $month"
            )

            assertNull(
                summaryIndexRepository.find(
                    ownerId = ownerId,
                    yearMonth = month,
                    bucketId = client.bucketId,
                    clientId = client.id
                ),
                "Summary index must be deleted for $month"
            )

            // The owner's summary should still exist, but this client's
            // position must no longer contribute to its totals.
            val summary = GlobalSummaryRepository(firestore).find(
                ownerId = ownerId,
                yearMonth = month
            )

            assertNotNull(summary, "Summary should exist for $month")
            assertEquals(0, summary!!.receivableClientCount)
            assertEquals(0, summary.advanceClientCount)
            assertEquals(0, summary.settledClientCount)
            assertEquals(BigDecimal.ZERO, summary.totalReceivableAmount)
            assertEquals(BigDecimal.ZERO, summary.totalAdvanceAmount)
        }

        // The history bucket documents remain, but their sizes must decrease.
        val historyBucketRepository = ClientHistoryBucketRepository(
            firestore = firestore,
            properties = properties
        )

        months.forEach { month ->
            val bucket = historyBucketRepository.find(
                ownerId = ownerId,
                yearMonth = month,
                bucketId = client.bucketId
            )

            assertNotNull(bucket, "History bucket should remain for $month")
            assertEquals(0, bucket!!.size)
        }

        // The deletion job must be available for Phase 2.
        val jobRepository = ClientDeletionJobRepository(firestore)
        val persistedJob = jobRepository.find(ownerId, client.id)

        assertNotNull(persistedJob)
        assertTrue(persistedJob!!.phase1Completed)

        // PHASE 2: no child documents exist in this scenario.
        phaseTwo.execute(ownerId, client.id)

        // Current Phase 2 behavior removes the job when no child documents
        // remain; it does not retain a completed job.
        assertNull(jobRepository.find(ownerId, client.id))
    }

    /**
     * Exercises Phase 2 against order, payment, and invoice child documents.
     * A second client's documents must not be deleted.
     */
    @Test
    fun phaseTwoDeletesOnlyTargetClientsChildDocuments() {
        val ownerId = uniqueOwnerId()
        val clientId = "CLI-DELETE-TARGET"
        val otherClientId = "CLI-KEEP-OTHER"

        val jobRepository = ClientDeletionJobRepository(firestore)
        val phaseTwo = createPhaseTwoService()

        // Seed a Phase-1-completed job for the target client.
        val now = testClock.instant()
        val job = ClientDeletionJob(
            ownerId = ownerId,
            clientId = clientId,
            clientName = "Target Client",
            clientCreatedAt = now,
            clientBucketId = "bucket-target",
            historyMonths = listOf("2026-09"),
            phase1Completed = true,
            phase1CompletedAt = now,
            phase2Completed = false,
            status = "PHASE_1_COMPLETED",
            createdAt = now,
            updatedAt = now
        )

        firestore.collection("owners")
            .document(ownerId)
            .collection("client_deletion_jobs")
            .document(clientId)
            .set(
                mapOf(
                    "ownerId" to job.ownerId,
                    "clientId" to job.clientId,
                    "clientName" to job.clientName,
                    "clientCreatedAt" to job.clientCreatedAt.toString(),
                    "clientBucketId" to job.clientBucketId,
                    "historyMonths" to job.historyMonths,
                    "phase1Completed" to job.phase1Completed,
                    "phase1CompletedAt" to job.phase1CompletedAt.toString(),
                    "phase2Completed" to job.phase2Completed,
                    "ordersDeleted" to 0L,
                    "paymentsDeleted" to 0L,
                    "invoicesDeleted" to 0L,
                    "status" to job.status,
                    "createdAt" to job.createdAt.toString(),
                    "updatedAt" to job.updatedAt.toString(),
                    "lastError" to null
                )
            ).get()

        // The expected layout is:
        // owners/{ownerId}/transactions/{period}/items/{documentId}
        // owners/{ownerId}/invoices/{period}/items/{documentId}
        val targetOrder = childDocument(
            ownerId, "transactions", "2026-09", "ORD-delete-target", clientId
        )
        val targetPayment = childDocument(
            ownerId, "transactions", "2026-09", "PAY-delete-target", clientId
        )
        val targetInvoice = childDocument(
            ownerId, "invoices", "2026-09", "INV-delete-target", clientId
        )

        val otherOrder = childDocument(
            ownerId, "transactions", "2026-09", "ORD-keep-other", otherClientId
        )
        val otherPayment = childDocument(
            ownerId, "transactions", "2026-09", "PAY-keep-other", otherClientId
        )
        val otherInvoice = childDocument(
            ownerId, "invoices", "2026-09", "INV-keep-other", otherClientId
        )

        val targetDocuments = listOf(
            targetOrder,
            targetPayment,
            targetInvoice
        )
        val otherDocuments = listOf(
            otherOrder,
            otherPayment,
            otherInvoice
        )

        // These minimal documents exercise Phase 2's path classification
        // and ownerId/clientId filtering. Phase 2 does not parse financial data.
        targetDocuments.forEach { reference ->
            reference.set(
                mapOf(
                    "ownerId" to ownerId,
                    "clientId" to clientId
                )
            ).get()
        }

        otherDocuments.forEach { reference ->
            reference.set(
                mapOf(
                    "ownerId" to ownerId,
                    "clientId" to otherClientId
                )
            ).get()
        }

        assertNotNull(jobRepository.find(ownerId, clientId))
        targetDocuments.forEach { assertTrue(it.get().get().exists()) }
        otherDocuments.forEach { assertTrue(it.get().get().exists()) }

        // Execute Phase 2.
        phaseTwo.execute(ownerId, clientId)

        // All target child documents should be deleted.
        targetDocuments.forEach { reference ->
            assertFalse(
                reference.get().get().exists(),
                "Target child document should be deleted: ${reference.path}"
            )
        }

        // Other clients' documents must remain untouched.
        otherDocuments.forEach { reference ->
            assertTrue(
                reference.get().get().exists(),
                "Other client's document must remain: ${reference.path}"
            )
        }

        // The job is removed after Phase 2 finds no remaining target documents.
        assertNull(jobRepository.find(ownerId, clientId))

        // Confirm the remaining child documents belong only to the other client.
        val remainingTargetDocuments = firestore.collectionGroup("items")
            .whereEqualTo("ownerId", ownerId)
            .whereEqualTo("clientId", clientId)
            .get()
            .get()
            .documents

        assertTrue(remainingTargetDocuments.isEmpty())
    }

    private fun childDocument(
        ownerId: String,
        collectionName: String,
        period: String,
        documentId: String,
        clientId: String
    ) = firestore.collection("owners")
        .document(ownerId)
        .collection(collectionName)
        .document(period)
        .collection("items")
        .document(documentId)

    private fun createPhaseOneService(): ClientDeletionPhaseOneService {
        val transactionExecutor = FirestoreTransactionExecutor(firestore)
        val clientRepository = ClientRepository(firestore)
        val historyRepository = ClientHistoryRepository(firestore)
        val summaryIndexRepository = SummaryClientIndexRepository(firestore)
        val globalSummaryRepository = GlobalSummaryRepository(firestore)
        val deletionJobRepository = ClientDeletionJobRepository(firestore)
        val orderRepository = OrderRepository(firestore)
        val paymentRepository = PaymentRepository(firestore)
        val maintenanceRepository = MaintenanceRepository(firestore)

        val lockRepository = OwnerOperationLockRepository(
            firestore = firestore,
            maintenanceRepository = maintenanceRepository
        )

        return ClientDeletionPhaseOneService(
            firestore = firestore,
            transactionExecutor = transactionExecutor,
            clientRepository = clientRepository,
            historyRepository = historyRepository,
            summaryIndexRepository = summaryIndexRepository,
            globalSummaryRepository = globalSummaryRepository,
            deletionJobRepository = deletionJobRepository,
            orderRepository = orderRepository,
            paymentRepository = paymentRepository,
            ownerOperationLockRepository = lockRepository,
            ownerOperationLockService = ownerOperationLockService,
            clock = testClock
        )
    }

    private fun createPhaseTwoService(): ClientDeletionPhaseTwoService {
        val transactionExecutor = FirestoreTransactionExecutor(firestore)
        val deletionJobRepository = ClientDeletionJobRepository(firestore)
        val maintenanceRepository = MaintenanceRepository(firestore)

        val lockRepository = OwnerOperationLockRepository(
            firestore = firestore,
            maintenanceRepository = maintenanceRepository
        )

        return ClientDeletionPhaseTwoService(
            firestore = firestore,
            transactionExecutor = transactionExecutor,
            deletionJobRepository = deletionJobRepository,
            ownerOperationLockService = ownerOperationLockService,
            ownerOperationLockRepository = lockRepository
        )
    }

    private fun createClientService(): ClientService {
        val transactionExecutor = FirestoreTransactionExecutor(firestore)
        val clientRepository = ClientRepository(firestore)

        val historyBucketRepository = ClientHistoryBucketRepository(
            firestore = firestore,
            properties = properties
        )

        val historyRepository = ClientHistoryRepository(firestore)
        val globalSummaryRepository = GlobalSummaryRepository(firestore)

        val summaryIndexBucketRepository = SummaryClientIndexBucketRepository(
            firestore = firestore,
            properties = properties
        )

        val summaryIndexRepository = SummaryClientIndexRepository(firestore)
        val maintenanceRepository = MaintenanceRepository(firestore)

        val lockRepository = OwnerOperationLockRepository(
            firestore = firestore,
            maintenanceRepository = maintenanceRepository
        )

        return ClientService(
            firestore = firestore,
            transactionExecutor = transactionExecutor,
            clientRepository = clientRepository,
            historyBucketRepository = historyBucketRepository,
            historyRepository = historyRepository,
            globalSummaryRepository = globalSummaryRepository,
            summaryIndexBucketRepository = summaryIndexBucketRepository,
            summaryIndexRepository = summaryIndexRepository,
            ownerOperationLockRepository = lockRepository,
            properties = properties,
            clock = testClock,
            ownerOperationLockService = ownerOperationLockService,
            clientDeletionPhaseOneService = createPhaseOneService(),
            clientDeletionPhaseTwoService = createPhaseTwoService()
        )
    }

    private fun expectedMonths(): List<String> {
        val currentMonth = YearMonth.now(testClock)
        val editableMonths = properties.history.editableMonths

        return (editableMonths - 1 downTo 0).map { offset ->
            currentMonth.minusMonths(offset.toLong()).toString()
        }
    }

    private fun uniqueOwnerId(): String =
        "owner-delete-${System.nanoTime()}"
}
