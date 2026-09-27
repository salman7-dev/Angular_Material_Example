package com.clientledger.core.config

import com.google.auth.oauth2.GoogleCredentials
import com.google.firebase.FirebaseApp
import com.google.firebase.FirebaseOptions
import com.google.firebase.auth.FirebaseAuth
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import java.io.FileInputStream

@Configuration
class FirebaseAdminConfig(
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun firebaseApp(): FirebaseApp {

        FirebaseApp.getApps()
            .firstOrNull()
            ?.let { return it }

        val authProperties = properties.auth

        val optionsBuilder =
            FirebaseOptions.builder()
                .setProjectId(properties.firestore.projectId)

        when (authProperties.mode) {

            ClientLedgerProperties.AuthMode.EMULATOR -> {
                optionsBuilder.setCredentials(
                    GoogleCredentials.create(null)
                )
            }

            ClientLedgerProperties.AuthMode.CLOUD -> {
                val credentialsPath =
                    authProperties.credentialsPath

                require(credentialsPath.isNotBlank()) {
                    "client-ledger.auth.credentials-path must not be blank when auth mode is CLOUD"
                }

                optionsBuilder.setCredentials(
                    FileInputStream(credentialsPath).use { inputStream ->
                        GoogleCredentials.fromStream(inputStream)
                    }
                )
            }
        }

        return FirebaseApp.initializeApp(
            optionsBuilder.build(),
            FirebaseApp.DEFAULT_APP_NAME
        ) ?: error("Failed to initialize Firebase Admin SDK")
    }

    @Bean
    fun firebaseAuth(
        firebaseApp: FirebaseApp
    ): FirebaseAuth {
        return FirebaseAuth.getInstance(firebaseApp)
    }
}
