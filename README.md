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
}
