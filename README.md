PS D:\New folder\client-ledger-codespace-main> .\gradlew.bat :app:bootRun --no-daemon
To honour the JVM settings for this build a single-use Daemon process will be forked. For more on this, please refer to https://docs.gradle.org/8.5/userguide/gradle_daemon.html#sec:disabling_the_daemon in the Gradle documentation.
Daemon will be stopped at the end of the build 
Calculating task graph as configuration cache cannot be reused because environment variable 'FIREBASE_AUTH_EMULATOR_HOST' has changed.

> Task :app:bootRun
Standard Commons Logging discovery in action with spring-jcl: please remove commons-logging.jar from classpath in order to avoid potential conflicts

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.0)

2026-09-27T14:58:47.128+05:30  INFO 28436 --- [           main] c.c.core.app.ClientLedgerApplicationKt   : Starting ClientLedgerApplicationKt using Java 17.0.11 with PID 28436 (D:\New folder\client-ledger-codespace-main\app\build\classes\kotlin\main started by khans in D:\New folder\client-ledger-codespace-main\app)
2026-09-27T14:58:47.128+05:30  INFO 28436 --- [           main] c.c.core.app.ClientLedgerApplicationKt   : No active profile set, falling back to 1 default profile: "default"
2026-09-27T14:58:49.529+05:30  INFO 28436 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8081 (http)
2026-09-27T14:58:49.545+05:30  INFO 28436 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2026-09-27T14:58:49.545+05:30  INFO 28436 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.16]
2026-09-27T14:58:49.701+05:30  INFO 28436 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2026-09-27T14:58:49.701+05:30  INFO 28436 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 2458 ms
Standard Commons Logging discovery in action with spring-jcl: please remove commons-logging.jar from classpath in order to avoid potential conflicts
================= FIRESTORE CONFIG =================
projectId     = client-ledger-dashboard
host          = 
emulatorHost  = 127.0.0.1:8080
=====================================================
2026-09-27T14:58:50.967+05:30  WARN 28436 --- [           main] .s.s.UserDetailsServiceAutoConfiguration : 

Using generated security password: 8389e9a7-4af0-4e63-91ee-c7c3f8e2e379

This generated password is for development use only. Your security configuration must be updated before running your application in production.

2026-09-27T14:58:51.451+05:30  INFO 28436 --- [           main] o.s.s.web.DefaultSecurityFilterChain     : Will secure any request with [org.springframework.security.web.session.DisableEncodeUrlFilter@566fe83c, org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@743bb39a, org.springframework.security.web.context.Security
ContextHolderFilter@b5d9f1e, org.springframework.security.web.header.HeaderWriterFilter@611c3eae, org.springframework.web.filter.CorsFilter@2de3b052, org.springframework.security.
web.authentication.logout.LogoutFilter@2e3f324e, org.springframework.security.web.savedrequest.RequestCacheAwareFilter@401317a0, org.springframework.security.web.servletapi.Securi
tyContextHolderAwareRequestFilter@3f866f50, org.springframework.security.web.authentication.AnonymousAuthenticationFilter@3f3c8b60, org.springframework.security.web.session.Sessio
nManagementFilter@423f8a73, org.springframework.security.web.access.ExceptionTranslationFilter@4bf4680c, org.springframework.security.web.access.intercept.AuthorizationFilter@44ccd75c]
2026-09-27T14:58:52.107+05:30  INFO 28436 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8081 (http) with context path ''
2026-09-27T14:58:52.123+05:30  INFO 28436 --- [           main] c.c.core.app.ClientLedgerApplicationKt   : Started ClientLedgerApplicationKt in 5.944 seconds (process running for 6.894)
2026-09-27T14:58:59.407+05:30  INFO 28436 --- [nio-8081-exec-2] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2026-09-27T14:58:59.407+05:30  INFO 28436 --- [nio-8081-exec-2] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2026-09-27T14:58:59.407+05:30  INFO 28436 --- [nio-8081-exec-2] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
2026-09-27T14:58:59.509+05:30  WARN 28436 --- [nio-8081-exec-2] o.s.w.s.h.HandlerMappingIntrospector     : Cache miss for REQUEST dispatch to '/api/owners' (previous null). Performing CorsConfiguration lookup. This is logged once only at WARN level, and every time at TRACE.
2026-09-27T14:59:09.814+05:30 ERROR 28436 --- [nio-8081-exec-2] c.c.c.exception.GlobalExceptionHandler   : Unexpected ledger error. clientId=N/A, message=com.google.api.gax.rpc.InternalException: io.grpc.StatusRuntimeException: INTERNAL: http2 exception

