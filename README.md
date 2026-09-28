package com.clientledger.core.config

import org.springframework.boot.context.properties.ConfigurationProperties

@ConfigurationProperties(prefix = "client-ledger")
data class ClientLedgerProperties(

    val mode: ApplicationMode = ApplicationMode.EMULATOR,

    val history: HistoryProperties = HistoryProperties(),

    val summary: SummaryProperties = SummaryProperties(),

    val firestore: FirestoreProperties = FirestoreProperties(),

    val auth: AuthProperties = AuthProperties()
) {

    enum class ApplicationMode {
        EMULATOR,
        CLOUD
    }

    data class HistoryProperties(
        val editableMonths: Int = 8,
        val bucketCapacity: Int = 100
    )

    data class FirestoreProperties(
        val projectId: String = "",
        val host: String = "",
        val emulatorHost: String = "127.0.0.1:8080"
    )

    data class SummaryProperties(
        val indexBucketCapacity: Int = 300
    )

    data class AuthProperties(
        val enabled: Boolean = true,
        val mode: AuthMode = AuthMode.EMULATOR,
        val emulatorHost: String = "127.0.0.1:9099",
        val credentialsPath: String = "",
        val localOwnerId: String = ""
    )

    enum class AuthMode {
        EMULATOR,
        CLOUD
    }
}package com.clientledger.core.config

import com.google.auth.oauth2.GoogleCredentials
import com.google.firebase.FirebaseApp
import com.google.firebase.FirebaseOptions
import com.google.firebase.auth.FirebaseAuth
import jakarta.annotation.PostConstruct
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.io.ClassPathResource
import java.io.FileInputStream

@Configuration
class FirebaseAdminConfig(
    private val properties: ClientLedgerProperties
) {

    @PostConstruct
    fun initEmulatorEnvironment() {
        val authProperties = properties.auth
        if (authProperties.enabled && authProperties.mode == ClientLedgerProperties.AuthMode.EMULATOR) {
            val emulatorHost = authProperties.emulatorHost.ifBlank { "127.0.0.1:9099" }
            System.setProperty("FIREBASE_AUTH_EMULATOR_HOST", emulatorHost)
        } else {
            // Clear environment variable when running in CLOUD mode
            System.clearProperty("FIREBASE_AUTH_EMULATOR_HOST")
        }
    }

    @Bean
    fun firebaseApp(): FirebaseApp {
        FirebaseApp.getApps()
            .firstOrNull()
            ?.let { return it }

        val authProperties = properties.auth

        val optionsBuilder = FirebaseOptions.builder()
            .setProjectId(properties.firestore.projectId)

        when (authProperties.mode) {
            ClientLedgerProperties.AuthMode.EMULATOR -> {
                optionsBuilder.setCredentials(GoogleCredentials.create(null))
            }

            ClientLedgerProperties.AuthMode.CLOUD -> {
                val credentialsPath = authProperties.credentialsPath

                require(credentialsPath.isNotBlank()) {
                    "client-ledger.auth.credentials-path must not be blank when auth mode is CLOUD"
                }

                // Check if path starts with classpath: or points to a resource in src/main/resources
                val credentialsStream = if (credentialsPath.startsWith("classpath:")) {
                    ClassPathResource(credentialsPath.removePrefix("classpath:")).inputStream
                } else {
                    // Falls back to direct file stream if an absolute file path is passed
                    runCatching { ClassPathResource(credentialsPath).inputStream }
                        .getOrElse { FileInputStream(credentialsPath) }
                }

                credentialsStream.use { inputStream ->
                    optionsBuilder.setCredentials(GoogleCredentials.fromStream(inputStream))
                }
            }
        }

        return FirebaseApp.initializeApp(
            optionsBuilder.build(),
            FirebaseApp.DEFAULT_APP_NAME
        ) ?: error("Failed to initialize Firebase Admin SDK")
    }

    @Bean
    fun firebaseAuth(firebaseApp: FirebaseApp): FirebaseAuth {
        return FirebaseAuth.getInstance(firebaseApp)
    }
}package com.clientledger.core.config

import com.google.api.gax.grpc.InstantiatingGrpcChannelProvider
import com.google.auth.oauth2.GoogleCredentials
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.io.FileInputStream

