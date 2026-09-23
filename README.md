<!DOCTYPE html>
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
<meta http-equiv="x-ua-compatible" content="IE=edge"/>
<title>Test results - ClientControllerIntegrationTest</title>
<link href="../css/base-style.css" rel="stylesheet" type="text/css"/>
<link href="../css/style.css" rel="stylesheet" type="text/css"/>
<script src="../js/report.js" type="text/javascript"></script>
</head>
<body>
<div id="content">
<h1>ClientControllerIntegrationTest</h1>
<div class="breadcrumbs">
<a href="../index.html">all</a> &gt; 
<a href="../packages/com.clientledger.core.controller.html">com.clientledger.core.controller</a> &gt; ClientControllerIntegrationTest</div>
<div id="summary">
<table>
<tr>
<td>
<div class="summaryGroup">
<table>
<tr>
<td>
<div class="infoBox" id="tests">
<div class="counter">1</div>
<p>tests</p>
</div>
</td>
<td>
<div class="infoBox" id="failures">
<div class="counter">1</div>
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
<div class="counter">2.255s</div>
<p>duration</p>
</div>
</td>
</tr>
</table>
</div>
</td>
<td>
<div class="infoBox failures" id="successRate">
<div class="percent">0%</div>
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
<li>
<a href="#tab2">Standard output</a>
</li>
</ul>
<div id="tab0" class="tab">
<h2>Failed tests</h2>
<div class="test">
<a name="createsClientAndMaterializesEightMonths()"></a>
<h3 class="failures">createsClientAndMaterializesEightMonths()</h3>
<span class="code">
<pre>com.fasterxml.jackson.module.kotlin.MissingKotlinParameterException: Instantiation of [simple type, class com.clientledger.core.domain.Client] value failed for JSON property name due to missing (therefore NULL) value for creator parameter name which is a non-nullable type
 at [Source: (String)&quot;{&quot;success&quot;:false,&quot;message&quot;:&quot;An unexpected error occurred&quot;}&quot;; line: 1, column: 58] (through reference chain: com.clientledger.core.domain.Client[&quot;name&quot;])
	at app//com.fasterxml.jackson.module.kotlin.KotlinValueInstantiator.createFromObjectWith(KotlinValueInstantiator.kt:84)
	at app//com.fasterxml.jackson.databind.deser.impl.PropertyBasedCreator.build(PropertyBasedCreator.java:202)
	at app//com.fasterxml.jackson.databind.deser.BeanDeserializer._deserializeUsingPropertyBased(BeanDeserializer.java:525)
	at app//com.fasterxml.jackson.databind.deser.BeanDeserializerBase.deserializeFromObjectUsingNonDefault(BeanDeserializerBase.java:1409)
	at app//com.fasterxml.jackson.databind.deser.BeanDeserializer.deserializeFromObject(BeanDeserializer.java:352)
	at app//com.fasterxml.jackson.databind.deser.BeanDeserializer.deserialize(BeanDeserializer.java:185)
	at app//com.fasterxml.jackson.databind.deser.DefaultDeserializationContext.readRootValue(DefaultDeserializationContext.java:323)
	at app//com.fasterxml.jackson.databind.ObjectMapper._readMapAndClose(ObjectMapper.java:4825)
	at app//com.fasterxml.jackson.databind.ObjectMapper.readValue(ObjectMapper.java:3772)
	at app//com.fasterxml.jackson.databind.ObjectMapper.readValue(ObjectMapper.java:3740)
	at app//com.clientledger.core.controller.ClientControllerIntegrationTest.createsClientAndMaterializesEightMonths(ClientControllerIntegrationTest.kt:69)
	at java.base@17.0.11/java.lang.reflect.Method.invoke(Method.java:568)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)
	at java.base@17.0.11/java.util.ArrayList.forEach(ArrayList.java:1511)
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
<td class="failures">createsClientAndMaterializesEightMonths()</td>
<td class="failures">2.255s</td>
<td class="failures">failed</td>
</tr>
</table>
</div>
<div id="tab2" class="tab">
<h2>Standard output</h2>
<span class="code">
<pre>Standard Commons Logging discovery in action with spring-jcl: please remove commons-logging.jar from classpath in order to avoid potential conflicts

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.0)

