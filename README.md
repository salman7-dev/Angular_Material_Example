PS D:\New folder\client-ledger-codespace-main> .\gradlew.bat :app:bootRun                        
Starting a Gradle Daemon, 2 incompatible and 1 stopped Daemons could not be reused, use --status for details
Calculating task graph as no cached configuration is available for tasks: :app:bootRun

> Task :app:bootRun FAILED
Error: Could not find or load main class com.clientledger.core.app.ClientLedgerApplicationKt
Caused by: java.lang.ClassNotFoundException: com.clientledger.core.app.ClientLedgerApplicationKt

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:bootRun'.
> Process 'command 'C:\Program Files\Java\jdk-17\bin\java.exe'' finished with non-zero exit value 1

* Try:
> Run with --stacktrace option to get the stack trace.
> Run with --info or --debug option to get more log output.
> Run with --scan to get full insights.
> Get more help at https://help.gradle.org.

BUILD FAILED in 2m 31s
18 actionable tasks: 14 executed, 4 up-to-date
Configuration cache entry stored.
PS D:\New folder\client-ledger-codespace-main> 
