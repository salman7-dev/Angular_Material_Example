
SummaryIntegrationTest > getReceivableClientsReturnsPaginatedClients() FAILED
    org.opentest4j.AssertionFailedError at SummaryIntegrationTest.kt:354

SummaryIntegrationTest > getAdvanceClientsReturnsPaginatedClients() FAILED
    org.opentest4j.AssertionFailedError at SummaryIntegrationTest.kt:444

5 tests completed, 2 failed
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
