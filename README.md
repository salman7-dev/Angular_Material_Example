package com.clientledger.core.app

import com.clientledger.core.config.ClientLedgerProperties
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.context.properties.ConfigurationPropertiesScan
import org.springframework.boot.context.properties.EnableConfigurationProperties
import org.springframework.boot.runApplication

@SpringBootApplication
@EnableConfigurationProperties(ClientLedgerProperties::class)
@ConfigurationPropertiesScan(basePackages = ["com.clientledger.core"])
class ClientLedgerApplication

fun main(args: Array<String>) {
    runApplication<ClientLedgerApplication>(*args)
}
