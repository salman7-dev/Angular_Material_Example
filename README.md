PS D:\New folder\client-ledger-backend> .\gradlew.bat test
Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :test

SummaryIntegrationTest > getReceivableClientsReturnsPaginatedClients() FAILED
    org.opentest4j.AssertionFailedError at SummaryIntegrationTest.kt:354

SummaryIntegrationTest > getAdvanceClientsReturnsPaginatedClients() FAILED
    org.opentest4j.AssertionFailedError at SummaryIntegrationTest.kt:444

SummaryClientIndexBucketRepositoryTest > bucketUsesFlatYearMonthStructure() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexBucketRepositoryTest.kt:268

SummaryClientIndexBucketRepositoryTest > planBucketAllocationInTransactionPlansNewBucketWhenAllAreFull() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexBucketRepositoryTest.kt:362

SummaryClientIndexBucketRepositoryTest > allocateClient301UsesBucket001() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexBucketRepositoryTest.kt:98

SummaryClientIndexBucketRepositoryTest > allocateFirstClientUsesBucket000() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexBucketRepositoryTest.kt:75

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

248 tests completed, 10 failed

> Task :test FAILED

[Incubating] Problems report is available at: file:///D:/New%20folder/client-ledger-backend/build/reports/problems/problems-report.html

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':test'.
> There were failing tests. See the report at: file:///D:/New%20folder/client-ledger-backend/build/reports/tests/test/index.html

* Try:
> Run with --scan to get full insights from a Build Scan (powered by Develocity).

Deprecated Gradle features were used in this build, making it incompatible with Gradle 10.

You can use '--warning-mode all' to show the individual deprecation warnings and determine if they come from your own scripts or plugins.

For more on this, please refer to https://docs.gradle.org/9.7.1/userguide/command_line_interface.html#sec:command_line_warnings in the Gradle documentation.

BUILD FAILED in 1m 32s
6 actionable tasks: 2 executed, 4 up-to-date
PS D:\New folder\client-ledger-backend> 
getAdvanceClientsReturnsPaginatedClients()
org.opentest4j.AssertionFailedError: expected: <2> but was: <0>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:150)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:145)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:531)
	at com.clientledger.core.integration.summary.SummaryIntegrationTest.getAdvanceClientsReturnsPaginatedClients(SummaryIntegrationTest.kt:444)
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
	at com.clientledger.core.integration.summary.SummaryIntegrationTest.getReceivableClientsReturnsPaginatedClients(SummaryIntegrationTest.kt:354)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

 allocateClient301UsesBucket001()
 org.opentest4j.AssertionFailedError: expected: <bucket_001> but was: <bucket_003>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:182)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:177)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:1145)
	at com.clientledger.core.repository.summary.SummaryClientIndexBucketRepositoryTest.allocateClient301UsesBucket001(SummaryClientIndexBucketRepositoryTest.kt:98)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

planBucketAllocationInTransactionPlansNewBucketWhenAllAreFull()

org.opentest4j.AssertionFailedError: expected: <bucket_001> but was: <bucket_003>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:182)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:177)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:1145)
	at com.clientledger.core.repository.summary.SummaryClientIndexBucketRepositoryTest.planBucketAllocationInTransactionPlansNewBucketWhenAllAreFull(SummaryClientIndexBucketRepositoryTest.kt:362)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)


 package com.clientledger.core.repository.summary

import com.clientledger.core.config.ClientLedgerProperties
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.junit.jupiter.api.AfterAll
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertNotNull
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.BeforeAll
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles

@SpringBootTest
@ActiveProfiles("test")
class SummaryClientIndexBucketRepositoryTest {

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

    private fun repository(): SummaryClientIndexBucketRepository {
        return SummaryClientIndexBucketRepository(
            firestore = firestore,
            properties = properties
        )
    }

