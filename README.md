package com.clientledger.core.config

import com.google.auth.oauth2.GoogleCredentials
import com.google.firebase.FirebaseApp
import com.google.firebase.FirebaseOptions
import com.google.firebase.auth.FirebaseAuth
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.io.ClassPathResource
import java.io.FileInputStream

@Configuration
@ConditionalOnProperty(
    prefix = "client-ledger",
    name = ["mode"],
    havingValue = "CLOUD"
)
class FirebaseCloudConfig(
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun firebaseApp(): FirebaseApp {

        FirebaseApp.getApps()
            .firstOrNull()
            ?.let { return it }

        val credentialsPath =
            properties.auth.credentialsPath

        require(credentialsPath.isNotBlank()) {
            "client-ledger.auth.credentials-path must not be blank when mode is CLOUD"
        }

        val credentials =
            if (credentialsPath.startsWith("classpath:")) {

                val resourcePath =
                    credentialsPath.removePrefix("classpath:")

                val resource =
                    ClassPathResource(resourcePath)

                require(resource.exists()) {
                    "Firebase service account file not found on classpath: $resourcePath"
                }

                resource.inputStream.use { inputStream ->
                    GoogleCredentials.fromStream(inputStream)
                }

            } else {

                FileInputStream(credentialsPath).use { inputStream ->
                    GoogleCredentials.fromStream(inputStream)
                }
            }

        println(
            """
            =================================================
             FIREBASE CONFIGURATION
            =================================================
             Environment : CLOUD
             Project ID  : ${properties.firestore.projectId}
             Auth        : FIREBASE CLOUD
             Firestore   : FIREBASE CLOUD
            =================================================
            """.trimIndent()
        )

        val options =
            FirebaseOptions.builder()
                .setProjectId(properties.firestore.projectId)
                .setCredentials(credentials)
                .build()

        return FirebaseApp.initializeApp(
            options,
            FirebaseApp.DEFAULT_APP_NAME
        ) ?: error("Failed to initialize Firebase Cloud")
    }

    @Bean
    fun firebaseAuth(
        firebaseApp: FirebaseApp
    ): FirebaseAuth {
        return FirebaseAuth.getInstance(firebaseApp)
    }
}

package com.clientledger.core.config

import com.google.auth.oauth2.GoogleCredentials
import com.google.firebase.FirebaseApp
import com.google.firebase.FirebaseOptions
import com.google.firebase.auth.FirebaseAuth
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
@ConditionalOnProperty(
    prefix = "client-ledger",
    name = ["mode"],
    havingValue = "EMULATOR"
)
class FirebaseEmulatorConfig(
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun firebaseApp(): FirebaseApp {

        FirebaseApp.getApps()
            .firstOrNull()
            ?.let { return it }

        val emulatorHost =
            properties.auth.emulatorHost.ifBlank {
                "127.0.0.1:9099"
            }

        println(
            """
            =================================================
             FIREBASE CONFIGURATION
            =================================================
             Environment : EMULATOR
             Project ID  : ${properties.firestore.projectId}
             Auth        : FIREBASE AUTH EMULATOR ($emulatorHost)
             Firestore   : FIRESTORE EMULATOR (${properties.firestore.emulatorHost})
            =================================================
            """.trimIndent()
        )

        val options =
            FirebaseOptions.builder()
                .setProjectId(properties.firestore.projectId)
                .setCredentials(GoogleCredentials.create(null))
                .build()

        return FirebaseApp.initializeApp(
            options,
            FirebaseApp.DEFAULT_APP_NAME
        ) ?: error("Failed to initialize Firebase Emulator")
    }

    @Bean
    fun firebaseAuth(
        firebaseApp: FirebaseApp
    ): FirebaseAuth {
        return FirebaseAuth.getInstance(firebaseApp)
    }
}

package com.clientledger.core.config

import com.google.auth.oauth2.GoogleCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.core.io.ClassPathResource
import java.io.FileInputStream

