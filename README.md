PS D:\New folder\client-ledger-backend> .\gradlew.bat test
Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :test

SummaryIntegrationTest > getReceivableClientsReturnsPaginatedClients() FAILED
    org.opentest4j.AssertionFailedError at SummaryIntegrationTest.kt:353

SummaryIntegrationTest > getAdvanceClientsReturnsPaginatedClients() FAILED
    org.opentest4j.AssertionFailedError at SummaryIntegrationTest.kt:442

SummaryClientIndexBucketRepositoryTest > bucketUsesFlatYearMonthStructure() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexBucketRepositoryTest.kt:280

SummaryClientIndexBucketRepositoryTest > allocateFirstClientUsesBucket000() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexBucketRepositoryTest.kt:75

SummaryClientIndexRepositoryTest > findPageReturnsExactly50Clients() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexRepositoryTest.kt:405

SummaryClientIndexRepositoryTest > findPageReturns100ClientsAcrossTwoPages() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexRepositoryTest.kt:469

SummaryClientIndexRepositoryTest > findPageReturns51ClientsAcrossTwoPages() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexRepositoryTest.kt:436

SummaryClientIndexRepositoryTest > findPageReturnsSingleClient() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexRepositoryTest.kt:381

SummaryClientIndexRepositoryTest > findPageHandlesMultipleBuckets() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexRepositoryTest.kt:604

Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :test

248 tests completed, 9 failed

> Task :test FAILED

[Incubating] Problems report is available at: file:///D:/New%20folder/client-ledger-backend/build/reports/problems/problems-report.html

FAILURE: Build failed with an exception.


