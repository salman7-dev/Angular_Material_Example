PS D:\New folder\client-ledger-codespace-main> $env:FIREBASE_AUTH_EMULATOR_HOST="127.0.0.1:9099"
PS D:\New folder\client-ledger-codespace-main> $env:FIREBASE_AUTH_EMULATOR_HOST
127.0.0.1:9099
PS D:\New folder\client-ledger-codespace-main> .\gradlew.bat --stop
Stopping Daemon(s)
2 Daemons stopped
PS D:\New folder\client-ledger-codespace-main> .\gradlew.bat :app:bootRun --no-daemon
To honour the JVM settings for this build a single-use Daemon process will be forked. For more on this, please refer to https://docs.gradle.org/8.5/userguide/gradle_daemon.html#sec:disabling_the_daemon in the Gradle documentation.
Daemon will be stopped at the end of the build 
Reusing configuration cache.

> Task :app:bootRun                                                                                                                                                                
Standard Commons Logging discovery in action with spring-jcl: please remove commons-logging.jar from classpath in order to avoid potential conflicts

  .   ____          _            __ _ _                                                                                                                                            
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.0)

2026-09-27T14:43:24.943+05:30  INFO 16544 --- [           main] c.c.core.app.ClientLedgerApplicationKt   : Starting ClientLedgerApplicationKt using Java 17.0.11 with PID 16544 (D:\New folder\client-ledger-codespace-main\app\build\classes\kotlin\main started by khans in D:\New folder\client-ledger-codespace-main\app)
2026-09-27T14:43:24.947+05:30  INFO 16544 --- [           main] c.c.core.app.ClientLedgerApplicationKt   : No active profile set, falling back to 1 default profile: "default"
2026-09-27T14:43:27.711+05:30  INFO 16544 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port 8081 (http)
2026-09-27T14:43:27.746+05:30  INFO 16544 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]                                               
2026-09-27T14:43:27.747+05:30  INFO 16544 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.16]
2026-09-27T14:43:27.935+05:30  INFO 16544 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2026-09-27T14:43:27.938+05:30  INFO 16544 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 2855 ms         
Standard Commons Logging discovery in action with spring-jcl: please remove commons-logging.jar from classpath in order to avoid potential conflicts
2026-09-27T14:43:29.041+05:30  WARN 16544 --- [           main] .s.s.UserDetailsServiceAutoConfiguration : 

Using generated security password: db650c4b-1c07-43d2-8ab5-a51abc8f9466

This generated password is for development use only. Your security configuration must be updated before running your application in production.

2026-09-27T14:43:29.302+05:30  INFO 16544 --- [           main] o.s.s.web.DefaultSecurityFilterChain     : Will secure any request with [org.springframework.security.web.session.DisableEncodeUrlFilter@76437e9b, org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@236ae13d, org.springframework.security.web.context.SecurityContextHolderFilter@6f1c3f18, org.springframework.security.web.header.HeaderWriterFilter@1416ff46, org.springframework.web.filter.CorsFilter@193eb1ba, org.springframework.security.web.csrf.CsrfFilter@49c099b, org.springframework.security.web.authentication.logout.LogoutFilter@1b4872bc, org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter@61e86192, org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter@48581a3b, org.springframework.security.web.authentication.ui.DefaultLog
outPageGeneratingFilter@2be818da, org.springframework.security.web.authentication.www.BasicAuthenticationFilter@52d0f583, org.springframework.security.web.savedrequest.RequestCach
eAwareFilter@489bc8fd, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@5ac53c06, org.springframework.security.web.authentication.AnonymousAuthe
nticationFilter@46320c9a, org.springframework.security.web.access.ExceptionTranslationFilter@2eada095, org.springframework.security.web.access.intercept.AuthorizationFilter@59303963]
2026-09-27T14:43:29.410+05:30  INFO 16544 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8081 (http) with context path ''
2026-09-27T14:43:29.427+05:30  INFO 16544 --- [           main] c.c.core.app.ClientLedgerApplicationKt   : Started ClientLedgerApplicationKt in 5.505 seconds (process running for 6.425)
2026-09-27T14:44:09.100+05:30  INFO 16544 --- [nio-8081-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2026-09-27T14:44:09.101+05:30  INFO 16544 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'                                
2026-09-27T14:44:09.104+05:30  INFO 16544 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 3 ms
2026-09-27T14:44:09.486+05:30  WARN 16544 --- [nio-8081-exec-1] o.s.w.s.h.HandlerMappingIntrospector     : Cache miss for REQUEST dispatch to '/api/owners' (previous null). Performing CorsConfiguration lookup. This is logged once only at WARN level, and every time at TRACE.                                                                                    
2026-09-27T14:44:09.655+05:30  WARN 16544 --- [nio-8081-exec-1] o.a.c.util.SessionIdGeneratorBase        : Creation of SecureRandom instance for session ID generation using [SHA1PRNG] took [126] milliseconds.
<============-> 94% EXECUTING [2m 18s]
> IDLE
> IDLE                                                                                                                                                                             
> IDLE
> :app:bootRun

