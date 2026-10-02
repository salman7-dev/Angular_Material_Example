allocateFirstClientUsesBucket000()
org.opentest4j.AssertionFailedError: expected: java.lang.Integer@37f60cd4<100> but was: java.lang.Long@4cf46574<100>
	at org.junit.jupiter.api.AssertionFailureBuilder.build(AssertionFailureBuilder.java:151)
	at org.junit.jupiter.api.AssertionFailureBuilder.buildAndThrow(AssertionFailureBuilder.java:132)
	at org.junit.jupiter.api.AssertEquals.failNotEqual(AssertEquals.java:197)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:182)
	at org.junit.jupiter.api.AssertEquals.assertEquals(AssertEquals.java:177)
	at org.junit.jupiter.api.Assertions.assertEquals(Assertions.java:1145)
	at com.clientledger.core.repository.summary.SummaryClientIndexBucketRepositoryTest.allocateFirstClientUsesBucket000(SummaryClientIndexBucketRepositoryTest.kt:75)
	at java.base/java.lang.reflect.Method.invoke(Method.java:580)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1596)

    > Task :test

SummaryClientIndexBucketRepositoryTest > allocateFirstClientUsesBucket000() FAILED
    org.opentest4j.AssertionFailedError at SummaryClientIndexBucketRepositoryTest.kt:75
