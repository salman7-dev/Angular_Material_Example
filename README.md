PS D:\New folder\client-ledger-codespace-main> .\gradlew.bat :app:test --tests "com.clientledger.core.integration.security.FirebaseSecurityIntegrationTest"
Reusing configuration cache.
Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :app:test

FirebaseSecurityIntegrationTest > protectedApiRejectsRequestWithoutBearerToken() FAILED
    java.lang.AssertionError at FirebaseSecurityIntegrationTest.kt:35

FirebaseSecurityIntegrationTest > protectedApiRejectsInvalidBearerToken() FAILED
    java.lang.AssertionError at FirebaseSecurityIntegrationTest.kt:55

3 tests completed, 2 failed

> Task :app:test FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:test'.
> There were failing tests. See the report at: file:///D:/New%20folder/client-ledger-codespace-main/app/build/reports/tests/test/index.html

* Try:
> Run with --scan to get full insights.

BUILD FAILED in 24s
9 actionable tasks: 3 executed, 6 up-to-date
Configuration cache entry reused.
PS D:\New folder\client-ledger-codespace-main> 



