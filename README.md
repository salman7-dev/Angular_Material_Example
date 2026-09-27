package com.clientledger.core.service.owner

import com.clientledger.core.auth.FirebaseRoleService
import com.clientledger.core.auth.UserRole
import com.clientledger.core.domain.Address
import com.clientledger.core.domain.owner.Owner
import com.clientledger.core.repository.owner.OwnerRepository
import org.junit.jupiter.api.Assertions.assertEquals
import org.junit.jupiter.api.Assertions.assertThrows
import org.junit.jupiter.api.Assertions.assertTrue
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.ExtendWith
import org.mockito.Mockito.doNothing
import org.mockito.Mockito.never
import org.mockito.Mockito.verify
import org.mockito.Mockito.`when`
import org.mockito.junit.jupiter.MockitoExtension
import java.time.Instant

@ExtendWith(MockitoExtension::class)
class OwnerServiceTest {

    private lateinit var ownerRepository: OwnerRepository
    private lateinit var firebaseRoleService: FirebaseRoleService
    private lateinit var ownerService: OwnerService

    @BeforeEach
    fun setUp() {

        ownerRepository =
            org.mockito.Mockito.mock(
                OwnerRepository::class.java
            )

        firebaseRoleService =
            org.mockito.Mockito.mock(
                FirebaseRoleService::class.java
            )

        ownerService =
            OwnerService(
                ownerRepository = ownerRepository,
                firebaseRoleService = firebaseRoleService
            )
    }

    @Test
    fun createOwnerUsesFirebaseUidAsOwnerId() {

        val ownerId = "firebase-uid-001"

        `when`(
            ownerRepository.exists(ownerId)
        ).thenReturn(false)

        val created =
            ownerService.createOwner(
                ownerId = ownerId,
                name = "Test Owner",
                businessName = "Test Business",
                phone = "9999999999",
                email = "owner@example.com",
                gstNumber = "GST123",
                address = testAddress()
            )

        assertEquals(
            ownerId,
            created.ownerId
        )

        verify(
            ownerRepository
        ).create(
            org.mockito.ArgumentMatchers.argThat { owner ->
                owner.ownerId == ownerId
            }
        )
    }

    @Test
    fun createOwnerAssignsGeneralRole() {

        val ownerId = "firebase-uid-002"

        `when`(
            ownerRepository.exists(ownerId)
        ).thenReturn(false)

        ownerService.createOwner(
            ownerId = ownerId,
            name = "Test Owner",
            businessName = "Test Business",
            phone = "9999999999",
            email = "owner@example.com",
            gstNumber = "GST123",
            address = testAddress()
        )

        verify(
            firebaseRoleService
        ).setRole(
            uid = ownerId,
            role = UserRole.GENERAL
        )
    }

    @Test
    fun createOwnerPersistsOwner() {

        val ownerId = "firebase-uid-003"

        `when`(
            ownerRepository.exists(ownerId)
        ).thenReturn(false)

        val created =
            ownerService.createOwner(
                ownerId = ownerId,
                name = "Test Owner",
                businessName = "Test Business",
                phone = "9999999999",
                email = "owner@example.com",
                gstNumber = "GST123",
                address = testAddress()
            )

        verify(
            ownerRepository
        ).create(created)
    }

    @Test
    fun createOwnerRejectsDuplicateOwner() {

        val ownerId = "firebase-uid-004"

        `when`(
            ownerRepository.exists(ownerId)
        ).thenReturn(true)

        val exception =
            assertThrows(
                IllegalArgumentException::class.java
            ) {
                ownerService.createOwner(
                    ownerId = ownerId,
                    name = "Test Owner",
                    businessName = "Test Business",
                    phone = "9999999999",
                    email = "owner@example.com",
                    gstNumber = "GST123",
                    address = testAddress()
                )
            }

        assertEquals(
            "Owner already exists",
            exception.message
        )

        verify(
            firebaseRoleService,
            never()
        ).setRole(
            uid = ownerId,
            role = UserRole.GENERAL
        )

        verify(
            ownerRepository,
            never()
        ).create(
            org.mockito.ArgumentMatchers.any()
        )
    }