java.util.concurrent.ExecutionException: com.google.api.gax.rpc.InternalException: io.grpc.StatusRuntimeException: INTERNAL: http2 exception
        at com.google.common.util.concurrent.AbstractFuture.getDoneValue(AbstractFuture.java:588) ~[guava-31.1-jre.jar:na]
        at com.google.common.util.concurrent.AbstractFuture.get(AbstractFuture.java:567) ~[guava-31.1-jre.jar:na]
        at com.google.common.util.concurrent.FluentFuture$TrustedFuture.get(FluentFuture.java:91) ~[guava-31.1-jre.jar:na]
        at com.google.common.util.concurrent.ForwardingFuture.get(ForwardingFuture.java:66) ~[guava-31.1-jre.jar:na]
        at com.clientledger.core.repository.owner.OwnerRepository.exists(OwnerRepository.kt:36) ~[main/:na]
        at com.clientledger.core.service.owner.OwnerService.createOwner(OwnerService.kt:39) ~[main/:na]
        at com.clientledger.core.controller.owner.OwnerController.createOwner(OwnerController.kt:26) ~[main/:na]
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
        at jakarta.servlet.http.HttpServlet.service(HttpServlet.java:658) ~[tomcat-embed-core-10.1.16.jar:6.0]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:205) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:149) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:51) ~[tomcat-embed-websocket-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:174) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:149) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at com.clientledger.core.auth.FirebaseAuthenticationFilter.doFilterInternal(FirebaseAuthenticationFilter.kt:32) ~[main/:na]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.1.jar:6.1.1]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:174) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:149) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.springframework.security.web.FilterChainProxy.lambda$doFilterInternal$3(FilterChainProxy.java:231) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:365) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.access.intercept.AuthorizationFilter.doFilter(AuthorizationFilter.java:100) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.access.ExceptionTranslationFilter.doFilter(ExceptionTranslationFilter.java:126) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.access.ExceptionTranslationFilter.doFilter(ExceptionTranslationFilter.java:120) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.session.SessionManagementFilter.doFilter(SessionManagementFilter.java:131) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.session.SessionManagementFilter.doFilter(SessionManagementFilter.java:85) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.authentication.AnonymousAuthenticationFilter.doFilter(AnonymousAuthenticationFilter.java:100) ~[spring-security-web-6.2.0.jar:6.2.0]   
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter.doFilter(SecurityContextHolderAwareRequestFilter.java:179) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.savedrequest.RequestCacheAwareFilter.doFilter(RequestCacheAwareFilter.java:63) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:107) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.authentication.logout.LogoutFilter.doFilter(LogoutFilter.java:93) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.web.filter.CorsFilter.doFilterInternal(CorsFilter.java:91) ~[spring-web-6.1.1.jar:6.1.1]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.1.jar:6.1.1]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.header.HeaderWriterFilter.doHeadersAfter(HeaderWriterFilter.java:90) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.header.HeaderWriterFilter.doFilterInternal(HeaderWriterFilter.java:75) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.1.jar:6.1.1]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.context.SecurityContextHolderFilter.doFilter(SecurityContextHolderFilter.java:82) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.context.SecurityContextHolderFilter.doFilter(SecurityContextHolderFilter.java:69) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter.doFilterInternal(WebAsyncManagerIntegrationFilter.java:62) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.1.jar:6.1.1]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.session.DisableEncodeUrlFilter.doFilterInternal(DisableEncodeUrlFilter.java:42) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.1.jar:6.1.1]
        at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:374) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.FilterChainProxy.doFilterInternal(FilterChainProxy.java:233) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.security.web.FilterChainProxy.doFilter(FilterChainProxy.java:191) ~[spring-security-web-6.2.0.jar:6.2.0]
        at org.springframework.web.filter.DelegatingFilterProxy.invokeDelegate(DelegatingFilterProxy.java:352) ~[spring-web-6.1.1.jar:6.1.1]
        at org.springframework.web.filter.DelegatingFilterProxy.doFilter(DelegatingFilterProxy.java:268) ~[spring-web-6.1.1.jar:6.1.1]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:174) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:149) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100) ~[spring-web-6.1.1.jar:6.1.1]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.1.jar:6.1.1]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:174) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:149) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93) ~[spring-web-6.1.1.jar:6.1.1]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.1.jar:6.1.1]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:174) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:149) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201) ~[spring-web-6.1.1.jar:6.1.1]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:116) ~[spring-web-6.1.1.jar:6.1.1]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:174) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:149) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:167) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:90) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:482) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:115) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:93) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:340) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:391) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:63) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:896) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1744) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:52) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1191) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) ~[tomcat-embed-core-10.1.16.jar:10.1.16]
        at java.base/java.lang.Thread.run(Thread.java:842) ~[na:na]
