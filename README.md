Standard Commons Logging discovery in action with spring-jcl: please remove commons-logging.jar from classpath in order to avoid potential conflicts

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.0)

2026-09-23T16:10:17.141+05:30  INFO 25784 --- [    Test worker] c.c.c.c.ClientControllerIntegrationTest  : Starting ClientControllerIntegrationTest using Java 17.0.11 with PID 25784 (started by khans in D:\New folder\client-ledger-codespace-main\app)
2026-09-23T16:10:17.147+05:30  INFO 25784 --- [    Test worker] c.c.c.c.ClientControllerIntegrationTest  : No active profile set, falling back to 1 default profile: "default"
2026-09-23T16:10:23.829+05:30  INFO 25784 --- [    Test worker] o.s.b.t.m.w.SpringBootMockServletContext : Initializing Spring TestDispatcherServlet ''
2026-09-23T16:10:23.830+05:30  INFO 25784 --- [    Test worker] o.s.t.web.servlet.TestDispatcherServlet  : Initializing Servlet ''
2026-09-23T16:10:23.834+05:30  INFO 25784 --- [    Test worker] o.s.t.web.servlet.TestDispatcherServlet  : Completed initialization in 2 ms
2026-09-23T16:10:23.892+05:30  INFO 25784 --- [    Test worker] c.c.c.c.ClientControllerIntegrationTest  : Started ClientControllerIntegrationTest in 7.513 seconds (process running for 10.746)
CLIENT COUNT BEFORE CLEANUP = 1
HISTORY BUCKETS BEFORE CLEANUP = [bucket_000]
CLIENT COUNT AFTER CLEANUP = 0
HISTORY BUCKETS AFTER CLEANUP = [bucket_000]
BUCKET EXISTS AFTER CLEANUP = true
BUCKET DATA AFTER CLEANUP = {size=2, capacity=50}
CLIENT HISTORY COUNT AFTER CLEANUP = 2
CLIENT HISTORY IDS AFTER CLEANUP = [CLI-001, CLI-002]
2026-09-23T16:10:30.251+05:30 ERROR 25784 --- [    Test worker] c.c.c.exception.GlobalExceptionHandler   : Unexpected ledger error. clientId=N/A, message=java.lang.IllegalStateException: Bucket bucket_000 is full

java.util.concurrent.ExecutionException: java.lang.IllegalStateException: Bucket bucket_000 is full
	at com.google.common.util.concurrent.AbstractFuture.getDoneValue(AbstractFuture.java:588) ~[guava-31.1-jre.jar:na]
	at com.google.common.util.concurrent.AbstractFuture.get(AbstractFuture.java:567) ~[guava-31.1-jre.jar:na]
	at com.google.common.util.concurrent.FluentFuture$TrustedFuture.get(FluentFuture.java:91) ~[guava-31.1-jre.jar:na]
	at com.google.common.util.concurrent.ForwardingFuture.get(ForwardingFuture.java:66) ~[guava-31.1-jre.jar:na]
	at com.clientledger.core.repository.history.ClientMonthlyHistoryRepository.save(ClientMonthlyHistoryRepository.kt:122) ~[main/:na]
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
	at com.clientledger.core.controller.ClientControllerIntegrationTest.createsClientAndMaterializesEightMonths(ClientControllerIntegrationTest.kt:176) ~[test/:na]
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
	at com.clientledger.core.repository.history.ClientMonthlyHistoryRepository.save$lambda$2(ClientMonthlyHistoryRepository.kt:109) ~[main/:na]
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
BODY = {"success":false,"message":"An unexpected error occurred"}

MockHttpServletRequest:
      HTTP Method = POST
      Request URI = /api/clients
       Parameters = {}
          Headers = [Content-Type:"application/json;charset=UTF-8", Content-Length:"215"]
             Body = {"name":"ABC Traders","phone":"9876543210","email":"abc@example.com","gstNumber":"27ABCDE1234F1Z5","address":{"line1":"Shop 10","city":"Mumbai","state":"Maharashtra","pinCode":"400001"},"initialOpeningBalance":5000}
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
          Headers = [Content-Type:"application/json"]
     Content type = application/json
             Body = {"success":false,"message":"An unexpected error occurred"}
    Forwarded URL = null
   Redirected URL = null
          Cookies = []