    @Test
    fun createOwnerRejectsBlankOwnerId() {

        val exception =
            assertThrows(
                IllegalArgumentException::class.java
            ) {
                ownerService.createOwner(
                    ownerId = "",
                    name = "Test Owner",
                    businessName = "Test Business",
                    phone = "9999999999",
                    email = "owner@example.com",
                    gstNumber = "GST123",
                    address = testAddress()
                )
            }

        assertEquals(
            "ownerId must not be blank",
            exception.message
        )
    }

    @Test
    fun createOwnerRejectsBlankName() {

        val exception =
            assertThrows(
                IllegalArgumentException::class.java
            ) {
                ownerService.createOwner(
                    ownerId = "firebase-uid-005",
                    name = "",
                    businessName = "Test Business",
                    phone = "9999999999",
                    email = "owner@example.com",
                    gstNumber = "GST123",
                    address = testAddress()
                )
            }

        assertEquals(
            "name must not be blank",
            exception.message
        )
    }

    @Test
    fun createOwnerRejectsBlankBusinessName() {

        val exception =
            assertThrows(
                IllegalArgumentException::class.java
            ) {
                ownerService.createOwner(
                    ownerId = "firebase-uid-006",
                    name = "Test Owner",
                    businessName = "",
                    phone = "9999999999",
                    email = "owner@example.com",
                    gstNumber = "GST123",
                    address = testAddress()
                )
            }

        assertEquals(
            "businessName must not be blank",
            exception.message
        )
    }

    @Test
    fun getOwnerReturnsExistingOwner() {

        val ownerId = "firebase-uid-007"

        val owner =
            testOwner(ownerId)

        `when`(
            ownerRepository.find(ownerId)
        ).thenReturn(owner)

        val result =
            ownerService.getOwner(ownerId)

        assertEquals(
            owner,
            result
        )

        verify(
            ownerRepository
        ).find(ownerId)
    }

    @Test
    fun getOwnerReturnsNullWhenOwnerDoesNotExist() {

        val ownerId = "firebase-uid-008"

        `when`(
            ownerRepository.find(ownerId)
        ).thenReturn(null)

        val result =
            ownerService.getOwner(ownerId)

        assertEquals(
            null,
            result
        )

        verify(
            ownerRepository
        ).find(ownerId)
    }

    @Test
    fun getOwnerRejectsBlankOwnerId() {

        val exception =
            assertThrows(
                IllegalArgumentException::class.java
            ) {
                ownerService.getOwner("")
            }

        assertEquals(
            "ownerId must not be blank",
            exception.message
        )
    }

    private fun testOwner(
        ownerId: String
    ): Owner {

        val now =
            Instant.parse(
                "2026-01-01T00:00:00Z"
            )

        return Owner(
            ownerId = ownerId,
            name = "Test Owner",
            businessName = "Test Business",
            phone = "9999999999",
            email = "owner@example.com",
            gstNumber = "GST123",
            address = testAddress(),
            createdAt = now,
            updatedAt = now
        )
    }

    private fun testAddress(): Address {

        return Address(
            line1 = "Line 1",
            line2 = "Line 2",
            city = "Mumbai",
            state = "Maharashtra",
            pinCode = "400001",
            country = "India"
        )
    }
}
PS D:\New folder\client-ledger-codespace-main> .\gradlew.bat :app:test --tests "com.clientledger.core.service.owner.OwnerServiceTest"                                              
Calculating task graph as no cached configuration is available for tasks: :app:test --tests com.clientledger.core.service.owner.OwnerServiceTest                                   
Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :app:test

OwnerServiceTest > createOwnerRejectsDuplicateOwner() FAILED
    java.lang.NullPointerException at OwnerServiceTest.kt:175

