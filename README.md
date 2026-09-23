PS D:\New folder\client-ledger-codespace-main> ./gradlew :app:test
 Reusing configuration cache.
Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :app:test

ClientControllerIntegrationTest > createsSecondClientInExistingBuckets() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:176

ClientControllerIntegrationTest > updatingSameClientHistoryDoesNotIncreaseBucketSize() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:291

ClientControllerIntegrationTest > createsClientWithAdvanceOpeningBalance() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:361                                                                                                              

ClientControllerIntegrationTest > createsNewBucketAfterCapacityReached() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:245                                                                                                              

ClientControllerIntegrationTest > createsClientAndMaterializesEightMonths() FAILED
    com.fasterxml.jackson.module.kotlin.MissingKotlinParameterException at ClientControllerIntegrationTest.kt:122                                                                   

ClientControllerIntegrationTest > createsClientWithSettledOpeningBalance() FAILED
    java.lang.AssertionError at ClientControllerIntegrationTest.kt:414                                                                                                              

Failed to map supported failure 'org.opentest4j.AssertionFailedError: expected: <10500> but was: <null>' with mapper 'org.gradle.api.internal.tasks.testing.failure.mappers.OpenTestAssertionFailedMapper@556944cd': Cannot invoke "Object.getClass()" because "actualValue" is null                                                                                    

> Task :app:test

OwnerMonthInitializationServiceTest > createCurrentMonthFromPreviousMonth() FAILED
    org.opentest4j.AssertionFailedError at OwnerMonthInitializationServiceTest.kt:181

OwnerMonthInitializationServiceTest > doNothingWhenPreviousMonthHasNoData() FAILED
    org.opentest4j.AssertionFailedError at OwnerMonthInitializationServiceTest.kt:205

60 tests completed, 8 failed

> Task :app:test FAILED

FAILURE: Build failed with an exception.                                                                                                                                            

* What went wrong:                                                                                                                                                                  
Execution failed for task ':app:test'.
> There were failing tests. See the report at: file:///D:/New%20folder/client-ledger-codespace-main/app/build/reports/tests/test/index.html

* Try:
> Run with --scan to get full insights.

BUILD FAILED in 34s
8 actionable tasks: 2 executed, 6 up-to-date
Configuration cache entry reused.
PS D:\New folder\client-ledger-codespace-main>
