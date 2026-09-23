PS D:\New folder\client-ledger-codespace-main> ./gradlew :app:test --tests "com.clientledger.core.controller.ClientControllerIntegrationTest.createsClientAndMaterializesEightMonths"
Reusing configuration cache.
Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :app:test

ClientControllerIntegrationTest > createsClientAndMaterializesEightMonths() FAILED
    com.fasterxml.jackson.module.kotlin.MissingKotlinParameterException at ClientControllerIntegrationTest.kt:66

1 test completed, 1 failed

> Task :app:test FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:test'.
> There were failing tests. See the report at: file:///D:/New%20folder/client-ledger-codespace-main/app/build/reports/tests/test/index.html

* Try:
> Run with --scan to get full insights.

BUILD FAILED in 16s
8 actionable tasks: 2 executed, 6 up-to-date
Configuration cache entry reused.
PS D:\New folder\client-ledger-codespace-main>   