Caused by: com.google.api.gax.rpc.InternalException: io.grpc.StatusRuntimeException: INTERNAL: http2 exception
        at com.google.api.gax.rpc.ApiExceptionFactory.createException(ApiExceptionFactory.java:110) ~[gax-2.29.0.jar:2.29.0]
        at com.google.api.gax.rpc.ApiExceptionFactory.createException(ApiExceptionFactory.java:41) ~[gax-2.29.0.jar:2.29.0]
        at com.google.api.gax.grpc.GrpcApiExceptionFactory.create(GrpcApiExceptionFactory.java:86) ~[gax-grpc-2.29.0.jar:2.29.0]
        at com.google.api.gax.grpc.GrpcApiExceptionFactory.create(GrpcApiExceptionFactory.java:66) ~[gax-grpc-2.29.0.jar:2.29.0]
        at com.google.api.gax.grpc.ExceptionResponseObserver.onErrorImpl(ExceptionResponseObserver.java:82) ~[gax-grpc-2.29.0.jar:2.29.0]
        at com.google.api.gax.rpc.StateCheckingResponseObserver.onError(StateCheckingResponseObserver.java:84) ~[gax-2.29.0.jar:2.29.0]
        at com.google.api.gax.grpc.GrpcDirectStreamController$ResponseObserverAdapter.onClose(GrpcDirectStreamController.java:148) ~[gax-grpc-2.29.0.jar:2.29.0]
        at io.grpc.internal.ClientCallImpl.closeObserver(ClientCallImpl.java:567) ~[grpc-core-1.55.1.jar:1.55.1]
        at io.grpc.internal.ClientCallImpl.access$300(ClientCallImpl.java:71) ~[grpc-core-1.55.1.jar:1.55.1]
        at io.grpc.internal.ClientCallImpl$ClientStreamListenerImpl$1StreamClosed.runInternal(ClientCallImpl.java:735) ~[grpc-core-1.55.1.jar:1.55.1]
        at io.grpc.internal.ClientCallImpl$ClientStreamListenerImpl$1StreamClosed.runInContext(ClientCallImpl.java:716) ~[grpc-core-1.55.1.jar:1.55.1]
        at io.grpc.internal.ContextRunnable.run(ContextRunnable.java:37) ~[grpc-core-1.55.1.jar:1.55.1]
        at io.grpc.internal.SerializingExecutor.run(SerializingExecutor.java:133) ~[grpc-core-1.55.1.jar:1.55.1]
        at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:539) ~[na:na]
        at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264) ~[na:na]
        at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304) ~[na:na]
        at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136) ~[na:na]
        at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635) ~[na:na]
        ... 1 common frames omitted
Caused by: io.grpc.StatusRuntimeException: INTERNAL: http2 exception
        at io.grpc.Status.asRuntimeException(Status.java:539) ~[grpc-api-1.55.1.jar:1.55.1]
        ... 13 common frames omitted
Caused by: io.grpc.netty.shaded.io.netty.handler.codec.http2.Http2Exception: First received frame was not SETTINGS. Hex dump for first 5 bytes: 1503010002
        at io.grpc.netty.shaded.io.netty.handler.codec.http2.Http2Exception.connectionError(Http2Exception.java:109) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.handler.codec.http2.Http2ConnectionHandler$PrefaceDecoder.verifyFirstFrameIsSettings(Http2ConnectionHandler.java:353) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.handler.codec.http2.Http2ConnectionHandler$PrefaceDecoder.decode(Http2ConnectionHandler.java:247) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]  
        at io.grpc.netty.shaded.io.netty.handler.codec.http2.Http2ConnectionHandler.decode(Http2ConnectionHandler.java:453) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.handler.codec.ByteToMessageDecoder.decodeRemovalReentryProtection(ByteToMessageDecoder.java:529) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]   
        at io.grpc.netty.shaded.io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:468) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.handler.codec.ByteToMessageDecoder.channelRead(ByteToMessageDecoder.java:290) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:444) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]    
        at io.grpc.netty.shaded.io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:420) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]    
        at io.grpc.netty.shaded.io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:412) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]      
        at io.grpc.netty.shaded.io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1410) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:440) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]    
        at io.grpc.netty.shaded.io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:420) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]    
        at io.grpc.netty.shaded.io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:919) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:166) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:788) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:724) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:650) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:562) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.util.concurrent.SingleThreadEventExecutor$4.run(SingleThreadEventExecutor.java:997) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.util.internal.ThreadExecutorMap$2.run(ThreadExecutorMap.java:74) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        at io.grpc.netty.shaded.io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30) ~[grpc-netty-shaded-1.55.1.jar:1.55.1]
        ... 1 common frames omitted

<============-> 94% EXECUTING