2026-09-23T14:53:37.347+05:30  INFO 6120 --- [    Test worker] c.c.c.c.ClientControllerIntegrationTest  : Starting ClientControllerIntegrationTest using Java 17.0.11 with PID 6120 (started by khans in D:\New folder\client-ledger-codespace-main\app)
2026-09-23T14:53:37.352+05:30  INFO 6120 --- [    Test worker] c.c.c.c.ClientControllerIntegrationTest  : No active profile set, falling back to 1 default profile: &quot;default&quot;
2026-09-23T14:53:40.645+05:30  INFO 6120 --- [    Test worker] o.s.b.t.m.w.SpringBootMockServletContext : Initializing Spring TestDispatcherServlet ''
2026-09-23T14:53:40.646+05:30  INFO 6120 --- [    Test worker] o.s.t.web.servlet.TestDispatcherServlet  : Initializing Servlet ''
2026-09-23T14:53:40.650+05:30  INFO 6120 --- [    Test worker] o.s.t.web.servlet.TestDispatcherServlet  : Completed initialization in 2 ms
2026-09-23T14:53:40.701+05:30  INFO 6120 --- [    Test worker] c.c.c.c.ClientControllerIntegrationTest  : Started ClientControllerIntegrationTest in 3.802 seconds (process running for 5.711)
2026-09-23T14:53:42.916+05:30 ERROR 6120 --- [    Test worker] c.c.c.exception.GlobalExceptionHandler   : Unexpected ledger error. clientId=N/A, message=java.lang.IllegalStateException: Bucket bucket_000 is full