OwnerServiceTest > getOwnerReturnsNullWhenOwnerDoesNotExist() FAILED
    org.mockito.exceptions.misusing.UnfinishedVerificationException at OwnerServiceTest.kt:32

OwnerServiceTest > createOwnerRejectsBlankName() FAILED
    org.mockito.exceptions.misusing.InvalidUseOfMatchersException at OwnerServiceTest.kt:32

OwnerServiceTest > createOwnerUsesFirebaseUidAsOwnerId() FAILED
    java.lang.NullPointerException at OwnerServiceTest.kt:70

OwnerServiceTest > createOwnerPersistsOwner() FAILED
    org.mockito.exceptions.verification.opentest4j.ArgumentsAreDifferent at OwnerServiceTest.kt:131

10 tests completed, 5 failed

> Task :app:test FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:test'.
> There were failing tests. See the report at: file:///D:/New%20folder/client-ledger-codespace-main/app/build/reports/tests/test/index.html

* Try:
> Run with --scan to get full insights.

BUILD FAILED in 20s
19 actionable tasks: 3 executed, 16 up-to-date
Configuration cache entry stored.
PS D:\New folder\client-ledger-codespace-main>

Error:

createOwnerPersistsOwner()
Argument(s) are different! Wanted:
ownerRepository.create(
    null
);
-> at com.clientledger.core.repository.owner.OwnerRepository.create(OwnerRepository.kt:57)
Actual invocations have different arguments:
ownerRepository.exists(
    "firebase-uid-003"
);
-> at com.clientledger.core.service.owner.OwnerService.createOwner(OwnerService.kt:39)
ownerRepository.create(
    Owner(ownerId=firebase-uid-003, name=Test Owner, businessName=Test Business, phone=9999999999, email=owner@example.com, gstNumber=GST123, address=Address(line1=Line 1, line2=Line 2, city=Mumbai, state=Maharashtra, pinCode=400001, country=India), createdAt=2026-09-27T06:07:49.326536100Z, updatedAt=2026-09-27T06:07:49.326536100Z)
);
-> at com.clientledger.core.service.owner.OwnerService.createOwner(OwnerService.kt:63)

	at app//com.clientledger.core.repository.owner.OwnerRepository.create(OwnerRepository.kt:57)
	at app//com.clientledger.core.service.owner.OwnerServiceTest.createOwnerPersistsOwner(OwnerServiceTest.kt:131)
	at java.base@17.0.11/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)
createOwnerRejectsBlankName()
org.mockito.exceptions.misusing.InvalidUseOfMatchersException: 
Misplaced or misused argument matcher detected here:

-> at com.clientledger.core.service.owner.OwnerServiceTest.createOwnerRejectsDuplicateOwner(OwnerServiceTest.kt:175)

You cannot use argument matchers outside of verification or stubbing.
Examples of correct usage of argument matchers:
    when(mock.get(anyInt())).thenReturn(null);
    doThrow(new RuntimeException()).when(mock).someVoidMethod(any());
    verify(mock).someMethod(contains("foo"))

This message may appear after an NullPointerException if the last matcher is returning an object 
like any() but the stubbed method signature expect a primitive argument, in this case,
use primitive alternatives.
    when(mock.get(any())); // bad use, will raise NPE
    when(mock.get(anyInt())); // correct usage use

Also, this error might show up because you use argument matchers with methods that cannot be mocked.
Following methods *cannot* be stubbed/verified: final/private/equals()/hashCode().
Mocking methods declared on non-public parent classes is not supported.

	at app//com.clientledger.core.service.owner.OwnerServiceTest.setUp(OwnerServiceTest.kt:32)
	at java.base@17.0.11/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)
createOwnerRejectsDuplicateOwner()
java.lang.NullPointerException: any(...) must not be null
	at com.clientledger.core.service.owner.OwnerServiceTest.createOwnerRejectsDuplicateOwner(OwnerServiceTest.kt:175)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
