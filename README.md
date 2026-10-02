package com.clientledger.core.integration.summary

import com.clientledger.core.auth.CurrentOwnerResolver
import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.controller.summary.SummaryController
import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.GlobalSummary
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.service.summary.SummaryService
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import org.mockito.kotlin.mock
import org.mockito.kotlin.whenever
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.http.HttpStatus
import org.springframework.test.context.ActiveProfiles


@SpringBootTest
@ActiveProfiles("test")
class SummaryIntegrationTest {

    @Autowired
    private lateinit var properties: ClientLedgerProperties

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
            firestore.close()
        }
    }

    @Test
    fun getMonthSummaryReturnsStoredSummary() {

        val ownerId = "owner-${System.nanoTime()}"

        val repository =
            GlobalSummaryRepository(firestore)

        val expected =
            GlobalSummary(
                ownerId = ownerId,
                yearMonth = "2026-09",

                receivableClientCount = 3,
                advanceClientCount = 1,
                settledClientCount = 2,

                totalReceivableAmount = 30000,
                totalAdvanceAmount = 5000,

                cashFlow = 20000,
                netProfit = 12000,

                totalInvoiceAmount = 25000,
                totalInvoiceAmountWithGst = 29500,
                totalGstAmount = 4500,
                totalExpenseAmount = 3000,
                totalPaymentAmount = 18000
            )

        repository.applyDelta(
            ownerId = ownerId,
            yearMonth = "2026-09",
            delta = expected
        )

        val controller =
            createController(ownerId)

        val response =
            controller.getMonthSummary(
                yearMonth = "2026-09"
            )

        assertEquals(
            HttpStatus.OK,
            response.statusCode
        )

        assertEquals(
            expected,
            response.body
        )
    }

    @Test
    fun getMonthSummaryReturnsNotFoundWhenSummaryDoesNotExist() {

        val ownerId = "owner-${System.nanoTime()}"

        val controller =
            createController(ownerId)

        val response =
            controller.getMonthSummary(
                yearMonth = "2026-09"
            )

        assertEquals(
            HttpStatus.NOT_FOUND,
            response.statusCode
        )
    }

    @Test
    fun getYearSummarySumsFlowFieldsAndUsesLastAvailableMonthForBalances() {

        val ownerId = "owner-${System.nanoTime()}"

        val repository =
            GlobalSummaryRepository(firestore)

        repository.applyDelta(
            ownerId = ownerId,
            yearMonth = "2026-01",
            delta = GlobalSummary(
                ownerId = ownerId,
                yearMonth = "2026-01",

                receivableClientCount = 1,
                advanceClientCount = 2,
                settledClientCount = 3,

                totalReceivableAmount = 10000,
                totalAdvanceAmount = 2000,

                cashFlow = 1000,
                netProfit = 500,

                totalInvoiceAmount = 5000,
                totalInvoiceAmountWithGst = 5900,
                totalGstAmount = 900,
                totalExpenseAmount = 300,
                totalPaymentAmount = 4000
            )
        )

        repository.applyDelta(
            ownerId = ownerId,
            yearMonth = "2026-03",
            delta = GlobalSummary(
                ownerId = ownerId,
                yearMonth = "2026-03",

                receivableClientCount = 4,
                advanceClientCount = 1,
                settledClientCount = 5,

                totalReceivableAmount = 40000,
                totalAdvanceAmount = 7000,

                cashFlow = 3000,
                netProfit = 1500,

                totalInvoiceAmount = 15000,
                totalInvoiceAmountWithGst = 17700,
                totalGstAmount = 2700,
                totalExpenseAmount = 800,
                totalPaymentAmount = 12000
            )
        )

        repository.applyDelta(
            ownerId = ownerId,
            yearMonth = "2026-09",
            delta = GlobalSummary(
                ownerId = ownerId,
                yearMonth = "2026-09",

                receivableClientCount = 10,
                advanceClientCount = 3,
                settledClientCount = 7,

                totalReceivableAmount = 90000,
                totalAdvanceAmount = 15000,

                cashFlow = 5000,
                netProfit = 2500,

                totalInvoiceAmount = 25000,
                totalInvoiceAmountWithGst = 29500,
                totalGstAmount = 4500,
                totalExpenseAmount = 1000,
                totalPaymentAmount = 20000
            )
        )

        val controller =
            createController(ownerId)

        val response =
            controller.getYearSummary(
                year = 2026
            )

        assertEquals(
            HttpStatus.OK,
            response.statusCode
        )

        val summary =
            response.body

        assertNotNull(summary)

        assertEquals(
            ownerId,
            summary!!.ownerId
        )

        assertEquals(
            "2026",
            summary.yearMonth
        )

        assertEquals(
            10,
            summary.receivableClientCount
        )

        assertEquals(
            3,
            summary.advanceClientCount
        )

        assertEquals(
            7,
            summary.settledClientCount
        )

        assertEquals(
            90000,
            summary.totalReceivableAmount
        )

        assertEquals(
            15000,
            summary.totalAdvanceAmount
        )

        assertEquals(
            9000,
            summary.cashFlow
        )

        assertEquals(
            4500,
            summary.netProfit
        )

        assertEquals(
            45000,
            summary.totalInvoiceAmount
        )

        assertEquals(
            53100,
            summary.totalInvoiceAmountWithGst
        )

        assertEquals(
            8100,
            summary.totalGstAmount
        )

        assertEquals(
            2100,
            summary.totalExpenseAmount
        )

        assertEquals(
            36000,
            summary.totalPaymentAmount
        )
    }

    @Test
    fun getReceivableClientsReturnsPaginatedClients() {

        val ownerId = "owner-${System.nanoTime()}"

        insertBucket(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000",
            size = 2
        )

        insertClientIndex(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000",
            clientId = "CLI-001",
            amount = 10000
        )

        insertClientIndex(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.RECEIVABLE,
            bucketId = "bucket_000",
            clientId = "CLI-002",
            amount = 20000
        )

        val controller =
            createController(ownerId)

        val response =
            controller.getReceivableClients(
                yearMonth = "2026-09",
                size = 50,
                cursor = null
            )

        assertEquals(
            HttpStatus.OK,
            response.statusCode
        )

        val page =
            response.body

        assertNotNull(page)

        assertEquals(
            2,
            page!!.content.size
        )

        assertEquals(
            "CLI-001",
            page.content[0].clientId
        )

        assertEquals(
            10000,
            page.content[0].amount
        )

        assertEquals(
            ClientType.RECEIVABLE,
            page.content[0].status
        )

        assertEquals(
            "CLI-002",
            page.content[1].clientId
        )

        assertEquals(
            20000,
            page.content[1].amount
        )

        assertEquals(
            ClientType.RECEIVABLE,
            page.content[1].status
        )

        assertTrue(!page.hasNext)
        assertEquals(null, page.nextCursor)
    }

    @Test
    fun getAdvanceClientsReturnsPaginatedClients() {

        val ownerId = "owner-${System.nanoTime()}"

        insertBucket(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.ADVANCE,
            bucketId = "bucket_000",
            size = 2
        )

        insertClientIndex(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.ADVANCE,
            bucketId = "bucket_000",
            clientId = "CLI-003",
            amount = 5000
        )

        insertClientIndex(
            ownerId = ownerId,
            yearMonth = "2026-09",
            status = ClientType.ADVANCE,
            bucketId = "bucket_000",
            clientId = "CLI-004",
            amount = 7000
        )

        val controller =
            createController(ownerId)

        val response =
            controller.getAdvanceClients(
                yearMonth = "2026-09",
                size = 50,
                cursor = null
            )

        assertEquals(
            HttpStatus.OK,
            response.statusCode
        )

        val page =
            response.body

        assertNotNull(page)

        assertEquals(
            2,
            page!!.content.size
        )

        assertEquals(
            "CLI-003",
            page.content[0].clientId
        )

        assertEquals(
            5000,
            page.content[0].amount
        )

        assertEquals(
            ClientType.ADVANCE,
            page.content[0].status
        )

        assertEquals(
            "CLI-004",
            page.content[1].clientId
        )

        assertEquals(
            7000,
            page.content[1].amount
        )

        assertEquals(
            ClientType.ADVANCE,
            page.content[1].status
        )

        assertTrue(!page.hasNext)
        assertEquals(null, page.nextCursor)
    }

    private fun createController(
        ownerId: String
    ): SummaryController {

        val globalSummaryRepository =
            GlobalSummaryRepository(firestore)

        val summaryClientIndexRepository =
            SummaryClientIndexRepository(firestore)

        val summaryService =
            SummaryService(
                globalSummaryRepository = globalSummaryRepository,
                summaryClientIndexRepository = summaryClientIndexRepository
            )

        val currentOwnerResolver =
            mock<CurrentOwnerResolver>()

        whenever(
            currentOwnerResolver.getOwnerId()
        ).thenReturn(ownerId)

        return SummaryController(
            summaryService = summaryService,
            currentOwnerResolver = currentOwnerResolver
        )
    }

    private fun insertBucket(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String,
        size: Int
    ) {

        firestore
            .collection("owners")
            .document(ownerId)
            .collection("summary_client_index")
            .document(yearMonth)
            .collection("status")
            .document(status.name.lowercase())
            .collection("buckets")
            .document(bucketId)
            .set(
                mapOf(
                    "capacity" to 300,
                    "size" to size
                )
            )
            .get()
    }

    private fun insertClientIndex(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        bucketId: String,
        clientId: String,
        amount: Long
    ) {

        firestore
            .collection("owners")
            .document(ownerId)
            .collection("summary_client_index")
            .document(yearMonth)
            .collection("status")
            .document(status.name.lowercase())
            .collection("buckets")
            .document(bucketId)
            .collection("clients")
            .document(clientId)
            .set(
                mapOf(
                    "clientId" to clientId,
                    "amount" to amount,
                    "status" to status.name
                )
            )
            .get()
    }
}
package com.clientledger.core.service.summary

