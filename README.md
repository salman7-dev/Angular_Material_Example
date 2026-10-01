package com.clientledger.core.service.maintenance

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.enums.MaintenanceMode
import com.clientledger.core.repository.maintenance.MaintenanceRepository
import com.clientledger.core.service.monthlyrollover.MonthlyRolloverOrchestrator
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import io.mockk.mockk
import io.mockk.verify
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import java.time.Clock
import java.time.Instant
import java.time.ZoneOffset

class MaintenanceServiceTest {

    companion object {

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
            if (::firestore.isInitialized) {
                firestore.close()
            }
        }
    }

    @BeforeEach
    fun cleanMaintenanceDocument() {
        firestore
            .collection("system")
            .document("maintenance")
            .delete()
            .get()
    }

    @Test
    fun getCurrentStateReturnsNormalWhenDocumentDoesNotExist() {

        val repository = MaintenanceRepository(firestore)

        val clock = Clock.fixed(
            Instant.parse("2026-09-30T05:00:00Z"),
            ZoneOffset.UTC
        )

        val service = MaintenanceService(
            maintenanceRepository = repository,
            monthlyRolloverOrchestrator = mockk(),
            clock = clock,
            properties = ClientLedgerProperties()
        )

        val result = service.getCurrentState()

        assertEquals(
            MaintenanceMode.NORMAL,
            result.mode
        )

        assertEquals(
            Instant.parse("2026-09-30T05:00:00Z"),
            result.updatedAt
        )
    }

    @Test
    fun blockWritesChangesStateToWriteBlocked() {

        val repository = MaintenanceRepository(firestore)

        val clock = Clock.fixed(
            Instant.parse("2026-09-30T05:50:00Z"),
            ZoneOffset.UTC
        )

        val service = MaintenanceService(
            maintenanceRepository = repository,
            monthlyRolloverOrchestrator = mockk(),
            clock = clock,
            properties = ClientLedgerProperties()
        )

        val result = service.blockWrites(
            reason = "Monthly rollover"
        )

        assertEquals(
            MaintenanceMode.WRITE_BLOCKED,
            result.mode
        )

        assertEquals(
            Instant.parse("2026-09-30T05:50:00Z"),
            result.updatedAt
        )

        assertEquals(
            "Monthly rollover",
            result.reason
        )

        val persisted = repository.find()

        assertNotNull(persisted)

        assertEquals(
            MaintenanceMode.WRITE_BLOCKED,
            persisted!!.mode
        )

        assertEquals(
            "Monthly rollover",
            persisted.reason
        )
    }

    @Test
    fun restoreNormalChangesStateToNormal() {

        val repository = MaintenanceRepository(firestore)

        val clock = Clock.fixed(
            Instant.parse("2026-10-01T00:05:00Z"),
            ZoneOffset.UTC
        )

        val service = MaintenanceService(
            maintenanceRepository = repository,
            monthlyRolloverOrchestrator = mockk(),
            clock = clock,
            properties = ClientLedgerProperties()
        )

        service.blockWrites("Monthly rollover")

        val result = service.restoreNormal(
            reason = "Monthly rollover completed"
        )

        assertEquals(
            MaintenanceMode.NORMAL,
            result.mode
        )

        assertEquals(
            Instant.parse("2026-10-01T00:05:00Z"),
            result.updatedAt
        )

        assertEquals(
            "Monthly rollover completed",
            result.reason
        )

        val persisted = repository.find()

        assertNotNull(persisted)

        assertEquals(
            MaintenanceMode.NORMAL,
            persisted!!.mode
        )
    }

    @Test
    fun performMonthlyRolloverUsesPreviousAndCurrentMonth() {

        val repository = MaintenanceRepository(firestore)

        val clock = Clock.fixed(
            Instant.parse("2026-10-01T00:00:00Z"),
            ZoneOffset.UTC
        )

        val orchestrator =
            mockk<MonthlyRolloverOrchestrator>(relaxed = true)

        val service = MaintenanceService(
            maintenanceRepository = repository,
            monthlyRolloverOrchestrator = orchestrator,
            clock = clock,
            properties = ClientLedgerProperties()
        )

        service.performMonthlyRollover()

        verify(exactly = 1) {
            orchestrator.rollover(
                previousYearMonth = java.time.YearMonth.of(2026, 9),
                newYearMonth = java.time.YearMonth.of(2026, 10)
            )
        }
    }
}

