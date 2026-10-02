ClientCreateIntegrationTest > createClientCreatesConsistentDataAcrossAllCollections() FAILED
    org.opentest4j.AssertionFailedError at ClientCreateIntegrationTest.kt:238
org.opentest4j.AssertionFailedError: expected: java.lang.Long@48cbb4c5<100> but was: java.lang.Integer@af04d6d<100>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:182)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:177)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:1145)
	at com.clientledger.core.integration.client.ClientCreateIntegrationTest.verifyHistoryBucket(ClientCreateIntegrationTest.kt:238)
	at com.clientledger.core.integration.client.ClientCreateIntegrationTest.createClientCreatesConsistentDataAcrossAllCollections(ClientCreateIntegrationTest.kt:93)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