createOwnerUsesFirebaseUidAsOwnerId()
java.lang.NullPointerException: Cannot invoke "com.clientledger.core.domain.owner.Owner.getOwnerId()" because "created" is null
	at com.clientledger.core.service.owner.OwnerServiceTest.createOwnerUsesFirebaseUidAsOwnerId(OwnerServiceTest.kt:70)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
getOwnerReturnsNullWhenOwnerDoesNotExist()
org.mockito.exceptions.misusing.UnfinishedVerificationException: 
Missing method call for verify(mock) here:
-> at com.clientledger.core.service.owner.OwnerServiceTest.createOwnerRejectsDuplicateOwner(OwnerServiceTest.kt:171)

Example of correct verification:
    verify(mock).doSomething()

Also, this error might show up because you verify either of: final/private/equals()/hashCode() methods.
Those methods *cannot* be stubbed/verified.
Mocking methods declared on non-public parent classes is not supported.

	at app//com.clientledger.core.service.owner.OwnerServiceTest.setUp(OwnerServiceTest.kt:32)
	at java.base@17.0.11/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)

reference test class:
package com.clientledger.core.service.client

import com.clientledger.core.config.ClientLedgerProperties
import com.clientledger.core.domain.Client
import com.clientledger.core.domain.ClientType
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.history.ClientHistoryBucketRepository
import com.clientledger.core.repository.history.ClientHistoryRepository
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexBucketRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import com.clientledger.core.transaction.FirestoreTransactionExecutor
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
import java.time.Clock
import java.time.Instant
import java.time.YearMonth
import java.time.ZoneId

@SpringBootTest
@ActiveProfiles("test")
class ClientServiceTest {
    @Autowired
    private lateinit var properties: ClientLedgerProperties
    
    companion object {

        private lateinit var firestore: Firestore

        private val testClock = Clock.fixed(
            Instant.parse("2026-09-24T10:00:00Z"),
            ZoneId.of("UTC")
        )

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
    fun createClientCreatesClientAndEightMonthHistory() {

        val clientService = createClientService()

        val client = Client(
            ownerId = "owner-${System.nanoTime()}",
            name = "ABC Traders",
            phone = "9876543210",
            initialOpeningBalance = 5000
        )

        val savedClient = clientService.create(client)

        assertTrue(savedClient.id.startsWith("CLI-"))
        assertEquals(client.ownerId, savedClient.ownerId)
        assertEquals("ABC Traders", savedClient.name)
        assertEquals(5000, savedClient.initialOpeningBalance)
        assertEquals(5000, savedClient.latestAmount)
        assertEquals("", client.bucketId)
        assertTrue(savedClient.bucketId.isNotBlank())

        val clientRepository = ClientRepository(firestore)

        val storedClient = clientRepository.findById(
            ownerId = client.ownerId,
            clientId = savedClient.id
        )

        assertNotNull(storedClient)
        assertEquals(savedClient.bucketId, storedClient!!.bucketId)

        val historyRepository = ClientHistoryRepository(firestore)

        val expectedMonths = expectedMonths()

        expectedMonths.forEach { yearMonth ->

            val history = historyRepository.find(
                ownerId = client.ownerId,
                yearMonth = yearMonth,
                bucketId = savedClient.bucketId,
                clientId = savedClient.id
            )

            assertNotNull(history)
            assertEquals(savedClient.id, history!!.clientId)
            assertEquals(client.ownerId, history.ownerId)
            assertEquals(yearMonth, history.yearMonth)
            assertEquals(5000, history.openingBalance)
            assertEquals(5000, history.closingBalance)
            assertEquals(5000, history.receivable)
            assertEquals(0, history.advance)
            assertEquals(ClientType.RECEIVABLE, history.status)
        }
    }

    @Test
    fun createClientRejectsDuplicateClientId() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val firstClient = Client(
            id = "CLI-DUPLICATE-001",
            ownerId = ownerId,
            name = "ABC Traders",
            phone = "9876543210",
            initialOpeningBalance = 5000
        )

        clientService.create(firstClient)

        val exception =
            org.junit.jupiter.api.assertThrows<IllegalArgumentException> {
                clientService.create(
                    firstClient.copy(
                        name = "XYZ Traders"
                    )
                )
            }

        assertEquals(
            "Client already exists: CLI-DUPLICATE-001",
            exception.message
        )
    }

