package com.clientledger.core.service.maintenance

import org.slf4j.LoggerFactory
import org.springframework.beans.factory.annotation.Value
import org.springframework.scheduling.annotation.Scheduled
import org.springframework.stereotype.Component

@Component
class MonthlyMaintenanceScheduler(
    private val maintenanceService: MaintenanceService,
    @Value("\${server.port}") private val serverPort: String
) {
    private val logger = LoggerFactory.getLogger(MonthlyMaintenanceScheduler::class.java)

    @Scheduled(cron = "0 50 23 L * *", zone = "Asia/Kolkata")
    fun blockWrites() {
        logger.info( "[MONTHLY-MAINTENANCE] Attempting write freeze | instancePort={}", serverPort )
        val acquired = maintenanceService.tryBlockWrites( reason = "Monthly rollover write freeze" )
        if (acquired) {
            logger.info( "[MONTHLY-MAINTENANCE] WRITE FREEZE ACQUIRED | instancePort={}", serverPort )
        }
        else {
            logger.info( "[MONTHLY-MAINTENANCE] Write freeze already established by another instance | instancePort={}", serverPort )
        }
    }

    @Scheduled(cron = "0 0 0 1 * *", zone = "Asia/Kolkata")
    fun rollover() {
        logger.info( "[MONTHLY-ROLLOVER] Scheduler triggered | instancePort={}", serverPort )
        maintenanceService.performMonthlyRollover()
        logger.info( "[MONTHLY-ROLLOVER] Scheduler execution finished | instancePort={}", serverPort )
    }
}

