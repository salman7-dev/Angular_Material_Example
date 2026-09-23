<!DOCTYPE html>
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
<meta http-equiv="x-ua-compatible" content="IE=edge"/>
<title>Test results - OrderRepositoryTest</title>
<link href="../css/base-style.css" rel="stylesheet" type="text/css"/>
<link href="../css/style.css" rel="stylesheet" type="text/css"/>
<script src="../js/report.js" type="text/javascript"></script>
</head>
<body>
<div id="content">
<h1>OrderRepositoryTest</h1>
<div class="breadcrumbs">
<a href="../index.html">all</a> &gt; 
<a href="../packages/com.clientledger.core.repository.order.html">com.clientledger.core.repository.order</a> &gt; OrderRepositoryTest</div>
<div id="summary">
<table>
<tr>
<td>
<div class="summaryGroup">
<table>
<tr>
<td>
<div class="infoBox" id="tests">
<div class="counter">5</div>
<p>tests</p>
</div>
</td>
<td>
<div class="infoBox" id="failures">
<div class="counter">3</div>
<p>failures</p>
</div>
</td>
<td>
<div class="infoBox" id="ignored">
<div class="counter">0</div>
<p>ignored</p>
</div>
</td>
<td>
<div class="infoBox" id="duration">
<div class="counter">5.376s</div>
<p>duration</p>
</div>
</td>
</tr>
</table>
</div>
</td>
<td>
<div class="infoBox failures" id="successRate">
<div class="percent">40%</div>
<p>successful</p>
</div>
</td>
</tr>
</table>
</div>
<div id="tabs">
<ul class="tabLinks">
<li>
<a href="#tab0">Failed tests</a>
</li>
<li>
<a href="#tab1">Tests</a>
</li>
</ul>
<div id="tab0" class="tab">
<h2>Failed tests</h2>
<div class="test">
<a name="differentOwnersCanHaveSameOrderId()"></a>
<h3 class="failures">differentOwnersCanHaveSameOrderId()</h3>
<span class="code">
<pre>java.lang.RuntimeException: Could not deserialize object. Class com.clientledger.core.domain.Order does not define a no-argument constructor. If you are using ProGuard, make sure these constructors are not stripped
	at com.google.cloud.firestore.CustomClassMapper.deserializeError(CustomClassMapper.java:614)
	at com.google.cloud.firestore.CustomClassMapper.access$200(CustomClassMapper.java:53)
	at com.google.cloud.firestore.CustomClassMapper$BeanMapper.deserialize(CustomClassMapper.java:800)
	at com.google.cloud.firestore.CustomClassMapper$BeanMapper.deserialize(CustomClassMapper.java:792)
	at com.google.cloud.firestore.CustomClassMapper.convertBean(CustomClassMapper.java:593)
	at com.google.cloud.firestore.CustomClassMapper.deserializeToClass(CustomClassMapper.java:256)
	at com.google.cloud.firestore.CustomClassMapper.convertToCustomClass(CustomClassMapper.java:100)
	at com.google.cloud.firestore.DocumentSnapshot.toObject(DocumentSnapshot.java:189)
	at com.clientledger.core.repository.order.OrderRepository.findById(OrderRepository.kt:54)
	at com.clientledger.core.repository.order.OrderRepositoryTest.differentOwnersCanHaveSameOrderId(OrderRepositoryTest.kt:148)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
