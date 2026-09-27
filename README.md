package com.clientledger.core.config

import com.google.cloud.NoCredentials
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.FirestoreOptions
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class FireStoreConfig(
    private val properties: ClientLedgerProperties
) {

    @Bean
    fun firestore(): Firestore {

        println(
            """
        ================= FIRESTORE CONFIG =================
        projectId     = ${properties.firestore.projectId}
        host          = ${properties.firestore.host}
        emulatorHost  = ${properties.firestore.emulatorHost}
        =====================================================
        """.trimIndent()
        )

        val builder = FirestoreOptions.newBuilder()
            .setProjectId(properties.firestore.projectId)
            .setCredentials(NoCredentials.getInstance())

        if (properties.firestore.host.isNotBlank()) {
            builder.setHost(properties.firestore.host)
        }

        if (properties.firestore.emulatorHost.isNotBlank()) {
            builder.setEmulatorHost(properties.firestore.emulatorHost)
        }

        return builder.build().service
    }
}