@Configuration
class FireStoreConfig(
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun firestore(): Firestore {

        return when (properties.mode) {

            ClientLedgerProperties.ApplicationMode.EMULATOR -> {
                createEmulatorFirestore()
            }

            ClientLedgerProperties.ApplicationMode.CLOUD -> {
                createCloudFirestore()
            }
        }
    }

    private fun createEmulatorFirestore(): Firestore {

        val firestoreProperties = properties.firestore
        val targetHost = firestoreProperties.emulatorHost

        require(targetHost.isNotBlank()) {
            "client-ledger.firestore.emulator-host must not be blank when application mode is EMULATOR"
        }

        println(
            """
            ================= FIRESTORE CONFIG =================
            mode          = EMULATOR
            projectId     = ${firestoreProperties.projectId}
            emulatorHost  = $targetHost
            =====================================================
            """.trimIndent()
        )

        val channelProvider =
            InstantiatingGrpcChannelProvider.newBuilder()
                .setEndpoint(targetHost)
                .setChannelConfigurator { builder ->
                    builder.usePlaintext()
                }
                .build()

        return FirestoreOptions.newBuilder()
            .setProjectId(firestoreProperties.projectId)
            .setHost(targetHost)
            .setChannelProvider(channelProvider)
            .setCredentials(NoCredentials.getInstance())
            .build()
            .service
    }

    private fun createCloudFirestore(): Firestore {

        val firestoreProperties = properties.firestore
        val credentialsPath = properties.auth.credentialsPath

        require(firestoreProperties.projectId.isNotBlank()) {
            "client-ledger.firestore.project-id must not be blank"
        }

        require(credentialsPath.isNotBlank()) {
            "client-ledger.auth.credentials-path must not be blank when application mode is CLOUD"
        }

        println(
            """
            ================= FIRESTORE CONFIG =================
            mode             = CLOUD
            projectId        = ${firestoreProperties.projectId}
            credentialsPath  = $credentialsPath
            =====================================================
            """.trimIndent()
        )

        val credentials =
            if (credentialsPath.startsWith("classpath:")) {
                val resourcePath =
                    credentialsPath.removePrefix("classpath:")

                val resource =
                    javaClass.classLoader.getResource(resourcePath)
                        ?: error(
                            "Firebase service account file not found on classpath: $resourcePath"
                        )

                resource.openStream().use { inputStream ->
                    GoogleCredentials.fromStream(inputStream)
                }
            } else {
                FileInputStream(credentialsPath).use { inputStream ->
                    GoogleCredentials.fromStream(inputStream)
                }
            }
package com.clientledger.core.config

import com.clientledger.core.auth.FirebaseAuthenticationFilter
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter

@Configuration
@EnableWebSecurity
class SecurityConfig(
    private val firebaseAuthenticationFilter: FirebaseAuthenticationFilter,
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun securityFilterChain(
        http: HttpSecurity
    ): SecurityFilterChain {

        http
            .csrf { it.disable() }
            .sessionManagement {
                it.sessionCreationPolicy(
                    SessionCreationPolicy.STATELESS
                )
            }
            .authorizeHttpRequests { auth ->

                if (!properties.auth.enabled) {

                    auth.anyRequest().permitAll()

                } else {

                    auth
                        .requestMatchers(
                            "/actuator/health",
                            "/api/auth/**"
                        )
                        .permitAll()

                        // GENERAL users can access the normal APIs.
                        .requestMatchers(
                            "/api/clients/**",
                            "/api/orders/**",
                            "/api/payments/**",
                            "/api/invoices/**",
                            "/api/summary/**"
                        )
                        .hasAnyRole(
                            "GENERAL",
                            "PRO"
                        )

                        // PRO-only APIs will be added here.
                        // Example:
                        .requestMatchers("/api/pro/**")
                        .hasRole("PRO")

                        .anyRequest()
                        .authenticated()
                }
            }

        if (properties.auth.enabled) {
            http.addFilterBefore(
                firebaseAuthenticationFilter,
                UsernamePasswordAuthenticationFilter::class.java
            )
        }

        return http.build()
    }
}server:
  port: 8081

client-ledger:
  history:
    editable-months: 2
    bucket-capacity: 100

  summary:
    index-bucket-capacity: 300

  firestore:
    project-id: "client-ledger-dashboard"
    emulator-host: "127.0.0.1:8080"

  auth:
    # =====================================================
    # LOCAL DEVELOPMENT - AUTH DISABLED
    # =====================================================
    # enabled: false
    # local-owner-id: "local-owner"

    # =====================================================
    # LOCAL FIREBASE AUTH EMULATOR
    # Uncomment these and comment the above enabled/local-owner
    # =====================================================
#    enabled: true
#    mode: EMULATOR
#    emulator-host: "127.0.0.1:9099"
#    local-owner-id: "local-owner"

    # =====================================================
    # FIREBASE CLOUD
    # Uncomment these for cloud testing
    # always enabled true
    # =====================================================
     enabled: true
     mode: CLOUD
     credentials-path: "classpath:client-ledger-service-account.json"
        return FirestoreOptions.newBuilder()
            .setProjectId(firestoreProperties.projectId)
            .setCredentials(credentials)
            .build()
            .service
    }
}
