PS D:\New folder\client-ledger-codespace-main> firebase use

Error: Failed to authenticate, have you run firebase login?
PS D:\New folder\client-ledger-codespace-main> Get-Content .firebaserc
Get-Content : Cannot find path 'D:\New folder\client-ledger-codespace-main\.firebaserc' because it does not exist.
At line:1 char:1
+ Get-Content .firebaserc
+ ~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (D:\New folder\c...ain\.firebaserc:String) [Get-Content], ItemNotFoundException
    + FullyQualifiedErrorId : PathNotFound,Microsoft.PowerShell.Commands.GetContentCommand
 
PS D:\New folder\client-ledger-codespace-main> firebase use           

Error: Failed to authenticate, have you run firebase login?
PS D:\New folder\client-ledger-codespace-main> .\gradlew.bat :app:test --tests "com.clientledger.core.integration.security.FirebaseSecurityIntegrationTest"
Reusing configuration cache.
Java HotSpot(TM) 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended

> Task :app:test

FirebaseSecurityIntegrationTest > validFirebaseEmulatorTokenAuthenticatesRequest() FAILED
    java.lang.AssertionError at FirebaseSecurityIntegrationTest.kt:65

3 tests completed, 1 failed

> Task :app:test FAILED

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:test'.
> There were failing tests. See the report at: file:///D:/New%20folder/client-ledger-codespace-main/app/build/reports/tests/test/index.html

* Try:
> Run with --scan to get full insights.

BUILD FAILED in 19s
9 actionable tasks: 1 executed, 8 up-to-date
Configuration cache entry reused.
PS D:\New folder\client-ledger-codespace-main> Get-Content .firebaserc
Get-Content : Cannot find path 'D:\New folder\client-ledger-codespace-main\.firebaserc' because it does not exist.
At line:1 char:1
+ Get-Content .firebaserc
+ ~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (D:\New folder\c...ain\.firebaserc:String) [Get-Content], ItemNotFoundException
    + FullyQualifiedErrorId : PathNotFound,Microsoft.PowerShell.Commands.GetContentCommand

PS D:\New folder\client-ledger-codespace-main>
