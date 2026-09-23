PS D:\New folder\client-ledger-codespace-main> ./gradlew :app:test
Reusing configuration cache.
Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :app:test

ClientControllerIntegrationTest > createsSecondClientInExistingBuckets() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:120

ClientControllerIntegrationTest > updatingSameClientHistoryDoesNotIncreaseBucketSize() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:235

ClientControllerIntegrationTest > createsClientWithAdvanceOpeningBalance() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:305

ClientControllerIntegrationTest > createsNewBucketAfterCapacityReached() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:189

ClientControllerIntegrationTest > createsClientAndMaterializesEightMonths() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:62

ClientControllerIntegrationTest > createsClientWithSettledOpeningBalance() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:358

OwnerMonthInitializationServiceTest > createCurrentMonthFromPreviousMonth() FAILED
    org.opentest4j.AssertionFailedError at OwnerMonthInitializationServiceTest.kt:173

60 tests completed, 7 failed

> Task :app:test FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:test'.
> There were failing tests. See the report at: file:///D:/New%20folder/client-ledger-codespace-main/app/build/reports/tests/test/index.html

* Try:
> Run with --scan to get full insights.

BUILD FAILED in 20s
8 actionable tasks: 1 executed, 7 up-to-date
Configuration cache entry reused.
PS D:\New folder\client-ledger-codespace-main>                                                                                                                                      
                                                