    @Test
    fun createClientUsesNextBucketWhenCurrentBucketIsFull() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        repeat(100) { index ->
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Client $index",
                    phone = "900000000$index"
                )
            )
        }

        val hundredFirstClient = clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 101",
                phone = "9111111111"
            )
        )

        assertEquals("bucket_001", hundredFirstClient.bucketId)

        val historyRepository = ClientHistoryRepository(firestore)

        val months = expectedMonths()

        months.forEach { yearMonth ->

            val history = historyRepository.find(
                ownerId = ownerId,
                yearMonth = yearMonth,
                bucketId = "bucket_001",
                clientId = hundredFirstClient.id
            )

            assertNotNull(history)
            assertEquals(
                hundredFirstClient.id,
                history!!.clientId
            )
            assertEquals(
                yearMonth,
                history.yearMonth
            )
        }
    }

    @Test
    fun createClientWithReceivableOpeningBalanceCreatesReceivableHistory() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val client = clientService.create(
            Client(
                ownerId = ownerId,
                name = "Receivable Client",
                phone = "9000000001",
                initialOpeningBalance = 10000
            )
        )

        assertEquals(10000, client.latestAmount)

        val history = findCurrentHistory(
            ownerId = ownerId,
            client = client
        )

        assertEquals(10000, history.openingBalance)
        assertEquals(10000, history.closingBalance)
        assertEquals(10000, history.receivable)
        assertEquals(0, history.advance)
        assertEquals(ClientType.RECEIVABLE, history.status)
    }

    @Test
    fun createClientWithAdvanceOpeningBalanceCreatesAdvanceHistory() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val client = clientService.create(
            Client(
                ownerId = ownerId,
                name = "Advance Client",
                phone = "9000000002",
                initialOpeningBalance = -5000
            )
        )

        assertEquals(-5000, client.latestAmount)

        val history = findCurrentHistory(
            ownerId = ownerId,
            client = client
        )

        assertEquals(-5000, history.openingBalance)
        assertEquals(-5000, history.closingBalance)
        assertEquals(0, history.receivable)
        assertEquals(5000, history.advance)
        assertEquals(ClientType.ADVANCE, history.status)
    }

    @Test
    fun createClientWithZeroOpeningBalanceCreatesSettledHistory() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val client = clientService.create(
            Client(
                ownerId = ownerId,
                name = "Settled Client",
                phone = "9000000003",
                initialOpeningBalance = 0
            )
        )

        assertEquals(0, client.latestAmount)

        val history = findCurrentHistory(
            ownerId = ownerId,
            client = client
        )

        assertEquals(0, history.openingBalance)
        assertEquals(0, history.closingBalance)
        assertEquals(0, history.receivable)
        assertEquals(0, history.advance)
        assertEquals(ClientType.SETTLED, history.status)
    }


    @Test
    fun findByIdReturnsExistingClient() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val createdClient =
            clientService.create(
                Client(
                    ownerId = ownerId,
                    name = "Find Client",
                    phone = "9000000004",
                    initialOpeningBalance = 5000
                )
            )

        val foundClient =
            clientService.findById(
                ownerId = ownerId,
                clientId = createdClient.id
            )

        assertNotNull(foundClient)

        assertEquals(
            createdClient.id,
            foundClient!!.id
        )

        assertEquals(
            ownerId,
            foundClient.ownerId
        )

        assertEquals(
            "Find Client",
            foundClient.name
        )

        assertEquals(
            5000,
            foundClient.initialOpeningBalance
        )
    }

    @Test
    fun findByIdReturnsNullWhenClientDoesNotExist() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        val result =
            clientService.findById(
                ownerId = ownerId,
                clientId = "CLI-NOT-FOUND"
            )

        assertEquals(
            null,
            result
        )
    }

    @Test
    fun findPageReturnsFirstPage() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 001",
                phone = "9000000001"
            )
        )

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 002",
                phone = "9000000002"
            )
        )

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 003",
                phone = "9000000003"
            )
        )

        val page =
            clientService.findPage(
                ownerId = ownerId,
                size = 2,
                cursor = null
            )

        assertEquals(
            2,
            page.content.size
        )

        assertEquals(
            "Client 001",
            page.content[0].name
        )

        assertEquals(
            "Client 002",
            page.content[1].name
        )

        assertTrue(page.hasNext)
        assertNotNull(page.nextCursor)
    }

    @Test
    fun findPageReturnsNextPageUsingCursor() {

        val clientService = createClientService()

        val ownerId = "owner-${System.nanoTime()}"

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 001",
                phone = "9000000001"
            )
        )

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 002",
                phone = "9000000002"
            )
        )

        clientService.create(
            Client(
                ownerId = ownerId,
                name = "Client 003",
                phone = "9000000003"
            )
        )

        val firstPage =
            clientService.findPage(
                ownerId = ownerId,
                size = 2,
                cursor = null
            )

        assertNotNull(firstPage.nextCursor)

        val secondPage =
            clientService.findPage(
                ownerId = ownerId,
                size = 2,
                cursor = firstPage.nextCursor
            )

        assertEquals(
            1,
            secondPage.content.size
        )

        assertEquals(
            "Client 003",
            secondPage.content[0].name
        )

        assertTrue(
            !secondPage.hasNext
        )

        assertEquals(
            null,
            secondPage.nextCursor
        )
    }

    private fun expectedMonths(): List<String> {

        val currentMonth = YearMonth.now(testClock)

        val editableMonths =
            properties.history.editableMonths

        return (editableMonths - 1 downTo 0)
            .map { offset ->
                currentMonth.minusMonths(offset.toLong()).toString()
            }
    }

    private fun createClientService(): ClientService {

        val properties = properties

        val transactionExecutor =
            FirestoreTransactionExecutor(firestore)

        val clientRepository =
            ClientRepository(firestore)

        val historyBucketRepository =
            ClientHistoryBucketRepository(
                firestore = firestore,
                properties = properties
            )

        val historyRepository =
            ClientHistoryRepository(firestore)

        val globalSummaryRepository =
            GlobalSummaryRepository(firestore)

        val summaryIndexBucketRepository =
            SummaryClientIndexBucketRepository(
                firestore = firestore,
                properties = properties
            )

        val summaryIndexRepository =
            SummaryClientIndexRepository(firestore)

        val clock = testClock

        return ClientService(
            firestore = firestore,
            transactionExecutor = transactionExecutor,
            clientRepository = clientRepository,
            historyBucketRepository = historyBucketRepository,
            historyRepository = historyRepository,
            globalSummaryRepository = globalSummaryRepository,
            summaryIndexBucketRepository = summaryIndexBucketRepository,
            summaryIndexRepository = summaryIndexRepository,
            properties = properties,
            clock = clock
        )
    }

    private fun findCurrentHistory(
        ownerId: String,
        client: Client
    ): com.clientledger.core.domain.ClientMonthlyHistory {

        val historyRepository =
            ClientHistoryRepository(firestore)

        val history = historyRepository.find(
            ownerId = ownerId,
            yearMonth = YearMonth.now(testClock).toString(),
            bucketId = client.bucketId,
            clientId = client.id
        )

        assertNotNull(history)

        return history!!
    }
}