</pre>
</span>
</div>
<div class="test">
<a name="findsOrdersForClientInMonth()"></a>
<h3 class="failures">findsOrdersForClientInMonth()</h3>
<span class="code">
<pre>java.lang.RuntimeException: Could not deserialize object. Class com.clientledger.core.domain.Order does not define a no-argument constructor. If you are using ProGuard, make sure these constructors are not stripped
	at com.google.cloud.firestore.CustomClassMapper.deserializeError(CustomClassMapper.java:614)
	at com.google.cloud.firestore.CustomClassMapper.access$200(CustomClassMapper.java:53)
	at com.google.cloud.firestore.CustomClassMapper$BeanMapper.deserialize(CustomClassMapper.java:800)
	at com.google.cloud.firestore.CustomClassMapper$BeanMapper.deserialize(CustomClassMapper.java:792)
	at com.google.cloud.firestore.CustomClassMapper.convertBean(CustomClassMapper.java:593)
	at com.google.cloud.firestore.CustomClassMapper.deserializeToClass(CustomClassMapper.java:256)
	at com.google.cloud.firestore.CustomClassMapper.convertToCustomClass(CustomClassMapper.java:100)
	at com.google.cloud.firestore.QuerySnapshot.toObjects(QuerySnapshot.java:213)
	at com.clientledger.core.repository.order.OrderRepository.findAllForMonth(OrderRepository.kt:67)
	at com.clientledger.core.repository.order.OrderRepositoryTest.findsOrdersForClientInMonth(OrderRepositoryTest.kt:233)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
</pre>
</span>
</div>
<div class="test">
<a name="savesAndFindsOrderForOwner()"></a>
<h3 class="failures">savesAndFindsOrderForOwner()</h3>
<span class="code">
<pre>java.lang.RuntimeException: Could not deserialize object. Class com.clientledger.core.domain.Order does not define a no-argument constructor. If you are using ProGuard, make sure these constructors are not stripped
	at com.google.cloud.firestore.CustomClassMapper.deserializeError(CustomClassMapper.java:614)
	at com.google.cloud.firestore.CustomClassMapper.access$200(CustomClassMapper.java:53)
	at com.google.cloud.firestore.CustomClassMapper$BeanMapper.deserialize(CustomClassMapper.java:800)
	at com.google.cloud.firestore.CustomClassMapper$BeanMapper.deserialize(CustomClassMapper.java:792)
	at com.google.cloud.firestore.CustomClassMapper.convertBean(CustomClassMapper.java:593)
	at com.google.cloud.firestore.CustomClassMapper.deserializeToClass(CustomClassMapper.java:256)
	at com.google.cloud.firestore.CustomClassMapper.convertToCustomClass(CustomClassMapper.java:100)
	at com.google.cloud.firestore.DocumentSnapshot.toObject(DocumentSnapshot.java:189)
	at com.clientledger.core.repository.order.OrderRepository.findById(OrderRepository.kt:54)
	at com.clientledger.core.repository.order.OrderRepositoryTest.savesAndFindsOrderForOwner(OrderRepositoryTest.kt:99)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511)
</pre>
</span>
</div>
</div>
<div id="tab1" class="tab">
<h2>Tests</h2>
<table>
<thead>
<tr>
<th>Test</th>
<th>Duration</th>
<th>Result</th>
</tr>
</thead>
<tr>
<td class="success">deleteRemovesOnlyCurrentOwnersOrder()</td>
<td class="success">0.073s</td>
<td class="success">passed</td>
</tr>
<tr>
<td class="failures">differentOwnersCanHaveSameOrderId()</td>
<td class="failures">0.092s</td>
<td class="failures">failed</td>
</tr>
<tr>
<td class="success">findByIdDoesNotCrossOwnerBoundary()</td>
<td class="success">0.041s</td>
<td class="success">passed</td>
</tr>
<tr>
<td class="failures">findsOrdersForClientInMonth()</td>
<td class="failures">5.073s</td>
<td class="failures">failed</td>
</tr>
<tr>
<td class="failures">savesAndFindsOrderForOwner()</td>
<td class="failures">0.097s</td>
<td class="failures">failed</td>
</tr>
</table>
</div>
</div>
<div id="footer">
<p>
<div>
<label class="hidden" id="label-for-line-wrapping-toggle" for="line-wrapping-toggle">Wrap lines
<input id="line-wrapping-toggle" type="checkbox" autocomplete="off"/>
</label>
</div>Generated by 
<a href="http://www.gradle.org">Gradle 8.5</a> at 23-Sep-2026, 5:12:26 pm</p>
</div>
</div>
</body>
</html>