import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.GlobalSummary
import com.clientledger.core.domain.SummaryClientIndex
import com.clientledger.core.pagination.PageResult
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import org.springframework.stereotype.Service

@Service
class SummaryService(
    private val globalSummaryRepository: GlobalSummaryRepository,
    private val summaryClientIndexRepository: SummaryClientIndexRepository
) {

    fun getMonthSummary(
        ownerId: String,
        yearMonth: String
    ): GlobalSummary? {

        return globalSummaryRepository.find(
            ownerId = ownerId,
            yearMonth = yearMonth
        )
    }

    fun getYearSummary(
        ownerId: String,
        year: Int
    ): GlobalSummary {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(year in 1..9999) {
            "year must be between 1 and 9999"
        }

        var lastAvailableSummary: GlobalSummary? = null

        var totalCashFlow = 0L
        var totalNetProfit = 0L
        var totalInvoiceAmount = 0L
        var totalInvoiceAmountWithGst = 0L
        var totalGstAmount = 0L
        var totalExpenseAmount = 0L
        var totalPaymentAmount = 0L

        for (month in 1..12) {

            val yearMonth =
                "%04d-%02d".format(year, month)

            val summary =
                globalSummaryRepository.find(
                    ownerId = ownerId,
                    yearMonth = yearMonth
                )
                    ?: continue

            lastAvailableSummary = summary

            totalCashFlow += summary.cashFlow
            totalNetProfit += summary.netProfit
            totalInvoiceAmount += summary.totalInvoiceAmount
            totalInvoiceAmountWithGst += summary.totalInvoiceAmountWithGst
            totalGstAmount += summary.totalGstAmount
            totalExpenseAmount += summary.totalExpenseAmount
            totalPaymentAmount += summary.totalPaymentAmount
        }

        val lastSummary = lastAvailableSummary
            ?: return GlobalSummary(
                ownerId = ownerId,
                yearMonth = year.toString()
            )

        return GlobalSummary(
            ownerId = ownerId,
            yearMonth = year.toString(),

            receivableClientCount =
            lastSummary.receivableClientCount,

            advanceClientCount =
            lastSummary.advanceClientCount,

            settledClientCount =
            lastSummary.settledClientCount,

            totalReceivableAmount =
            lastSummary.totalReceivableAmount,

            totalAdvanceAmount =
            lastSummary.totalAdvanceAmount,

            cashFlow = totalCashFlow,
            netProfit = totalNetProfit,

            totalInvoiceAmount =
            totalInvoiceAmount,

            totalInvoiceAmountWithGst =
            totalInvoiceAmountWithGst,

            totalGstAmount =
            totalGstAmount,

            totalExpenseAmount =
            totalExpenseAmount,

            totalPaymentAmount =
            totalPaymentAmount
        )
    }

    fun getClients(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        size: Int,
        cursor: String?
    ): PageResult<SummaryClientIndex> {

        require(
            status == ClientType.RECEIVABLE ||
                    status == ClientType.ADVANCE
        ) {
            "status must be RECEIVABLE or ADVANCE"
        }

        return summaryClientIndexRepository.findPage(
            ownerId = ownerId,
            yearMonth = yearMonth,
            size = size,
            cursor = cursor
        )
    }

}
