createOwnerAssignsGeneralRole()
org.mockito.exceptions.misusing.InvalidUseOfMatchersException: 
Misplaced or misused argument matcher detected here:

-> at com.clientledger.core.service.owner.OwnerServiceTest.createOwnerPersistsOwner(OwnerServiceTest.kt:140)

You cannot use argument matchers outside of verification or stubbing.
Examples of correct usage of argument matchers:
    when(mock.get(anyInt())).thenReturn(null);
    doThrow(new RuntimeException()).when(mock).someVoidMethod(any());
    verify(mock).someMethod(contains("foo"))

This message may appear after an NullPointerException if the last matcher is returning an object 
like any() but the stubbed method signature expect a primitive argument, in this case,
use primitive alternatives.
    when(mock.get(any())); // bad use, will raise NPE
    when(mock.get(anyInt())); // correct usage use

Also, this error might show up because you use argument matchers with methods that cannot be mocked.
Following methods *cannot* be stubbed/verified: final/private/equals()/hashCode().
Mocking methods declared on non-public parent classes is not supported.

	at app//com.clientledger.core.service.owner.OwnerServiceTest.setUp(OwnerServiceTest.kt:34)
	at java.base@17.0.11/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)
createOwnerPersistsOwner()
java.lang.NullPointerException: any(...) must not be null
	at com.clientledger.core.service.owner.OwnerServiceTest.createOwnerPersistsOwner(OwnerServiceTest.kt:140)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
createOwnerRejectsBlankOwnerId()
org.mockito.exceptions.misusing.InvalidUseOfMatchersException: 
Misplaced or misused argument matcher detected here:

-> at com.clientledger.core.service.owner.OwnerServiceTest.createOwnerUsesFirebaseUidAsOwnerId(OwnerServiceTest.kt:65)

You cannot use argument matchers outside of verification or stubbing.
Examples of correct usage of argument matchers:
    when(mock.get(anyInt())).thenReturn(null);
    doThrow(new RuntimeException()).when(mock).someVoidMethod(any());
    verify(mock).someMethod(contains("foo"))

This message may appear after an NullPointerException if the last matcher is returning an object 
like any() but the stubbed method signature expect a primitive argument, in this case,
use primitive alternatives.
    when(mock.get(any())); // bad use, will raise NPE
    when(mock.get(anyInt())); // correct usage use

Also, this error might show up because you use argument matchers with methods that cannot be mocked.
Following methods *cannot* be stubbed/verified: final/private/equals()/hashCode().
Mocking methods declared on non-public parent classes is not supported.

	at app//com.clientledger.core.service.owner.OwnerServiceTest.setUp(OwnerServiceTest.kt:34)
	at java.base@17.0.11/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)
createOwnerUsesFirebaseUidAsOwnerId()
java.lang.NullPointerException: any(...) must not be null
	at com.clientledger.core.service.owner.OwnerServiceTest.createOwnerUsesFirebaseUidAsOwnerId(OwnerServiceTest.kt:65)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
