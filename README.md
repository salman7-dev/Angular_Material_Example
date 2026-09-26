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

BUILD FAILED in 26s
9 actionable tasks: 2 executed, 7 up-to-date
Configuration cache entry reused.
PS D:\New folder\client-ledger-codespace-main> 


validFirebaseEmulatorTokenAuthenticatesRequest()
java.lang.AssertionError: Status expected:<200> but was:<401>
	at org.springframework.test.util.AssertionErrors.fail(AssertionErrors.java:59)
	at org.springframework.test.util.AssertionErrors.assertEquals(AssertionErrors.java:122)
	at org.springframework.test.web.servlet.result.StatusResultMatchers.lambda$matcher$9(StatusResultMatchers.java:637)
	at org.springframework.test.web.servlet.MockMvc$1.andExpect(MockMvc.java:214)
	at com.clientledger.core.integration.security.FirebaseSecurityIntegrationTest.validFirebaseEmulatorTokenAuthenticatesRequest(FirebaseSecurityIntegrationTest.kt:65)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)