@Configuration
@ConditionalOnProperty(
    prefix = "client-ledger",
    name = ["mode"],
    havingValue = "CLOUD"
)
class FireStoreCloudConfig(
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun firestore(): Firestore {

        val projectId =
            properties.firestore.projectId

        val credentialsPath =
            properties.auth.credentialsPath

        require(projectId.isNotBlank()) {
            "client-ledger.firestore.project-id must not be blank when mode is CLOUD"
        }

        require(credentialsPath.isNotBlank()) {
            "client-ledger.auth.credentials-path must not be blank when mode is CLOUD"
        }

        val credentials =
            if (credentialsPath.startsWith("classpath:")) {

                val resourcePath =
                    credentialsPath.removePrefix("classpath:")

                val resource =
                    ClassPathResource(resourcePath)

                require(resource.exists()) {
                    "Firebase service account file not found on classpath: $resourcePath"
                }

                resource.inputStream.use { inputStream ->
                    GoogleCredentials.fromStream(inputStream)
                }

            } else {

                FileInputStream(credentialsPath).use { inputStream ->
                    GoogleCredentials.fromStream(inputStream)
                }
            }

        println(
            """
            =================================================
             FIRESTORE CONFIGURATION
            =================================================
             Environment : CLOUD
             Project ID  : $projectId
             Firestore   : FIREBASE CLOUD
            =================================================
            """.trimIndent()
        )

        return FirestoreOptions.newBuilder()
            .setProjectId(projectId)
            .setCredentials(credentials)
            .build()
            .service
    }
}


package com.clientledger.core.config

import com.google.api.gax.grpc.InstantiatingGrpcChannelProvider
import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
@ConditionalOnProperty(
    prefix = "client-ledger",
    name = ["mode"],
    havingValue = "EMULATOR"
)
class FireStoreEmulatorConfig(
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun firestore(): Firestore {

        val projectId =
            properties.firestore.projectId

        val emulatorHost =
            properties.firestore.emulatorHost

        require(projectId.isNotBlank()) {
            "client-ledger.firestore.project-id must not be blank when mode is EMULATOR"
        }

        require(emulatorHost.isNotBlank()) {
            "client-ledger.firestore.emulator-host must not be blank when mode is EMULATOR"
        }

        println(
            """
            =================================================
             FIRESTORE CONFIGURATION
            =================================================
             Environment : EMULATOR
             Project ID  : $projectId
             Firestore   : FIRESTORE EMULATOR ($emulatorHost)
            =================================================
            """.trimIndent()
        )

        val channelProvider =
            InstantiatingGrpcChannelProvider.newBuilder()
                .setEndpoint(emulatorHost)
                .setChannelConfigurator { builder ->
                    builder.usePlaintext()
                }
                .build()

        return FirestoreOptions.newBuilder()
            .setProjectId(projectId)
            .setHost(emulatorHost)
            .setChannelProvider(channelProvider)
            .setCredentials(NoCredentials.getInstance())
            .build()
            .service
    }
}


server:
  port: 8081

client-ledger:
  mode: CLOUD  # change to EMULATOR if you want to use and below line uncomment in auth
  history:
    editable-months: 2
    bucket-capacity: 100

  summary:
    index-bucket-capacity: 300

    # =====================================================
    # FIREBASE EMULATOR
    # Uncomment these for cloud testing
    # always enabled true
    # =====================================================

#  firestore:
#    project-id: "client-ledger-dashboard"
#    emulator-host: "127.0.0.1:8080"
#  auth:
#    enabled: true
#    emulator-host: "127.0.0.1:9099"
#    local-owner-id: "local-owner"


    # =====================================================
    # FIREBASE CLOUD
    # Uncomment these for cloud testing
    # always enabled true
    # =====================================================
  firestore:
    project-id: "client-ledger-dashboard"
  auth:
     enabled: true
     credentials-path: "classpath:client-ledger-service-account.json"