findPageHandlesMultipleBuckets()
org.opentest4j.AssertionFailedError: expected: <client-000> but was: <client-003>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:182)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:1156)
	at kotlin.test.junit5.JUnit5Asserter.assertEquals(JUnitSupport.kt:32)
	at kotlin.test.AssertionsKt__AssertionsKt.assertEquals(Assertions.kt:63)
	at kotlin.test.AssertionsKt.assertEquals(Unknown Source)
	at kotlin.test.AssertionsKt__AssertionsKt.assertEquals$default(Assertions.kt:62)
	at kotlin.test.AssertionsKt.assertEquals$default(Unknown Source)
	at com.clientledger.core.repository.summary.SummaryClientIndexRepositoryTest.findPageHandlesMultipleBuckets(SummaryClientIndexRepositoryTest.kt:604)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

	findPageReturns100ClientsAcrossTwoPages()
	org.opentest4j.AssertionFailedError: Expected value to be false.
	at org.junit.jupiter.api.AssertionUtils.fail(AssertionUtils.java:38)
	at org.junit.jupiter.api.Assertions.fail(Assertions.java:138)
	at kotlin.test.junit5.JUnit5Asserter.fail(JUnitSupport.kt:56)
	at kotlin.test.Asserter$DefaultImpls.assertTrue(Assertions.kt:694)
	at kotlin.test.junit5.JUnit5Asserter.assertTrue(JUnitSupport.kt:30)
	at kotlin.test.Asserter$DefaultImpls.assertTrue(Assertions.kt:704)
	at kotlin.test.junit5.JUnit5Asserter.assertTrue(JUnitSupport.kt:30)
	at kotlin.test.AssertionsKt__AssertionsKt.assertFalse(Assertions.kt:58)
	at kotlin.test.AssertionsKt.assertFalse(Unknown Source)
	at kotlin.test.AssertionsKt__AssertionsKt.assertFalse$default(Assertions.kt:56)
	at kotlin.test.AssertionsKt.assertFalse$default(Unknown Source)
	at com.clientledger.core.repository.summary.SummaryClientIndexRepositoryTest.findPageReturns100ClientsAcrossTwoPages(SummaryClientIndexRepositoryTest.kt:469)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

	findPageReturns51ClientsAcrossTwoPages()
	org.opentest4j.AssertionFailedError: expected: <1> but was: <50>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:182)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:1156)
	at kotlin.test.junit5.JUnit5Asserter.assertEquals(JUnitSupport.kt:32)
	at kotlin.test.AssertionsKt__AssertionsKt.assertEquals(Assertions.kt:63)
	at kotlin.test.AssertionsKt.assertEquals(Unknown Source)
	at kotlin.test.AssertionsKt__AssertionsKt.assertEquals$default(Assertions.kt:62)
	at kotlin.test.AssertionsKt.assertEquals$default(Unknown Source)
	at com.clientledger.core.repository.summary.SummaryClientIndexRepositoryTest.findPageReturns51ClientsAcrossTwoPages(SummaryClientIndexRepositoryTest.kt:436)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

	findPageReturnsExactly50Clients()
	org.opentest4j.AssertionFailedError: Expected value to be false.
	at org.junit.jupiter.api.AssertionUtils.fail(AssertionUtils.java:38)
	at org.junit.jupiter.api.Assertions.fail(Assertions.java:138)
	at kotlin.test.junit5.JUnit5Asserter.fail(JUnitSupport.kt:56)
	at kotlin.test.Asserter$DefaultImpls.assertTrue(Assertions.kt:694)
	at kotlin.test.junit5.JUnit5Asserter.assertTrue(JUnitSupport.kt:30)
	at kotlin.test.Asserter$DefaultImpls.assertTrue(Assertions.kt:704)
	at kotlin.test.junit5.JUnit5Asserter.assertTrue(JUnitSupport.kt:30)
	at kotlin.test.AssertionsKt__AssertionsKt.assertFalse(Assertions.kt:58)
	at kotlin.test.AssertionsKt.assertFalse(Unknown Source)
	at kotlin.test.AssertionsKt__AssertionsKt.assertFalse$default(Assertions.kt:56)
	at kotlin.test.AssertionsKt.assertFalse$default(Unknown Source)
	at com.clientledger.core.repository.summary.SummaryClientIndexRepositoryTest.findPageReturnsExactly50Clients(SummaryClientIndexRepositoryTest.kt:405)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

	findPageReturnsSingleClient()

	org.opentest4j.AssertionFailedError: expected: <1> but was: <10>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:182)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:1156)
	at kotlin.test.junit5.JUnit5Asserter.assertEquals(JUnitSupport.kt:32)
	at kotlin.test.AssertionsKt__AssertionsKt.assertEquals(Assertions.kt:63)
	at kotlin.test.AssertionsKt.assertEquals(Unknown Source)
	at kotlin.test.AssertionsKt__AssertionsKt.assertEquals$default(Assertions.kt:62)
	at kotlin.test.AssertionsKt.assertEquals$default(Unknown Source)
	at com.clientledger.core.repository.summary.SummaryClientIndexRepositoryTest.findPageReturnsSingleClient(SummaryClientIndexRepositoryTest.kt:381)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

	allocateFirstClientUsesBucket000()
	org.opentest4j.AssertionFailedError: expected: java.lang.Integer@64c79b69<100> but was: java.lang.Long@a94c8e3<100>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:182)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:177)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:1145)
	at com.clientledger.core.repository.summary.SummaryClientIndexBucketRepositoryTest.allocateFirstClientUsesBucket000(SummaryClientIndexBucketRepositoryTest.kt:75)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

	bucketUsesFlatYearMonthStructure()

	org.opentest4j.AssertionFailedError: expected: java.lang.Integer@64c79b69<100> but was: java.lang.Long@a94c8e3<100>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:182)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:177)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:1145)
	at com.clientledger.core.repository.summary.SummaryClientIndexBucketRepositoryTest.bucketUsesFlatYearMonthStructure(SummaryClientIndexBucketRepositoryTest.kt:280)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

	getAdvanceClientsReturnsPaginatedClients()
	org.opentest4j.AssertionFailedError: expected: <2> but was: <0>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:150)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:145)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:531)
	at com.clientledger.core.integration.summary.SummaryIntegrationTest.getAdvanceClientsReturnsPaginatedClients(SummaryIntegrationTest.kt:442)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

	getReceivableClientsReturnsPaginatedClients()

	org.opentest4j.AssertionFailedError: expected: <2> but was: <0>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:150)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:145)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:531)
	at com.clientledger.core.integration.summary.SummaryIntegrationTest.getReceivableClientsReturnsPaginatedClients(SummaryIntegrationTest.kt:353)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	




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
        bucketId: String,
        size: Int
    ) {
        firestore
            .collection("owners")
            .document(ownerId)
            .collection("summary_client_index")
            .document(yearMonth)
            .collection("buckets")
            .document(bucketId)
            .set(
                mapOf(
                    "capacity" to properties.summary.indexBucketCapacity,
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