java.util.concurrent.ExecutionException: java.lang.IllegalStateException: Bucket bucket_000 is full
	at com.google.common.util.concurrent.AbstractFuture.getDoneValue(AbstractFuture.java:588) ~[guava-31.1-jre.jar:na]
	at com.google.common.util.concurrent.AbstractFuture.get(AbstractFuture.java:567) ~[guava-31.1-jre.jar:na]
	at com.google.common.util.concurrent.FluentFuture$TrustedFuture.get(FluentFuture.java:91) ~[guava-31.1-jre.jar:na]
	at com.google.common.util.concurrent.ForwardingFuture.get(ForwardingFuture.java:66) ~[guava-31.1-jre.jar:na]
	at com.clientledger.core.repository.history.ClientMonthlyHistoryRepository.save(ClientMonthlyHistoryRepository.kt:79) ~[main/:na]
	at com.clientledger.core.service.ClientMonthlyHistoryMaterializationService.materializeForNewClient(ClientMonthlyHistoryMaterializationService.kt:40) ~[main/:na]
	at com.clientledger.core.service.ClientService.createClient(ClientService.kt:50) ~[main/:na]
	at com.clientledger.core.controller.ClientController.createClient(ClientController.kt:23) ~[main/:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77) ~[na:na]
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
	at java.base/java.lang.reflect.Method.invoke(Method.java:568) ~[na:na]
	at kotlin.reflect.jvm.internal.calls.CallerImpl$Method.callMethod(CallerImpl.kt:97) ~[kotlin-reflect-2.0.0.jar:2.0.0-release-341]
	at kotlin.reflect.jvm.internal.calls.CallerImpl$Method$Instance.call(CallerImpl.kt:113) ~[kotlin-reflect-2.0.0.jar:2.0.0-release-341]
	at kotlin.reflect.jvm.internal.KCallableImpl.callDefaultMethod$kotlin_reflection(KCallableImpl.kt:207) ~[kotlin-reflect-2.0.0.jar:2.0.0-release-341]
	at kotlin.reflect.jvm.internal.KCallableImpl.callBy(KCallableImpl.kt:112) ~[kotlin-reflect-2.0.0.jar:2.0.0-release-341]
	at org.springframework.web.method.support.InvocableHandlerMethod$KotlinDelegate.invokeFunction(InvocableHandlerMethod.java:319) ~[spring-web-6.1.1.jar:6.1.1]
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:251) ~[spring-web-6.1.1.jar:6.1.1]
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:182) ~[spring-web-6.1.1.jar:6.1.1]
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:118) ~[spring-webmvc-6.1.1.jar:6.1.1]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:917) ~[spring-webmvc-6.1.1.jar:6.1.1]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:829) ~[spring-webmvc-6.1.1.jar:6.1.1]
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87) ~[spring-webmvc-6.1.1.jar:6.1.1]
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1089) ~[spring-webmvc-6.1.1.jar:6.1.1]
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:979) ~[spring-webmvc-6.1.1.jar:6.1.1]
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1014) ~[spring-webmvc-6.1.1.jar:6.1.1]
	at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:914) ~[spring-webmvc-6.1.1.jar:6.1.1]
	at jakarta.servlet.http.HttpServlet.service(HttpServlet.java:590) ~[tomcat-embed-core-10.1.16.jar:6.0]
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:885) ~[spring-webmvc-6.1.1.jar:6.1.1]
	at org.springframework.test.web.servlet.TestDispatcherServlet.service(TestDispatcherServlet.java:72) ~[spring-test-6.1.1.jar:6.1.1]
	at jakarta.servlet.http.HttpServlet.service(HttpServlet.java:658) ~[tomcat-embed-core-10.1.16.jar:6.0]
	at org.springframework.mock.web.MockFilterChain$ServletFilterProxy.doFilter(MockFilterChain.java:165) ~[spring-test-6.1.1.jar:6.1.1]
	at org.springframework.mock.web.MockFilterChain.doFilter(MockFilterChain.java:132) ~[spring-test-6.1.1.jar:6.1.1]
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100) ~[spring-web-6.1.1.jar:6.1.1]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.1.jar:6.1.1]
	at org.springframework.test.web.servlet.setup.MockMvcFilterDecorator.doFilter(MockMvcFilterDecorator.java:151) ~[spring-test-6.1.1.jar:6.1.1]
	at org.springframework.mock.web.MockFilterChain.doFilter(MockFilterChain.java:132) ~[spring-test-6.1.1.jar:6.1.1]
	at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93) ~[spring-web-6.1.1.jar:6.1.1]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.1.jar:6.1.1]
	at org.springframework.test.web.servlet.setup.MockMvcFilterDecorator.doFilter(MockMvcFilterDecorator.java:151) ~[spring-test-6.1.1.jar:6.1.1]
	at org.springframework.mock.web.MockFilterChain.doFilter(MockFilterChain.java:132) ~[spring-test-6.1.1.jar:6.1.1]
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201) ~[spring-web-6.1.1.jar:6.1.1]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.1.jar:6.1.1]
	at org.springframework.test.web.servlet.setup.MockMvcFilterDecorator.doFilter(MockMvcFilterDecorator.java:151) ~[spring-test-6.1.1.jar:6.1.1]
	at org.springframework.mock.web.MockFilterChain.doFilter(MockFilterChain.java:132) ~[spring-test-6.1.1.jar:6.1.1]
	at org.springframework.test.web.servlet.MockMvc.perform(MockMvc.java:201) ~[spring-test-6.1.1.jar:6.1.1]
	at com.clientledger.core.controller.ClientControllerIntegrationTest.createsClientAndMaterializesEightMonths(ClientControllerIntegrationTest.kt:58) ~[test/:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77) ~[na:na]
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
	at java.base/java.lang.reflect.Method.invoke(Method.java:568) ~[na:na]
	at org.junit.platform.commons.util.ReflectionUtils.invokeMethod(ReflectionUtils.java:728) ~[junit-platform-commons-1.10.1.jar:1.10.1]
	at org.junit.jupiter.engine.execution.MethodInvocation.proceed(MethodInvocation.java:60) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain$ValidatingInvocation.proceed(InvocationInterceptorChain.java:131) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.extension.TimeoutExtension.intercept(TimeoutExtension.java:156) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.extension.TimeoutExtension.interceptTestableMethod(TimeoutExtension.java:147) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.extension.TimeoutExtension.interceptTestMethod(TimeoutExtension.java:86) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.execution.InterceptingExecutableInvoker$ReflectiveInterceptorCall.lambda$ofVoidMethod$0(InterceptingExecutableInvoker.java:103) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.execution.InterceptingExecutableInvoker.lambda$invoke$0(InterceptingExecutableInvoker.java:93) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain$InterceptedInvocation.proceed(InvocationInterceptorChain.java:106) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain.proceed(InvocationInterceptorChain.java:64) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain.chainAndInvoke(InvocationInterceptorChain.java:45) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain.invoke(InvocationInterceptorChain.java:37) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.execution.InterceptingExecutableInvoker.invoke(InterceptingExecutableInvoker.java:92) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.execution.InterceptingExecutableInvoker.invoke(InterceptingExecutableInvoker.java:86) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.lambda$invokeTestMethod$7(TestMethodTestDescriptor.java:218) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.invokeTestMethod(TestMethodTestDescriptor.java:214) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.execute(TestMethodTestDescriptor.java:139) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.execute(TestMethodTestDescriptor.java:69) ~[junit-jupiter-engine-5.10.0.jar:5.10.0]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$6(NodeTestTask.java:151) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:141) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$9(NodeTestTask.java:139) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:138) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:95) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511) ~[na:na]
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.invokeAll(SameThreadHierarchicalTestExecutorService.java:41) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$6(NodeTestTask.java:155) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:141) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$9(NodeTestTask.java:139) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:138) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:95) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at java.base/java.util.ArrayList.forEach(ArrayList.java:1511) ~[na:na]
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.invokeAll(SameThreadHierarchicalTestExecutorService.java:41) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$6(NodeTestTask.java:155) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:141) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$9(NodeTestTask.java:139) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:138) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:95) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.submit(SameThreadHierarchicalTestExecutorService.java:35) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.HierarchicalTestExecutor.execute(HierarchicalTestExecutor.java:57) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.engine.support.hierarchical.HierarchicalTestEngine.execute(HierarchicalTestEngine.java:54) ~[junit-platform-engine-1.10.1.jar:1.10.1]
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:198) ~[junit-platform-launcher-1.10.1.jar:1.10.1]
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:169) ~[junit-platform-launcher-1.10.1.jar:1.10.1]
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:93) ~[junit-platform-launcher-1.10.1.jar:1.10.1]
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.lambda$execute$0(EngineExecutionOrchestrator.java:58) ~[junit-platform-launcher-1.10.1.jar:1.10.1]
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.withInterceptedStreams(EngineExecutionOrchestrator.java:141) ~[junit-platform-launcher-1.10.1.jar:1.10.1]
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:57) ~[junit-platform-launcher-1.10.1.jar:1.10.1]
	at org.junit.platform.launcher.core.DefaultLauncher.execute(DefaultLauncher.java:103) ~[junit-platform-launcher-1.10.1.jar:1.10.1]
	at org.junit.platform.launcher.core.DefaultLauncher.execute(DefaultLauncher.java:85) ~[junit-platform-launcher-1.10.1.jar:1.10.1]
	at org.junit.platform.launcher.core.DelegatingLauncher.execute(DelegatingLauncher.java:47) ~[junit-platform-launcher-1.10.1.jar:1.10.1]
	at org.gradle.api.internal.tasks.testing.junitplatform.JUnitPlatformTestClassProcessor$CollectAllTestClassesExecutor.processAllTestClasses(JUnitPlatformTestClassProcessor.java:119) ~[na:na]
	at org.gradle.api.internal.tasks.testing.junitplatform.JUnitPlatformTestClassProcessor$CollectAllTestClassesExecutor.access$000(JUnitPlatformTestClassProcessor.java:94) ~[na:na]
	at org.gradle.api.internal.tasks.testing.junitplatform.JUnitPlatformTestClassProcessor.stop(JUnitPlatformTestClassProcessor.java:89) ~[na:na]
	at org.gradle.api.internal.tasks.testing.SuiteTestClassProcessor.stop(SuiteTestClassProcessor.java:62) ~[na:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:na]
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77) ~[na:na]
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:na]
	at java.base/java.lang.reflect.Method.invoke(Method.java:568) ~[na:na]
	at org.gradle.internal.dispatch.ReflectionDispatch.dispatch(ReflectionDispatch.java:36) ~[na:na]
	at org.gradle.internal.dispatch.ReflectionDispatch.dispatch(ReflectionDispatch.java:24) ~[na:na]
	at org.gradle.internal.dispatch.ContextClassLoaderDispatch.dispatch(ContextClassLoaderDispatch.java:33) ~[na:na]
	at org.gradle.internal.dispatch.ProxyDispatchAdapter$DispatchingInvocationHandler.invoke(ProxyDispatchAdapter.java:94) ~[na:na]
	at jdk.proxy1/jdk.proxy1.$Proxy2.stop(Unknown Source) ~[na:na]
	at org.gradle.api.internal.tasks.testing.worker.TestWorker$3.run(TestWorker.java:193) ~[na:na]
	at org.gradle.api.internal.tasks.testing.worker.TestWorker.executeAndMaintainThreadName(TestWorker.java:129) ~[na:na]
	at org.gradle.api.internal.tasks.testing.worker.TestWorker.execute(TestWorker.java:100) ~[na:na]
	at org.gradle.api.internal.tasks.testing.worker.TestWorker.execute(TestWorker.java:60) ~[na:na]
	at org.gradle.process.internal.worker.child.ActionExecutionWorker.execute(ActionExecutionWorker.java:56) ~[na:na]
	at org.gradle.process.internal.worker.child.SystemApplicationClassLoaderWorker.call(SystemApplicationClassLoaderWorker.java:113) ~[na:na]
	at org.gradle.process.internal.worker.child.SystemApplicationClassLoaderWorker.call(SystemApplicationClassLoaderWorker.java:65) ~[na:na]
	at worker.org.gradle.process.internal.worker.GradleWorkerMain.run(GradleWorkerMain.java:69) ~[gradle-worker.jar:na]
	at worker.org.gradle.process.internal.worker.GradleWorkerMain.main(GradleWorkerMain.java:74) ~[gradle-worker.jar:na]
