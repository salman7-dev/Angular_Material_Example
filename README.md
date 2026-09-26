PS D:\New folder\client-ledger-codespace-main> .\gradlew.bat :app:compileKotlin
Reusing configuration cache.

> Task :app:compileKotlin FAILED
e: file:///D:/New%20folder/client-ledger-codespace-main/app/src/main/kotlin/com/clientledger/core/auth/CurrentOwnerResolver.kt:13:30 Unresolved reference 'enabled'.
e: file:///D:/New%20folder/client-ledger-codespace-main/app/src/main/kotlin/com/clientledger/core/auth/CurrentOwnerResolver.kt:14:48 Unresolved reference 'localOwnerId'.
e: file:///D:/New%20folder/client-ledger-codespace-main/app/src/main/kotlin/com/clientledger/core/auth/FirebaseAuthenticationFilter.kt:22:30 Unresolved reference 'enabled'.       

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:compileKotlin'.
> A failure occurred while executing org.jetbrains.kotlin.compilerRunner.GradleCompilerRunnerWithWorkers$GradleKotlinCompilerWorkAction
   > Compilation error. See log for more details

* Try:
> Run with --stacktrace option to get the stack trace.
> Run with --info or --debug option to get more log output.
> Run with --scan to get full insights.
> Get more help at https://help.gradle.org.

BUILD FAILED in 11s
3 actionable tasks: 1 executed, 2 up-to-date
Configuration cache entry reused.
PS D:\New folder\client-ledger-codespace-main>