    @Test
    fun allocateFirstClientUsesBucket000() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        val bucketId = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09"
        )

        assertEquals("bucket_000", bucketId)

        val bucket = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            bucketId = "bucket_000"
        )

        assertNotNull(bucket)
        assertEquals(300, bucket!!["capacity"])
        assertEquals(1, bucket["size"])
    }

    @Test
    fun allocateClient301UsesBucket001() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        repeat(300) {
            repository.allocateBucket(
                ownerId = ownerId,
                yearMonth = "2026-09"
            )
        }

        val bucketId = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09"
        )

        assertEquals("bucket_001", bucketId)

        val bucket000 = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            bucketId = "bucket_000"
        )

        val bucket001 = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            bucketId = "bucket_001"
        )

        assertNotNull(bucket000)
        assertNotNull(bucket001)

        assertEquals(300, bucket000!!["size"])
        assertEquals(1, bucket001!!["size"])
    }

    @Test
    fun differentStatusUsesSameBucket() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        val receivableBucket = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09"
        )

        val advanceBucket = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09"
        )

        assertEquals("bucket_000", receivableBucket)
        assertEquals("bucket_000", advanceBucket)

        val bucket = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            bucketId = "bucket_000"
        )

        assertNotNull(bucket)

        assertEquals(2, bucket!!["size"])
    }

    @Test
    fun differentOwnerUsesSeparateBuckets() {

        val repository = repository()

        val owner1 = "owner-1-${System.nanoTime()}"
        val owner2 = "owner-2-${System.nanoTime()}"

        val firstOwnerBucket = repository.allocateBucket(
            ownerId = owner1,
            yearMonth = "2026-09"
        )

        val secondOwnerBucket = repository.allocateBucket(
            ownerId = owner2,
            yearMonth = "2026-09"
        )

        assertEquals("bucket_000", firstOwnerBucket)
        assertEquals("bucket_000", secondOwnerBucket)

        val firstOwner = repository.find(
            ownerId = owner1,
            yearMonth = "2026-09",
            bucketId = "bucket_000"
        )

        val secondOwner = repository.find(
            ownerId = owner2,
            yearMonth = "2026-09",
            bucketId = "bucket_000"
        )

        assertNotNull(firstOwner)
        assertNotNull(secondOwner)

        assertEquals(1, firstOwner!!["size"])
        assertEquals(1, secondOwner!!["size"])
    }

    @Test
    fun differentMonthUsesSeparateBuckets() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        val septemberBucket = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09"
        )

        val octoberBucket = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-10"
        )

        assertEquals("bucket_000", septemberBucket)
        assertEquals("bucket_000", octoberBucket)

        val september = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            bucketId = "bucket_000"
        )

        val october = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-10",
            bucketId = "bucket_000"
        )

        assertNotNull(september)
        assertNotNull(october)

        assertEquals(1, september!!["size"])
        assertEquals(1, october!!["size"])
    }

    @Test
    fun findReturnsNullWhenBucketDoesNotExist() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        val bucket = repository.find(
            ownerId = ownerId,
            yearMonth = "2026-09",
            bucketId = "bucket_000"
        )

        assertEquals(null, bucket)
    }

    @Test
    fun bucketUsesFlatYearMonthStructure() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"

        repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = "2026-09"
        )

        val snapshot = firestore
            .collection("owners")
            .document(ownerId)
            .collection("summary_client_index")
            .document("2026-09")
            .collection("buckets")
            .document("bucket_000")
            .get()
            .get()

        assertTrue(snapshot.exists())
        assertEquals(100, snapshot.getLong("capacity"))
        assertEquals(1, snapshot.getLong("size"))
    }

    @Test
    fun planBucketAllocationInTransactionFindsAvailableBucket() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val bucketId = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = yearMonth
        )

        val plan = firestore.runTransaction { transaction ->

            repository.planBucketAllocationInTransaction(
                transaction = transaction,
                ownerId = ownerId,
                yearMonth = yearMonth
            )
        }.get()

        assertEquals(bucketId, plan.bucketId)
        assertEquals(1, plan.currentSize)
        assertEquals(false, plan.isNewBucket)
    }

    @Test
    fun applyBucketAllocationInTransactionIncreasesBucketSize() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        val bucketId = repository.allocateBucket(
            ownerId = ownerId,
            yearMonth = yearMonth
        )

        firestore.runTransaction { transaction ->

            val plan = repository.planBucketAllocationInTransaction(
                transaction = transaction,
                ownerId = ownerId,
                yearMonth = yearMonth
            )

            repository.applyBucketAllocationInTransaction(
                transaction = transaction,
                plan = plan
            )

            null
        }.get()

        val bucket = repository.find(
            ownerId = ownerId,
            yearMonth = yearMonth,
            bucketId = bucketId
        )

        assertNotNull(bucket)
        assertEquals(2, bucket!!["size"])
    }

    @Test
    fun planBucketAllocationInTransactionPlansNewBucketWhenAllAreFull() {

        val repository = repository()

        val ownerId = "owner-${System.nanoTime()}"
        val yearMonth = "2026-09"

        repeat(300) {
            repository.allocateBucket(
                ownerId = ownerId,
                yearMonth = yearMonth
            )
        }

        val plan = firestore.runTransaction { transaction ->

            repository.planBucketAllocationInTransaction(
                transaction = transaction,
                ownerId = ownerId,
                yearMonth = yearMonth
            )
        }.get()

        assertEquals("bucket_001", plan.bucketId)
        assertEquals(0, plan.currentSize)
        assertEquals(true, plan.isNewBucket)
    }
}

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