Caused by: java.lang.IllegalStateException: Bucket bucket_000 is full
	at com.clientledger.core.repository.history.ClientMonthlyHistoryRepository.save$lambda$0(ClientMonthlyHistoryRepository.kt:66) ~[main/:na]
	at com.google.cloud.firestore.FirestoreImpl$TransactionAsyncAdapter.updateCallback(FirestoreImpl.java:496) ~[google-cloud-firestore-3.13.0.jar:3.13.0]
	at com.google.cloud.firestore.TransactionRunner.lambda$invokeUserCallback$0(TransactionRunner.java:151) ~[google-cloud-firestore-3.13.0.jar:3.13.0]
	at io.grpc.Context$1.run(Context.java:566) ~[grpc-context-1.55.1.jar:1.55.1]
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:539) ~[na:na]
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264) ~[na:na]
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304) ~[na:na]
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136) ~[na:na]
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635) ~[na:na]
	at java.base/java.lang.Thread.run(Thread.java:842) ~[na:na]

STATUS = 500
BODY = {&quot;success&quot;:false,&quot;message&quot;:&quot;An unexpected error occurred&quot;}

MockHttpServletRequest:
      HTTP Method = POST
      Request URI = /api/clients
       Parameters = {}
          Headers = [Content-Type:&quot;application/json;charset=UTF-8&quot;, Content-Length:&quot;215&quot;]
             Body = {&quot;name&quot;:&quot;ABC Traders&quot;,&quot;phone&quot;:&quot;9876543210&quot;,&quot;email&quot;:&quot;abc@example.com&quot;,&quot;gstNumber&quot;:&quot;27ABCDE1234F1Z5&quot;,&quot;address&quot;:{&quot;line1&quot;:&quot;Shop 10&quot;,&quot;city&quot;:&quot;Mumbai&quot;,&quot;state&quot;:&quot;Maharashtra&quot;,&quot;pinCode&quot;:&quot;400001&quot;},&quot;initialOpeningBalance&quot;:5000}
    Session Attrs = {}

Handler:
             Type = com.clientledger.core.controller.ClientController
           Method = com.clientledger.core.controller.ClientController#createClient(CreateClientRequest)

Async:
    Async started = false
     Async result = null

Resolved Exception:
             Type = java.util.concurrent.ExecutionException

ModelAndView:
        View name = null
             View = null
            Model = null

FlashMap:
       Attributes = null

MockHttpServletResponse:
           Status = 500
    Error message = null
          Headers = [Content-Type:&quot;application/json&quot;]
     Content type = application/json
             Body = {&quot;success&quot;:false,&quot;message&quot;:&quot;An unexpected error occurred&quot;}
    Forwarded URL = null
   Redirected URL = null
          Cookies = []
</pre>
</span>
</div>
</div>
<div id="footer">
<p>
<div>
<label class="hidden" id="label-for-line-wrapping-toggle" for="line-wrapping-toggle">Wrap lines
<input id="line-wrapping-toggle" type="checkbox" autocomplete="off"/>
</label>
</div>Generated by 
<a href="http://www.gradle.org">Gradle 8.5</a> at 23-Sep-2026, 2:53:43 pm</p>
</div>
</div>
</body>
</html>
