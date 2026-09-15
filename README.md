package com.clientledger.core.transaction

import com.clientledger.core.domain.client.Client
import com.clientledger.core.domain.invoice.Invoice
import com.clientledger.core.domain.order.Order
import com.clientledger.core.domain.payment.Payment
import com.clientledger.core.domain.snapshot.ClientBalanceSnapshot
import com.clientledger.core.domain.summary.GlobalSummary
import com.clientledger.core.engine.LedgerCalculationEngine
import com.clientledger.core.engine.LedgerCalculationResult
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.invoice.InvoiceRepository
import com.clientledger.core.repository.order.OrderRepository
import com.clientledger.core.repository.payment.PaymentRepository
import com.clientledger.core.repository.snapshot.SnapshotRepository
import com.clientledger.core.service.reporting.ClientMonthlyReportingService
import com.clientledger.core.service.summary.GlobalSummaryService
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Component
import java.time.Instant
import java.time.LocalDate
import java.time.YearMonth
import java.time.temporal.ChronoUnit
import kotlin.math.abs

@Component
class LedgerTransactionCoordinator(
    private val transactionExecutor: FirestoreTransactionExecutor,
    private val clientRepository: ClientRepository,
    private val orderRepository: OrderRepository,
    private val paymentRepository: PaymentRepository,
    private val invoiceRepository: InvoiceRepository,
    private val snapshotRepository: SnapshotRepository,
    private val ledgerCalculationEngine: LedgerCalculationEngine,
    private val db: Firestore,
    private val globalSummaryService: GlobalSummaryService,
    private val clientMonthlyReportingService: ClientMonthlyReportingService
) {

    fun saveOrderAndRecalculateLedger(
        order: Order,
        transactionDate: LocalDate
    ): LedgerCalculationResult {

        val transactionResult =
            transactionExecutor.execute(order.clientId) { transaction ->

                println("============================================================")
                println("DEBUG: saveOrderAndRecalculateLedger - START")
                println("DEBUG: orderId=${order.id}")
                println("DEBUG: transactionDate=$transactionDate")
                println("============================================================")

                // =============================================================
                // 1. READ EXISTING ORDER
                // =============================================================

                val existingOrder =
                    orderRepository.findById(
                        transaction,
                        order.id
                    )

                println("DEBUG: Existing Order = $existingOrder")

                // =============================================================
                // 2. DETERMINE OLD / NEW CLIENT AND MONTH
                // =============================================================

                val oldClientId =
                    existingOrder?.clientId

                val newClientId =
                    order.clientId

                val oldMonth =
                    existingOrder?.let {
                        YearMonth.from(it.orderDate)
                    }

                val newMonth =
                    YearMonth.from(order.orderDate)

                println(
                    "DEBUG: oldClientId=$oldClientId, " +
                            "newClientId=$newClientId, " +
                            "oldMonth=$oldMonth, " +
                            "newMonth=$newMonth"
                )

                // =============================================================
                // 3. DETERMINE AFFECTED CLIENTS
                // =============================================================

                val affectedClientIds =
                    linkedSetOf<String>().apply {

                        oldClientId?.let {
                            add(it)
                        }

                        add(newClientId)
                    }

                println(
                    "DEBUG: affectedClientIds=$affectedClientIds"
                )

                // =============================================================
                // 4. READ ALL CLIENTS
                // =============================================================

                val clientsById =
                    affectedClientIds.associateWith { clientId ->

                        println(
                            "DEBUG: Reading Client $clientId"
                        )

                        clientRepository.findById(
                            transaction,
                            clientId
                        )
                    }

                // =============================================================
                // 5. SNAPSHOTS ARE NOW READ TARGETED
                // =============================================================

                // =============================================================
                // 6. CALCULATE ALL CLIENT SNAPSHOTS IN MEMORY
                // =============================================================

                val allCalculatedSnapshots =
                    mutableListOf<ClientBalanceSnapshot>()

                val finalClosingBalanceByClient =
                    mutableMapOf<String, Long>()

                val oldSnapshotForGlobalByClientMonth =
                    mutableMapOf<
                            Pair<String, YearMonth>,
                            ClientBalanceSnapshot
                            >()

                val snapshotsByClient =
                    mutableMapOf<
                            String,
                            List<ClientBalanceSnapshot>
                            >()

                affectedClientIds.forEach { clientId ->

                    println("------------------------------------------------------------")
                    println("DEBUG: Processing Client $clientId")
                    println("------------------------------------------------------------")

                    val client =
                        clientsById[clientId]

                    val isOldClient =
                        oldClientId == clientId

                    val isNewClient =
                        newClientId == clientId

                    // ---------------------------------------------------------
                    // Determine affected months
                    // ---------------------------------------------------------

                    val affectedMonths =
                        linkedSetOf<YearMonth>().apply {

                            if (
                                isOldClient &&
                                oldMonth != null
                            ) {
                                add(oldMonth)
                            }

                            if (isNewClient) {
                                add(newMonth)
                            }
                        }

                    require(
                        affectedMonths.isNotEmpty()
                    ) {
                        "No affected month found for Client $clientId."
                    }

                    val startMonth =
                        affectedMonths.minOrNull()
                            ?: throw IllegalStateException(
                                "Unable to determine start month for Client $clientId."
                            )

                    println(
                        "DEBUG: Client $clientId startMonth=$startMonth"
                    )

                    // ---------------------------------------------------------
                    // Determine latest stored transaction month
                    // ---------------------------------------------------------

                    val latestStoredTransactionMonth =
                        getEndMonth(
                            transaction = transaction,
                            clientId = clientId
                        )

                    println(
                        "DEBUG: Client $clientId " +
                                "latestStoredTransactionMonth=$latestStoredTransactionMonth"
                    )

                    // ---------------------------------------------------------
                    // Determine latest stored snapshot month
                    // ---------------------------------------------------------

                    val latestSnapshot =
                        snapshotRepository.findLatestSnapshotForClient(
                            transaction = transaction,
                            clientId = clientId
                        )

                    val latestSnapshotMonth =
                        latestSnapshot?.yearMonth

                    println(
                        "DEBUG: Client $clientId " +
                                "latestSnapshotMonth=$latestSnapshotMonth"
                    )

                    val endMonth =
                        listOfNotNull(
                            startMonth,
                            latestSnapshotMonth,
                            latestStoredTransactionMonth,
                            if (isNewClient) newMonth else null
                        ).maxOrNull()
                            ?: startMonth

                    println(
                        "DEBUG: Client $clientId endMonth=$endMonth"
                    )

                    // ---------------------------------------------------------
                    // Read snapshots only inside required range
                    // ---------------------------------------------------------

                    val snapshots =
                        snapshotRepository.getSnapshotsForClientInRange(
                            transaction = transaction,
                            clientId = clientId,
                            startMonth = startMonth,
                            endMonth = endMonth
                        )

                    println(
                        "DEBUG: Client $clientId has " +
                                "${snapshots.size} snapshots in range " +
                                "$startMonth -> $endMonth"
                    )

                    snapshots.forEach { snapshot ->
                        println(
                            "DEBUG: STORED SNAPSHOT " +
                                    "client=${snapshot.clientId}, " +
                                    "month=${snapshot.yearMonth}, " +
                                    "orders=${snapshot.totalOrdersCount}, " +
                                    "invoice=${snapshot.totalInvoiceAmount}, " +
                                    "invoiceWithGst=${snapshot.totalInvoiceAmountWithGst}, " +
                                    "payments=${snapshot.totalPayments}, " +
                                    "expenses=${snapshot.totalExpenses}, " +
                                    "discount=${snapshot.totalDiscount}, " +
                                    "opening=${snapshot.openingBalance}, " +
                                    "closing=${snapshot.closingBalance}"
                        )
                    }

                    snapshotsByClient[clientId] =
                        snapshots

                    val snapshotMap =
                        snapshots.associateBy {
                            it.yearMonth
                        }

                    // ---------------------------------------------------------
                    // Opening balance
                    // ---------------------------------------------------------

                    val openingBalance =
                        snapshotRepository.getOpeningBalanceSeed(
                            transaction = transaction,
                            clientId = clientId,
                            startMonth = startMonth,
                            fallbackOpeningBalance =
                            client?.lastClosingBalance ?: 0L
                        )

                    println(
                        "DEBUG: Client $clientId openingBalance=$openingBalance"
                    )

                    // ---------------------------------------------------------
                    // Process months
                    // ---------------------------------------------------------

                    var runningClosingBalance =
                        openingBalance

                    var monthCursor =
                        startMonth

                    while (!monthCursor.isAfter(endMonth)) {

                        println("------------------------------------------------------------")
                        println(
                            "DEBUG: PROCESSING MONTH " +
                                    "client=$clientId " +
                                    "month=$monthCursor"
                        )
                        println("------------------------------------------------------------")

                        val existingSnapshot =
                            snapshotMap[monthCursor]

                        val isAffectedMonth =
                            monthCursor in affectedMonths

                        println(
                            "DEBUG: existingSnapshot=$existingSnapshot"
                        )

                        println(
                            "DEBUG: isAffectedMonth=$isAffectedMonth"
                        )

                        println(
                            "DEBUG: runningClosingBalance BEFORE=$runningClosingBalance"
                        )

                        // -----------------------------------------------------
                        // Determine old/new Order for this month
                        // -----------------------------------------------------

                        val oldOrderForMonth =
                            if (
                                isOldClient &&
                                oldMonth == monthCursor
                            ) {
                                existingOrder
                            } else {
                                null
                            }

                        val newOrderForMonth =
                            if (
                                isNewClient &&
                                newMonth == monthCursor
                            ) {
                                order
                            } else {
                                null
                            }

                        println(
                            "DEBUG: oldOrderForMonth=" +
                                    "${oldOrderForMonth?.id}"
                        )

                        println(
                            "DEBUG: newOrderForMonth=" +
                                    "${newOrderForMonth?.id}"
                        )

                        // -----------------------------------------------------
                        // Calculate month
                        // -----------------------------------------------------

                        val monthSnapshot: ClientBalanceSnapshot

                        when {

                            // =================================================
                            // CASE 1
                            // Existing snapshot + affected month
                            // =================================================

                            isAffectedMonth &&
                                    existingSnapshot != null -> {

                                println(
                                    "DEBUG: CASE 1 - " +
                                            "Existing snapshot + affected month"
                                )

                                println(
                                    "DEBUG: Applying Order delta " +
                                            "for Client=$clientId " +
                                            "Month=$monthCursor " +
                                            "oldOrder=${oldOrderForMonth?.id} " +
                                            "newOrder=${newOrderForMonth?.id}"
                                )

                                oldSnapshotForGlobalByClientMonth[
                                    clientId to monthCursor
                                ] =
                                    existingSnapshot

                                monthSnapshot =
                                    ledgerCalculationEngine.applyOrderDelta(
                                        existingSnapshot =
                                        existingSnapshot,

                                        openingBalance =
                                        runningClosingBalance,

                                        oldOrder =
                                        oldOrderForMonth,

                                        newOrder =
                                        newOrderForMonth
                                    )
                            }

                            // =================================================
                            // CASE 2
                            // Existing snapshot + unaffected month
                            // =================================================

                            existingSnapshot != null -> {

                                println(
                                    "DEBUG: CASE 2 - " +
                                            "Existing snapshot + unaffected month"
                                )

                                oldSnapshotForGlobalByClientMonth[
                                    clientId to monthCursor
                                ] =
                                    existingSnapshot

                                monthSnapshot =
                                    ledgerCalculationEngine.propagateBalance(
                                        existingSnapshot =
                                        existingSnapshot,

                                        openingBalance =
                                        runningClosingBalance
                                    )
                            }

                            // =================================================
                            // CASE 3
                            // Missing snapshot + affected month
                            // =================================================

                            isAffectedMonth -> {

                                println(
                                    "DEBUG: CASE 3 - " +
                                            "Missing snapshot + affected month"
                                )

                                val recoveredSnapshot =
                                    getOrRecalculateMonthSnapshot(
                                        transaction = transaction,
                                        clientId = clientId,
                                        yearMonth = monthCursor,
                                        openingBalance = runningClosingBalance,
                                        client = client
                                    )

                                oldSnapshotForGlobalByClientMonth[
                                    clientId to monthCursor
                                ] =
                                    recoveredSnapshot

                                println(
                                    "DEBUG: Recovered OLD snapshot " +
                                            "Client=$clientId " +
                                            "Month=$monthCursor " +
                                            "orders=${recoveredSnapshot.totalOrdersCount} " +
                                            "invoice=${recoveredSnapshot.totalInvoiceAmount} " +
                                            "invoiceWithGst=${recoveredSnapshot.totalInvoiceAmountWithGst} " +
                                            "payments=${recoveredSnapshot.totalPayments} " +
                                            "closing=${recoveredSnapshot.closingBalance}"
                                )

                                println(
                                    "DEBUG: Applying Order delta to recovered " +
                                            "snapshot for Client=$clientId " +
                                            "Month=$monthCursor"
                                )

                                monthSnapshot =
                                    ledgerCalculationEngine.applyOrderDelta(
                                        existingSnapshot =
                                        recoveredSnapshot,

                                        openingBalance =
                                        runningClosingBalance,

                                        oldOrder =
                                        oldOrderForMonth,

                                        newOrder =
                                        newOrderForMonth
                                    )
                            }

                            // =================================================
                            // CASE 4
                            // Missing snapshot + unaffected month
                            // =================================================

                            else -> {

                                println(
                                    "DEBUG: CASE 4 - " +
                                            "Missing snapshot + unaffected month"
                                )

                                val recoveredSnapshot =
                                    getOrRecalculateMonthSnapshot(
                                        transaction = transaction,
                                        clientId = clientId,
                                        yearMonth = monthCursor,
                                        openingBalance = runningClosingBalance,
                                        client = client
                                    )

                                oldSnapshotForGlobalByClientMonth[
                                    clientId to monthCursor
                                ] =
                                    recoveredSnapshot

                                println(
                                    "DEBUG: Recovered snapshot " +
                                            "Client=$clientId " +
                                            "Month=$monthCursor " +
                                            "orders=${recoveredSnapshot.totalOrdersCount} " +
                                            "invoice=${recoveredSnapshot.totalInvoiceAmount} " +
                                            "invoiceWithGst=${recoveredSnapshot.totalInvoiceAmountWithGst} " +
                                            "payments=${recoveredSnapshot.totalPayments} " +
                                            "closing=${recoveredSnapshot.closingBalance}"
                                )

                                monthSnapshot =
                                    recoveredSnapshot
                            }
                        }

                        println(
                            "DEBUG: CALCULATED MONTH SNAPSHOT " +
                                    "client=$clientId " +
                                    "month=$monthCursor " +
                                    "orders=${monthSnapshot.totalOrdersCount} " +
                                    "invoice=${monthSnapshot.totalInvoiceAmount} " +
                                    "invoiceWithGst=${monthSnapshot.totalInvoiceAmountWithGst} " +
                                    "payments=${monthSnapshot.totalPayments} " +
                                    "expenses=${monthSnapshot.totalExpenses} " +
                                    "gst=${monthSnapshot.totalGstAmounts} " +
                                    "discount=${monthSnapshot.totalDiscount} " +
                                    "opening=${monthSnapshot.openingBalance} " +
                                    "closing=${monthSnapshot.closingBalance}"
                        )

                        allCalculatedSnapshots.add(
                            monthSnapshot
                        )

                        runningClosingBalance =
                            monthSnapshot.closingBalance

                        println(
                            "DEBUG: runningClosingBalance AFTER=$runningClosingBalance"
                        )

                        monthCursor =
                            monthCursor.plusMonths(1)
                    }

                    finalClosingBalanceByClient[clientId] =
                        runningClosingBalance

                    println(
                        "DEBUG: Client $clientId " +
                                "finalClosingBalance=$runningClosingBalance"
                    )
                }

                // =============================================================
                // 7. READ ALL GLOBAL SUMMARIES
                // =============================================================

                println("============================================================")
                println("DEBUG: READING GLOBAL SUMMARIES")
                println("============================================================")

                val uniqueGlobalMonths =
                    allCalculatedSnapshots
                        .map {
                            it.yearMonth
                        }
                        .toSet()

                println(
                    "DEBUG: uniqueGlobalMonths=$uniqueGlobalMonths"
                )

                val existingGlobalSummaries =
                    uniqueGlobalMonths.associateWith { yearMonth ->

                        println(
                            "DEBUG: Reading Global Summary $yearMonth"
                        )

                        val data =
                            getGlobalSummaryData(
                                transaction = transaction,
                                yearMonth = yearMonth
                            )

                        println(
                            "DEBUG: EXISTING GLOBAL SUMMARY " +
                                    "month=$yearMonth " +
                                    "data=$data"
                        )

                        data
                    }

                // =============================================================
                // 8. CALCULATE GLOBAL SUMMARIES IN MEMORY
                // =============================================================

                println("============================================================")
                println("DEBUG: START GLOBAL SUMMARY CALCULATION")
                println("============================================================")

                val updatedGlobalSummaries =
                    mutableMapOf<YearMonth, GlobalSummary>()

                affectedClientIds.forEach { clientId ->

                    println("************************************************************")
                    println(
                        "DEBUG: GLOBAL SUMMARY CLIENT LOOP " +
                                "client=$clientId"
                    )
                    println("************************************************************")

                    val clientSnapshots =
                        allCalculatedSnapshots.filter {
                            it.clientId == clientId
                        }

                    println(
                        "DEBUG: Client $clientId has " +
                                "${clientSnapshots.size} calculated snapshots"
                    )

                    clientSnapshots.forEach { newSnapshot ->

                        val yearMonth =
                            newSnapshot.yearMonth

                        println("")
                        println("++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
                        println(
                            "DEBUG: GLOBAL SUMMARY CALCULATION " +
                                    "client=$clientId " +
                                    "month=$yearMonth"
                        )
                        println("++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")

                        /*
                         * Check whether this month has already been
                         * modified in memory.
                         */
                        val alreadyUpdated =
                            updatedGlobalSummaries.containsKey(
                                yearMonth
                            )

                        println(
                            "DEBUG: Month already updated in memory = " +
                                    alreadyUpdated
                        )

                        val existingGlobalData =
                            updatedGlobalSummaries[yearMonth]?.let { summary -> globalSummaryToMap(summary) } ?: (
                                    existingGlobalSummaries[yearMonth] ?: emptyMap())
                        
                        println("DEBUG: GLOBAL EXISTING DATA = $existingGlobalData")

                        val oldSnapshot =
                            if (existingGlobalData.isEmpty()) {
                                null
                            } else {
                                oldSnapshotForGlobalByClientMonth[
                                    clientId to yearMonth
                                ] ?: getEffectiveSnapshot(
                                    clientId = clientId,
                                    yearMonth = yearMonth,
                                    snapshots =
                                    snapshotsByClient[
                                        clientId
                                    ].orEmpty()
                                )
                            }

                        println("DEBUG: GLOBAL OLD SNAPSHOT = $oldSnapshot")
                        println("DEBUG: GLOBAL NEW SNAPSHOT = $newSnapshot")
                        println(
                            "DEBUG: GLOBAL DELTA INPUT " +
                                    "client=$clientId " +
                                    "month=$yearMonth " +
                                    "oldOrders=${oldSnapshot?.totalOrdersCount ?: 0} " +
                                    "newOrders=${newSnapshot.totalOrdersCount} " +
                                    "oldInvoice=${oldSnapshot?.totalInvoiceAmount ?: 0L} " +
                                    "newInvoice=${newSnapshot.totalInvoiceAmount} " +
                                    "oldInvoiceWithGst=${oldSnapshot?.totalInvoiceAmountWithGst ?: 0L} " +
                                    "newInvoiceWithGst=${newSnapshot.totalInvoiceAmountWithGst} " +
                                    "oldPayments=${oldSnapshot?.totalPayments ?: 0L} " +
                                    "newPayments=${newSnapshot.totalPayments} " +
                                    "oldClosing=${oldSnapshot?.closingBalance ?: 0L} " +
                                    "newClosing=${newSnapshot.closingBalance}"
                        )

                        val updatedSummary =
                            globalSummaryService.calculateGlobalSummaryDelta(
                                yearMonth = yearMonth,
                                existingData = existingGlobalData,
                                oldSnapshot = oldSnapshot,
                                newSnapshot = newSnapshot
                            )
                        println("DEBUG: GLOBAL SUMMARY RESULT = " + updatedSummary)

                        println(
                            "DEBUG: RESULT BREAKDOWN " +
                                    "month=$yearMonth " +
                                    "orders=${updatedSummary.totalOrdersCount} " +
                                    "invoice=${updatedSummary.totalInvoiceAmount} " +
                                    "invoiceWithGst=${updatedSummary.totalInvoiceAmountWithGst} " +
                                    "gst=${updatedSummary.totalGstAmount} " +
                                    "payments=${updatedSummary.totalPayments} " +
                                    "expenses=${updatedSummary.totalExpenses} " +
                                    "discount=${updatedSummary.totalDiscount} " +
                                    "receivable=${updatedSummary.totalReceivableAmount} " +
                                    "advance=${updatedSummary.totalAdvanceAmount}"
                        )

                        updatedGlobalSummaries[
                            yearMonth
                        ] =
                            updatedSummary

                        println(
                            "DEBUG: Stored updatedGlobalSummaries[$yearMonth]"
                        )
                    }
                }

                // =============================================================
                // GLOBAL SUMMARY FINAL DEBUG
                // =============================================================

                println("============================================================")
                println("DEBUG: FINAL UPDATED GLOBAL SUMMARIES")
                println("============================================================")

                updatedGlobalSummaries.forEach { (month, summary) ->

                    println(
                        "DEBUG: FINAL GLOBAL SUMMARY " +
                                "month=$month " +
                                "orders=${summary.totalOrdersCount} " +
                                "invoice=${summary.totalInvoiceAmount} " +
                                "invoiceWithGst=${summary.totalInvoiceAmountWithGst} " +
                                "gst=${summary.totalGstAmount} " +
                                "payments=${summary.totalPayments} " +
                                "expenses=${summary.totalExpenses} " +
                                "discount=${summary.totalDiscount} " +
                                "receivable=${summary.totalReceivableAmount} " +
                                "advance=${summary.totalAdvanceAmount}"
                    )
                }

                // =============================================================
                // 9. NOW START WRITES
                // =============================================================

                println("============================================================")
                println(
                    "DEBUG: All reads/calculations completed. " +
                            "Starting writes."
                )
                println("============================================================")

                // -------------------------------------------------------------
                // Save Order
                // -------------------------------------------------------------

                println(
                    "DEBUG: Saving Order ${order.id}"
                )

                orderRepository.save(
                    transaction,
                    order,
                    existingOrder
                )

                // -------------------------------------------------------------
                // Save Snapshots
                // -------------------------------------------------------------

                allCalculatedSnapshots.forEach { snapshot ->

                    println(
                        "DEBUG: Saving Snapshot " +
                                "client=${snapshot.clientId} " +
                                "month=${snapshot.yearMonth} " +
                                "orders=${snapshot.totalOrdersCount} " +
                                "invoice=${snapshot.totalInvoiceAmount} " +
                                "closing=${snapshot.closingBalance}"
                    )

                    snapshotRepository.saveSnapshot(
                        transaction,
                        snapshot
                    )
                }

                // -------------------------------------------------------------
                // Save Global Summaries
                // -------------------------------------------------------------

                updatedGlobalSummaries.values.forEach { summary ->

                    println(
                        "DEBUG: Saving Global Summary " +
                                "month=${summary.yearMonth} " +
                                "orders=${summary.totalOrdersCount} " +
                                "invoice=${summary.totalInvoiceAmount} " +
                                "receivable=${summary.totalReceivableAmount} " +
                                "advance=${summary.totalAdvanceAmount}"
                    )

                    saveGlobalSummary(
                        transaction,
                        summary
                    )
                }

                // -------------------------------------------------------------
                // Update Clients
                // -------------------------------------------------------------

                finalClosingBalanceByClient.forEach {
                        (clientId, closingBalance) ->

                    val client =
                        clientsById[clientId]

                    if (client != null) {

                        println(
                            "DEBUG: Updating Client $clientId " +
                                    "lastClosingBalance=$closingBalance"
                        )

                        clientRepository.save(
                            transaction,

                            client.copy(
                                lastClosingBalance =
                                closingBalance
                            ),

                            client
                        )
                    }
                }

                // =============================================================
                // 10. RETURN RESULT
                // =============================================================

                println("============================================================")
                println("DEBUG: TRANSACTION CALCULATION COMPLETE")
                println("============================================================")

                LedgerTransactionResult(
                    calculationResult =
                    LedgerCalculationResult(
                        snapshots =
                        allCalculatedSnapshots,

                        affectedMonths =
                        allCalculatedSnapshots
                            .map {
                                it.yearMonth
                            }
                            .toSet(),

                        finalClosingBalance =
                        finalClosingBalanceByClient[
                            newClientId
                        ] ?: 0L
                    ),

                    clientsById =
                    clientsById
                )
            }

        clientMonthlyReportingService.saveAll(
            clientsById =
            transactionResult.clientsById,

            snapshots =
            transactionResult.calculationResult.snapshots
        )

        return transactionResult.calculationResult
    }




//    fun moveOrderAndRecalculateLedgers(
//        order: Order,
//        existingOrder: Order
//    ):LedgerCalculationResult {
//        val transactionResult =transactionExecutor.execute(order.clientId) { transaction ->
//
//            // ---------------------------------------------------------
//            // 1. Read affected Clients
//            // ---------------------------------------------------------
//
//            val oldClientId =
//                existingOrder.clientId
//
//            val newClientId =
//                order.clientId
//
//            val oldClient =
//                clientRepository.findById(
//                    transaction,
//                    oldClientId
//                )
//
//            val newClient =
//                clientRepository.findById(
//                    transaction,
//                    newClientId
//                )
//
//            val oldMonth =
//                YearMonth.from(
//                    existingOrder.orderDate
//                )
//
//            val newMonth =
//                YearMonth.from(
//                    order.orderDate
//                )
//
//            println(
//                "DEBUG: moveOrder - orderId=${order.id} " +
//                        "oldClient=$oldClientId " +
//                        "newClient=$newClientId " +
//                        "oldMonth=$oldMonth " +
//                        "newMonth=$newMonth"
//            )
//
//            // ---------------------------------------------------------
//            // 2. Determine end month and read only required snapshots
//            // ---------------------------------------------------------
//
//            val snapshotsByClient =
//                mutableMapOf<String, List<ClientBalanceSnapshot>>()
//
//            // ---------------------------------------------------------
//            // OLD CLIENT
//            // ---------------------------------------------------------
//
//            val oldLatestTransactionMonth =
//                getEndMonth(
//                    transaction = transaction,
//                    clientId = oldClientId
//                )
//
//            val oldLatestSnapshot =
//                snapshotRepository.findLatestSnapshotForClient(
//                    transaction = transaction,
//                    clientId = oldClientId
//                )
//
//            val oldLatestSnapshotMonth =
//                oldLatestSnapshot?.yearMonth
//
//            val oldEndMonth =
//                listOfNotNull(
//                    oldMonth,
//                    oldLatestSnapshotMonth,
//                    oldLatestTransactionMonth
//                ).maxOrNull()
//                    ?: oldMonth
//
//            val oldSnapshots =
//                snapshotRepository.getSnapshotsForClientInRange(
//                    transaction = transaction,
//                    clientId = oldClientId,
//                    startMonth = oldMonth,
//                    endMonth = oldEndMonth
//                )
//
//            snapshotsByClient[oldClientId] =
//                oldSnapshots
//
//            // ---------------------------------------------------------
//            // NEW CLIENT
//            //
//            // If old and new client are different, read the new client's
//            // snapshots separately.
//            //
//            // If they are the same client, reuse the snapshots already
//            // loaded above.
//            // ---------------------------------------------------------
//
//            val newSnapshots: List<ClientBalanceSnapshot>
//
//            val newEndMonth: YearMonth
//
//            if (newClientId == oldClientId) {
//
//                newSnapshots =
//                    oldSnapshots
//
//                val newLatestSnapshotMonth =
//                    oldLatestSnapshotMonth
//
//                val newLatestTransactionMonth =
//                    oldLatestTransactionMonth
//
//                newEndMonth =
//                    listOfNotNull(
//                        newMonth,
//                        newLatestSnapshotMonth,
//                        newLatestTransactionMonth
//                    ).maxOrNull()
//                        ?: newMonth
//
//                // -----------------------------------------------------
//                // Important:
//                //
//                // The old snapshot range may not cover newMonth if the
//                // new order date is after oldEndMonth.
//                //
//                // Therefore, if newEndMonth extends beyond the already
//                // loaded range, fetch the larger range once.
//                // -----------------------------------------------------
//
//                if (newEndMonth > oldEndMonth) {
//
//                    val expandedSnapshots =
//                        snapshotRepository.getSnapshotsForClientInRange(
//                            transaction = transaction,
//                            clientId = newClientId,
//                            startMonth =
//                            minOf(
//                                oldMonth,
//                                newMonth
//                            ),
//                            endMonth = newEndMonth
//                        )
//
//                    snapshotsByClient[newClientId] =
//                        expandedSnapshots
//                }
//
//            } else {
//
//                val newLatestTransactionMonth =
//                    getEndMonth(
//                        transaction = transaction,
//                        clientId = newClientId
//                    )
//
//                val newLatestSnapshot =
//                    snapshotRepository.findLatestSnapshotForClient(
//                        transaction = transaction,
//                        clientId = newClientId
//                    )
//
//                val newLatestSnapshotMonth =
//                    newLatestSnapshot?.yearMonth
//
//                newEndMonth =
//                    listOfNotNull(
//                        newMonth,
//                        newLatestSnapshotMonth,
//                        newLatestTransactionMonth
//                    ).maxOrNull()
//                        ?: newMonth
//
//                newSnapshots =
//                    snapshotRepository.getSnapshotsForClientInRange(
//                        transaction = transaction,
//                        clientId = newClientId,
//                        startMonth = newMonth,
//                        endMonth = newEndMonth
//                    )
//
//                snapshotsByClient[newClientId] =
//                    newSnapshots
//            }
//
//            // ---------------------------------------------------------
//            // 3. Determine opening balances
//            // ---------------------------------------------------------
//
//            val oldOpeningBalance =
//                snapshotRepository.getOpeningBalanceSeed(
//                    transaction = transaction,
//                    clientId = oldClientId,
//                    startMonth = oldMonth,
//                    fallbackOpeningBalance =
//                    oldClient?.lastClosingBalance ?: 0L
//                )
//
//            val newOpeningBalance =
//                if (newClientId == oldClientId) {
//                    snapshotRepository.getOpeningBalanceSeed(
//                        transaction = transaction,
//                        clientId = newClientId,
//                        startMonth = newMonth,
//                        fallbackOpeningBalance =
//                        oldClient?.lastClosingBalance ?: 0L
//                    )
//                } else {
//                    snapshotRepository.getOpeningBalanceSeed(
//                        transaction = transaction,
//                        clientId = newClientId,
//                        startMonth = newMonth,
//                        fallbackOpeningBalance =
//                        newClient?.lastClosingBalance ?: 0L
//                    )
//                }
//
//            // ---------------------------------------------------------
//            // 4. Calculate OLD client
//            //
//            // The Order is being REMOVED from the old client.
//            // ---------------------------------------------------------
//
//            val oldCalculatedSnapshots =
//                mutableListOf<ClientBalanceSnapshot>()
//
//            val oldSnapshotForGlobalByMonth =
//                mutableMapOf<YearMonth, ClientBalanceSnapshot>()
//
//            var oldRunningClosingBalance =
//                oldOpeningBalance
//
//            var oldMonthCursor =
//                oldMonth
//
//            val oldSnapshotMap =
//                oldSnapshots.associateBy {
//                    it.yearMonth
//                }
//
//            while (!oldMonthCursor.isAfter(oldEndMonth)) {
//
//                val existingSnapshot =
//                    oldSnapshotMap[
//                        oldMonthCursor
//                    ]
//
//                val monthSnapshot: ClientBalanceSnapshot
//
//                if (oldMonthCursor == oldMonth) {
//
//                    if (existingSnapshot != null) {
//
//                        // -------------------------------------------------
//                        // FAST PATH
//                        //
//                        // Existing snapshot already contains the Order.
//                        // Remove only this Order's contribution.
//                        // No Order/Payment/Invoice reads.
//                        // -------------------------------------------------
//
//                        println(
//                            "DEBUG: moveOrder - DELETE delta " +
//                                    "Client=$oldClientId " +
//                                    "Month=$oldMonthCursor"
//                        )
//
//                        oldSnapshotForGlobalByMonth[
//                            oldMonthCursor
//                        ] = existingSnapshot
//
//                        monthSnapshot =
//                            ledgerCalculationEngine.applyOrderDelta(
//                                existingSnapshot = existingSnapshot,
//                                openingBalance = oldRunningClosingBalance,
//                                oldOrder = existingOrder,
//                                newOrder = null
//                            )
//
//                    } else {
//
//                        // -------------------------------------------------
//                        // RECOVERY PATH
//                        //
//                        // Rebuild OLD state first. The existing Order
//                        // is still present in Firestore at this point.
//                        // Then remove its contribution.
//                        // -------------------------------------------------
//
//                        println(
//                            "DEBUG: moveOrder - Missing OLD snapshot " +
//                                    "Client=$oldClientId " +
//                                    "Month=$oldMonthCursor. " +
//                                    "Recovering OLD state."
//                        )
//
//                        val recoveredSnapshot =
//                            getOrRecalculateMonthSnapshot(
//                                transaction = transaction,
//                                clientId = oldClientId,
//                                yearMonth = oldMonthCursor,
//                                openingBalance = oldRunningClosingBalance,
//                                client = oldClient
//                            )
//
//                        oldSnapshotForGlobalByMonth[
//                            oldMonthCursor
//                        ] = recoveredSnapshot
//
//                        monthSnapshot =
//                            ledgerCalculationEngine.applyOrderDelta(
//                                existingSnapshot = recoveredSnapshot,
//                                openingBalance = oldRunningClosingBalance,
//                                oldOrder = existingOrder,
//                                newOrder = null
//                            )
//                    }
//
//                } else {
//
//                    // -------------------------------------------------
//                    // Following months
//                    // -------------------------------------------------
//
//                    monthSnapshot =
//                        if (existingSnapshot != null) {
//
//                            oldSnapshotForGlobalByMonth[
//                                oldMonthCursor
//                            ] = existingSnapshot
//
//                            ledgerCalculationEngine.propagateBalance(
//                                existingSnapshot = existingSnapshot,
//                                openingBalance = oldRunningClosingBalance
//                            )
//
//                        } else {
//
//                            val recoveredSnapshot =
//                                getOrRecalculateMonthSnapshot(
//                                    transaction = transaction,
//                                    clientId = oldClientId,
//                                    yearMonth = oldMonthCursor,
//                                    openingBalance = oldRunningClosingBalance,
//                                    client = oldClient
//                                )
//
//                            oldSnapshotForGlobalByMonth[
//                                oldMonthCursor
//                            ] = recoveredSnapshot
//
//                            recoveredSnapshot
//                        }
//                }
//
//                oldCalculatedSnapshots.add(
//                    monthSnapshot
//                )
//
//                oldRunningClosingBalance =
//                    monthSnapshot.closingBalance
//
//                oldMonthCursor =
//                    oldMonthCursor.plusMonths(1)
//            }
//
//            // ---------------------------------------------------------
//            // 5. Calculate NEW client
//            //
//            // The Order is being ADDED to the new client.
//            // ---------------------------------------------------------
//
//            val newCalculatedSnapshots =
//                mutableListOf<ClientBalanceSnapshot>()
//
//            val newSnapshotForGlobalByMonth =
//                mutableMapOf<YearMonth, ClientBalanceSnapshot>()
//
//            var newRunningClosingBalance =
//                newOpeningBalance
//
//            var newMonthCursor =
//                newMonth
//
//            val newSnapshotMap =
//                newSnapshots.associateBy {
//                    it.yearMonth
//                }
//
//            while (!newMonthCursor.isAfter(newEndMonth)) {
//
//                val existingSnapshot =
//                    newSnapshotMap[
//                        newMonthCursor
//                    ]
//
//                val monthSnapshot: ClientBalanceSnapshot
//
//                if (newMonthCursor == newMonth) {
//
//                    if (existingSnapshot != null) {
//
//                        // -------------------------------------------------
//                        // FAST PATH
//                        //
//                        // Add the Order to the existing snapshot.
//                        // No Order/Payment/Invoice reads.
//                        // -------------------------------------------------
//
//                        println(
//                            "DEBUG: moveOrder - ADD delta " +
//                                    "Client=$newClientId " +
//                                    "Month=$newMonthCursor"
//                        )
//
//                        newSnapshotForGlobalByMonth[
//                            newMonthCursor
//                        ] = existingSnapshot
//
//                        monthSnapshot =
//                            ledgerCalculationEngine.applyOrderDelta(
//                                existingSnapshot = existingSnapshot,
//                                openingBalance = newRunningClosingBalance,
//                                oldOrder = null,
//                                newOrder = order
//                            )
//
//                    } else {
//
//                        // -------------------------------------------------
//                        // RECOVERY PATH
//                        //
//                        // The Order has not been saved yet, so recovery
//                        // represents the OLD state of the new client.
//                        // Then add the new Order.
//                        // -------------------------------------------------
//
//                        println(
//                            "DEBUG: moveOrder - Missing NEW snapshot " +
//                                    "Client=$newClientId " +
//                                    "Month=$newMonthCursor. " +
//                                    "Recovering OLD state."
//                        )
//
//                        val recoveredSnapshot =
//                            getOrRecalculateMonthSnapshot(
//                                transaction = transaction,
//                                clientId = newClientId,
//                                yearMonth = newMonthCursor,
//                                openingBalance = newRunningClosingBalance,
//                                client = newClient
//                            )
//
//                        newSnapshotForGlobalByMonth[
//                            newMonthCursor
//                        ] = recoveredSnapshot
//
//                        monthSnapshot =
//                            ledgerCalculationEngine.applyOrderDelta(
//                                existingSnapshot = recoveredSnapshot,
//                                openingBalance = newRunningClosingBalance,
//                                oldOrder = null,
//                                newOrder = order
//                            )
//                    }
//
//                } else {
//
//                    // -------------------------------------------------
//                    // Following months
//                    // -------------------------------------------------
//
//                    monthSnapshot =
//                        if (existingSnapshot != null) {
//
//                            newSnapshotForGlobalByMonth[
//                                newMonthCursor
//                            ] = existingSnapshot
//
//                            ledgerCalculationEngine.propagateBalance(
//                                existingSnapshot = existingSnapshot,
//                                openingBalance = newRunningClosingBalance
//                            )
//
//                        } else {
//
//                            val recoveredSnapshot =
//                                getOrRecalculateMonthSnapshot(
//                                    transaction = transaction,
//                                    clientId = newClientId,
//                                    yearMonth = newMonthCursor,
//                                    openingBalance = newRunningClosingBalance,
//                                    client = newClient
//                                )
//
//                            newSnapshotForGlobalByMonth[
//                                newMonthCursor
//                            ] = recoveredSnapshot
//
//                            recoveredSnapshot
//                        }
//                }
//
//                newCalculatedSnapshots.add(
//                    monthSnapshot
//                )
//
//                newRunningClosingBalance =
//                    monthSnapshot.closingBalance
//
//                newMonthCursor =
//                    newMonthCursor.plusMonths(1)
//            }
//
//            // ---------------------------------------------------------
//            // 6. Read existing Global Summaries
//            //
//            // Must happen before any writes.
//            // ---------------------------------------------------------
//
//            val allCalculatedSnapshots =
//                oldCalculatedSnapshots + newCalculatedSnapshots
//
//            val uniqueGlobalMonths =
//                allCalculatedSnapshots
//                    .map {
//                        it.yearMonth
//                    }
//                    .toSet()
//
//            val existingGlobalSummaries =
//                uniqueGlobalMonths.associateWith { yearMonth ->
//
//                    println(
//                        "DEBUG: Reading Global Summary $yearMonth"
//                    )
//
//                    getGlobalSummaryData(
//                        transaction = transaction,
//                        yearMonth = yearMonth
//                    )
//                }
//
//            // ---------------------------------------------------------
//            // 7. Calculate updated Global Summaries
//            // ---------------------------------------------------------
//
//            val updatedGlobalSummaries =
//                mutableMapOf<YearMonth, GlobalSummary>()
//
//            val updatedGlobalSummaryData =
//                mutableMapOf<YearMonth, Map<String, Any?>>()
//
//            // ---------------------------------------------------------
//            // OLD CLIENT -> Order removed
//            // ---------------------------------------------------------
//
//            oldCalculatedSnapshots.forEach { newSnapshot ->
//
//                val yearMonth =
//                    newSnapshot.yearMonth
//
//                val existingData =
//                    updatedGlobalSummaryData[
//                        yearMonth
//                    ] ?: existingGlobalSummaries[
//                        yearMonth
//                    ] ?: emptyMap()
//
//                val oldSnapshot =
//                    oldSnapshotForGlobalByMonth[
//                        yearMonth
//                    ] ?: getEffectiveSnapshot(
//                        clientId = oldClientId,
//                        yearMonth = yearMonth,
//                        snapshots = oldSnapshots
//                    )
//
//                val updatedSummary =
//                    globalSummaryService.calculateGlobalSummaryDelta(
//                        yearMonth = yearMonth,
//                        existingData = existingData,
//                        oldSnapshot = oldSnapshot,
//                        newSnapshot = newSnapshot
//                    )
//
//                updatedGlobalSummaries[
//                    yearMonth
//                ] = updatedSummary
//
//                updatedGlobalSummaryData[
//                    yearMonth
//                ] = mapOf(
//                    "totalOrdersCount" to updatedSummary.totalOrdersCount,
//                    "totalInvoiceAmount" to updatedSummary.totalInvoiceAmount,
//                    "totalGstAmount" to updatedSummary.totalGstAmount,
//                    "totalInvoiceAmountWithGst" to updatedSummary.totalInvoiceAmountWithGst,
//                    "totalExpenses" to updatedSummary.totalExpenses,
//                    "totalPayments" to updatedSummary.totalPayments,
//                    "totalReceivableAmount" to updatedSummary.totalReceivableAmount,
//                    "totalAdvanceAmount" to updatedSummary.totalAdvanceAmount,
//                    "netProfit" to updatedSummary.netProfit,
//                    "cashFlow" to updatedSummary.cashFlow,
//                    "totalDiscount" to updatedSummary.totalDiscount,
//                    "updatedAt" to updatedSummary.updatedAt
//                )
//            }
//
//            // ---------------------------------------------------------
//            // NEW CLIENT -> Order added
//            // ---------------------------------------------------------
//
//            newCalculatedSnapshots.forEach { newSnapshot ->
//
//                val yearMonth =
//                    newSnapshot.yearMonth
//
//                val existingData =
//                    updatedGlobalSummaryData[
//                        yearMonth
//                    ] ?: existingGlobalSummaries[
//                        yearMonth
//                    ] ?: emptyMap()
//
//                val oldSnapshot =
//                    newSnapshotForGlobalByMonth[
//                        yearMonth
//                    ] ?: getEffectiveSnapshot(
//                        clientId = newClientId,
//                        yearMonth = yearMonth,
//                        snapshots = newSnapshots
//                    )
//
//                val updatedSummary =
//                    globalSummaryService.calculateGlobalSummaryDelta(
//                        yearMonth = yearMonth,
//                        existingData = existingData,
//                        oldSnapshot = oldSnapshot,
//                        newSnapshot = newSnapshot
//                    )
//
//                updatedGlobalSummaries[
//                    yearMonth
//                ] = updatedSummary
//
//                updatedGlobalSummaryData[
//                    yearMonth
//                ] = mapOf(
//                    "totalOrdersCount" to
//                            updatedSummary.totalOrdersCount,
//
//                    "totalInvoiceAmount" to
//                            updatedSummary.totalInvoiceAmount,
//
//                    "totalGstAmount" to
//                            updatedSummary.totalGstAmount,
//
//                    "totalInvoiceAmountWithGst" to
//                            updatedSummary.totalInvoiceAmountWithGst,
//
//                    "totalExpenses" to
//                            updatedSummary.totalExpenses,
//
//                    "totalPayments" to
//                            updatedSummary.totalPayments,
//
//                    "totalReceivableAmount" to
//                            updatedSummary.totalReceivableAmount,
//
//                    "totalAdvanceAmount" to
//                            updatedSummary.totalAdvanceAmount,
//
//                    "netProfit" to
//                            updatedSummary.netProfit,
//
//                    "cashFlow" to
//                            updatedSummary.cashFlow,
//
//                    "totalDiscount" to
//                            updatedSummary.totalDiscount,
//
//                    "updatedAt" to
//                            updatedSummary.updatedAt
//                )
//            }
//
//            // ---------------------------------------------------------
//            // 8. All reads/calculations completed.
//            //    Start writes.
//            // ---------------------------------------------------------
//
//            println(
//                "DEBUG: moveOrder - All reads/calculations completed. " +
//                        "Starting writes."
//            )
//
//            // ---------------------------------------------------------
//            // 9. Save moved Order
//            // ---------------------------------------------------------
//
//            orderRepository.save(
//                transaction,
//                order,
//                existingOrder
//            )
//
//            // ---------------------------------------------------------
//            // 10. Save OLD client snapshots
//            // ---------------------------------------------------------
//
//            oldCalculatedSnapshots.forEach { snapshot ->
//
//                snapshotRepository.saveSnapshot(
//                    transaction,
//                    snapshot
//                )
//            }
//
//            // ---------------------------------------------------------
//            // 11. Save NEW client snapshots
//            // ---------------------------------------------------------
//
//            newCalculatedSnapshots.forEach { snapshot ->
//
//                snapshotRepository.saveSnapshot(
//                    transaction,
//                    snapshot
//                )
//            }
//
//            // ---------------------------------------------------------
//            // 12. Save Global Summaries
//            // ---------------------------------------------------------
//
//            updatedGlobalSummaries.values.forEach { summary ->
//
//                saveGlobalSummary(
//                    transaction,
//                    summary
//                )
//            }
//
//            // ---------------------------------------------------------
//            // 13. Update OLD client closing balance
//            // ---------------------------------------------------------
//
//            if (oldClient != null) {
//
//                clientRepository.save(
//                    transaction,
//                    oldClient.copy(
//                        lastClosingBalance =
//                        oldRunningClosingBalance
//                    ),
//                    oldClient
//                )
//            }
//
//            // ---------------------------------------------------------
//            // 14. Update NEW client closing balance
//            // ---------------------------------------------------------
//
//            if (newClient != null) {
//
//                clientRepository.save(
//                    transaction,
//                    newClient.copy(
//                        lastClosingBalance =
//                        newRunningClosingBalance
//                    ),
//                    newClient
//                )
//            }
//
//            LedgerTransactionResult(
//                calculationResult = LedgerCalculationResult(
//                    snapshots = allCalculatedSnapshots,
//                    affectedMonths = uniqueGlobalMonths,
//                    finalClosingBalance = newRunningClosingBalance
//                ),
//                clientsById = mapOf(
//                    oldClientId to oldClient,
//                    newClientId to newClient
//                )
//            )
//        }
//
//
//
//        clientMonthlyReportingService.saveAll(
//            clientsById = transactionResult.clientsById,
//            snapshots = transactionResult.calculationResult.snapshots
//        )
//
//        return transactionResult.calculationResult
//    }


    fun deleteOrderAndRecalculateLedger(orderId: String): LedgerCalculationResult {
        val existingOrder = orderRepository.findById(orderId)
            ?: throw IllegalArgumentException("Order not found: $orderId");

        val transactionResult = transactionExecutor.execute(existingOrder.clientId) { transaction ->

            // ---------------------------------------------------------
            // 1. Read existing Order
            // ---------------------------------------------------------

            val existingOrder =
                orderRepository.findById(
                    transaction,
                    orderId
                ) ?: throw IllegalStateException(
                    "Order not found: $orderId"
                )

            // ---------------------------------------------------------
            // 2. Order linked to Invoice cannot be directly deleted
            // ---------------------------------------------------------

            if (existingOrder.invoiceId != null) {
                throw IllegalStateException(
                    "Order cannot be deleted directly because it is linked to Invoice ${existingOrder.invoiceId}."
                )
            }

            // ---------------------------------------------------------
            // 3. Read Client
            // ---------------------------------------------------------

            val clientId =
                existingOrder.clientId

            val client =
                clientRepository.findById(
                    transaction,
                    clientId
                )

            val startMonth =
                YearMonth.from(
                    existingOrder.orderDate
                )

            // ---------------------------------------------------------
            // 4. Determine latest transaction month
            // ---------------------------------------------------------

            val latestStoredTransactionMonth =
                getEndMonth(
                    transaction = transaction,
                    clientId = clientId
                )

            // ---------------------------------------------------------
            // 5. Determine latest snapshot month
            //
            // Read only the latest snapshot instead of all snapshots.
            // ---------------------------------------------------------

            val latestSnapshot =
                snapshotRepository.findLatestSnapshotForClient(
                    transaction = transaction,
                    clientId = clientId
                )

            val latestSnapshotMonth =
                latestSnapshot?.yearMonth

            // ---------------------------------------------------------
            // 6. Determine end month
            // ---------------------------------------------------------

            val endMonth =
                listOfNotNull(
                    startMonth,
                    latestSnapshotMonth,
                    latestStoredTransactionMonth
                ).maxOrNull()
                    ?: startMonth

            // ---------------------------------------------------------
            // 7. Read only snapshots required for calculation range
            // ---------------------------------------------------------

            val snapshots =
                snapshotRepository.getSnapshotsForClientInRange(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    endMonth = endMonth
                )

            val snapshotMap =
                snapshots.associateBy {
                    it.yearMonth
                }

            // ---------------------------------------------------------
            // 8. Determine opening balance
            // ---------------------------------------------------------

            val openingBalance =
                snapshotRepository.getOpeningBalanceSeed(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    fallbackOpeningBalance =
                    client?.lastClosingBalance ?: 0L
                )

            // ---------------------------------------------------------
            // 9. Process affected month + following months
            // ---------------------------------------------------------

            val calculatedSnapshots =
                mutableListOf<ClientBalanceSnapshot>()

            val oldSnapshotForGlobalByMonth =
                mutableMapOf<YearMonth, ClientBalanceSnapshot>()

            var runningClosingBalance =
                openingBalance

            var monthCursor =
                startMonth

            while (!monthCursor.isAfter(endMonth)) {

                val existingSnapshot =
                    snapshotMap[monthCursor]

                val monthSnapshot: ClientBalanceSnapshot

                if (monthCursor == startMonth) {

                    // -------------------------------------------------
                    // Affected month
                    // -------------------------------------------------

                    if (existingSnapshot != null) {

                        // -------------------------------------------------
                        // FAST PATH
                        //
                        // Snapshot exists.
                        // No Order/Payment/Invoice reads required.
                        // Simply remove the existing Order's contribution.
                        // -------------------------------------------------

                        println(
                            "DEBUG: Applying Order DELETE delta " +
                                    "for Client=$clientId " +
                                    "Month=$monthCursor " +
                                    "orderId=${existingOrder.id}"
                        )

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = existingSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.applyOrderDelta(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance,
                                oldOrder = existingOrder,
                                newOrder = null
                            )

                    } else {

                        // -------------------------------------------------
                        // RECOVERY PATH
                        //
                        // Snapshot is missing.
                        // Rebuild OLD state from actual transactions,
                        // then remove the existing Order.
                        // -------------------------------------------------

                        println(
                            "DEBUG: Missing affected snapshot for " +
                                    "Client=$clientId " +
                                    "Month=$monthCursor. " +
                                    "Recovering OLD state first."
                        )

                        val recoveredSnapshot =
                            getOrRecalculateMonthSnapshot(
                                transaction = transaction,
                                clientId = clientId,
                                yearMonth = monthCursor,
                                openingBalance = runningClosingBalance,
                                client = client
                            )

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = recoveredSnapshot

                        println(
                            "DEBUG: Recovered OLD snapshot " +
                                    "Client=$clientId " +
                                    "Month=$monthCursor " +
                                    "orders=${recoveredSnapshot.totalOrdersCount}"
                        )

                        monthSnapshot =
                            ledgerCalculationEngine.applyOrderDelta(
                                existingSnapshot = recoveredSnapshot,
                                openingBalance = runningClosingBalance,
                                oldOrder = existingOrder,
                                newOrder = null
                            )
                    }

                } else {

                    // -------------------------------------------------
                    // Following months
                    // -------------------------------------------------

                    monthSnapshot =
                        if (existingSnapshot != null) {

                            // ---------------------------------------------
                            // Existing snapshot:
                            // only propagate new opening balance.
                            // ---------------------------------------------

                            println(
                                "DEBUG: Propagating snapshot " +
                                        "for Client=$clientId " +
                                        "Month=$monthCursor"
                            )

                            oldSnapshotForGlobalByMonth[
                                monthCursor
                            ] = existingSnapshot

                            ledgerCalculationEngine.propagateBalance(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance
                            )

                        } else {

                            // ---------------------------------------------
                            // Missing snapshot:
                            // recover from actual transactions.
                            // ---------------------------------------------

                            println(
                                "DEBUG: Missing snapshot for " +
                                        "Client=$clientId " +
                                        "Month=$monthCursor. " +
                                        "Recovering from actual transactions."
                            )

                            val recoveredSnapshot =
                                getOrRecalculateMonthSnapshot(
                                    transaction = transaction,
                                    clientId = clientId,
                                    yearMonth = monthCursor,
                                    openingBalance = runningClosingBalance,
                                    client = client
                                )

                            oldSnapshotForGlobalByMonth[
                                monthCursor
                            ] = recoveredSnapshot

                            recoveredSnapshot
                        }
                }

                calculatedSnapshots.add(
                    monthSnapshot
                )

                runningClosingBalance =
                    monthSnapshot.closingBalance

                println(
                    "DEBUG: Client=$clientId " +
                            "Month=$monthCursor " +
                            "Closing=${monthSnapshot.closingBalance}"
                )

                monthCursor =
                    monthCursor.plusMonths(1)
            }

            // ---------------------------------------------------------
            // 10. Read existing Global Summaries
            // ---------------------------------------------------------

            val existingGlobalSummaries =
                calculatedSnapshots.associate { snapshot ->
                    snapshot.yearMonth to
                            getGlobalSummaryData(
                                transaction = transaction,
                                yearMonth = snapshot.yearMonth
                            )
                }

            // ---------------------------------------------------------
            // 11. Calculate updated Global Summaries
            // ---------------------------------------------------------

            val updatedGlobalSummaries =
                calculatedSnapshots.associate { newSnapshot ->
                    val existingData = existingGlobalSummaries[newSnapshot.yearMonth] ?: emptyMap()
                    val oldSnapshot = oldSnapshotForGlobalByMonth[newSnapshot.yearMonth] ?: getEffectiveSnapshot(
                            clientId = clientId,
                            yearMonth = newSnapshot.yearMonth,
                            snapshots = snapshots
                        )

                    newSnapshot.yearMonth to
                            globalSummaryService.calculateGlobalSummaryDelta(
                                yearMonth = newSnapshot.yearMonth,
                                existingData = existingData,
                                oldSnapshot = oldSnapshot,
                                newSnapshot = newSnapshot
                            )
                }

            // ---------------------------------------------------------
            // 12. All reads/calculations completed
            //     Start writes
            // ---------------------------------------------------------

            println(
                "DEBUG: All reads/calculations completed. " +
                        "Starting writes."
            )

            // ---------------------------------------------------------
            // 13. Delete Order
            // ---------------------------------------------------------

            orderRepository.deleteById(
                transaction,
                existingOrder.id
            )

            // ---------------------------------------------------------
            // 14. Save calculated snapshots
            // ---------------------------------------------------------

            calculatedSnapshots.forEach { snapshot ->

                snapshotRepository.saveSnapshot(
                    transaction,
                    snapshot
                )
            }

            // ---------------------------------------------------------
            // 15. Save Global Summaries
            // ---------------------------------------------------------

            updatedGlobalSummaries.values.forEach { summary ->

                saveGlobalSummary(
                    transaction,
                    summary
                )
            }

            // ---------------------------------------------------------
            // 16. Update Client lastClosingBalance
            // ---------------------------------------------------------

            if (client != null) {

                clientRepository.save(
                    transaction,
                    client.copy(
                        lastClosingBalance =
                        runningClosingBalance
                    ),
                    client
                )
            }
            LedgerTransactionResult(
                calculationResult = LedgerCalculationResult(
                    snapshots = calculatedSnapshots,
                    affectedMonths = calculatedSnapshots
                        .map { it.yearMonth }
                        .toSet(),
                    finalClosingBalance = runningClosingBalance
                ),
                clientsById = mapOf(
                    clientId to client
                )
            )
        }
        clientMonthlyReportingService.saveAll(
            clientsById = transactionResult.clientsById,
            snapshots = transactionResult.calculationResult.snapshots
        )

        return transactionResult.calculationResult
    }



    private fun getGlobalSummaryData(
        transaction: Transaction,
        yearMonth: YearMonth
    ): Map<String, Any?> {

        val docRef =
            db.collection("global_summaries")
                .document(yearMonth.toString())

        val document = transaction.get(docRef).get()

        if (!document.exists()) {
            return emptyMap()
        }
        return document.data ?: emptyMap()
    }



    private fun saveGlobalSummary(
        transaction: Transaction,
        summary: GlobalSummary
    ) {
        val docRef =
            db.collection("global_summaries")
                .document(summary.yearMonth)

        transaction.set(
            docRef,
            summary
        )
    }

    private fun getEffectiveSnapshot(
        clientId: String,
        yearMonth: YearMonth,
        snapshots: List<ClientBalanceSnapshot>
    ): ClientBalanceSnapshot {

        val exactSnapshot =
            snapshots.find {
                it.yearMonth == yearMonth
            }

        if (exactSnapshot != null) {
            return exactSnapshot
        }

        val nearestSnapshot =
            snapshots.minByOrNull {
                abs(
                    ChronoUnit.MONTHS.between(
                        it.yearMonth.atDay(1),
                        yearMonth.atDay(1)
                    )
                )
            }

        val balance =
            if (nearestSnapshot != null) {
                if (nearestSnapshot.yearMonth.isBefore(yearMonth)) {
                    nearestSnapshot.closingBalance
                } else {
                    nearestSnapshot.openingBalance
                }
            } else {
                0L
            }

        return ClientBalanceSnapshot(
            clientId = clientId,
            yearMonth = yearMonth,
            openingBalance = balance,
            totalOrdersCount = 0,
            totalInvoiceAmount = 0L,
            totalInvoiceAmountWithGst = 0L,
            totalPayments = 0L,
            totalExpenses = 0L,
            closingBalance = balance,
            totalGstAmounts = 0L,
            totalDiscount = 0L,
            updatedAt = Instant.now()
        )
    }

    fun savePaymentAndRecalculateLedger(
        payment: Payment,
        transactionDate: LocalDate
    ): LedgerCalculationResult {

        val transactionResult = transactionExecutor.execute(payment.clientId) { transaction ->

            // ---------------------------------------------------------
            // 1. Read existing Payment
            // ---------------------------------------------------------

            val existingPayment =
                paymentRepository.findById(
                    transaction,
                    payment.id
                )

            val clientId =
                payment.clientId

            val client =
                clientRepository.findById(
                    transaction,
                    clientId
                )

            val oldMonth =
                existingPayment?.let {
                    YearMonth.from(it.paymentDate)
                }

            val newMonth =
                YearMonth.from(
                    payment.paymentDate
                )

            val affectedMonths =
                linkedSetOf<YearMonth>().apply {
                    oldMonth?.let {
                        add(it)
                    }
                    add(newMonth)
                }

            val startMonth =
                affectedMonths.minOrNull()
                    ?: YearMonth.from(transactionDate)

            // ---------------------------------------------------------
            // 2. Determine latest snapshot month
            //
            // Read only the latest snapshot instead of all snapshots.
            // ---------------------------------------------------------

            val latestSnapshot =
                snapshotRepository.findLatestSnapshotForClient(
                    transaction = transaction,
                    clientId = clientId
                )

            val latestSnapshotMonth =
                latestSnapshot?.yearMonth

            // ---------------------------------------------------------
            // 3. Determine latest transaction month
            // ---------------------------------------------------------

            val latestStoredTransactionMonth =
                getEndMonth(
                    transaction = transaction,
                    clientId = clientId
                )

            // ---------------------------------------------------------
            // 4. Determine end month
            // ---------------------------------------------------------

            val endMonth =
                listOfNotNull(
                    startMonth,
                    latestSnapshotMonth,
                    latestStoredTransactionMonth,
                    newMonth
                ).maxOrNull()
                    ?: startMonth

            // ---------------------------------------------------------
            // 5. Read only snapshots required for calculation range
            // ---------------------------------------------------------

            val snapshots =
                snapshotRepository.getSnapshotsForClientInRange(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    endMonth = endMonth
                )

            val snapshotMap =
                snapshots.associateBy {
                    it.yearMonth
                }

            // ---------------------------------------------------------
            // 6. Determine opening balance
            // ---------------------------------------------------------

            val openingBalance =
                snapshotRepository.getOpeningBalanceSeed(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    fallbackOpeningBalance =
                    client?.lastClosingBalance ?: 0L
                )

            // ---------------------------------------------------------
            // 7. Process affected + following months
            // ---------------------------------------------------------

            val calculatedSnapshots =
                mutableListOf<ClientBalanceSnapshot>()

            val oldSnapshotForGlobalByMonth =
                mutableMapOf<YearMonth, ClientBalanceSnapshot>()

            var runningClosingBalance =
                openingBalance

            var monthCursor =
                startMonth

            while (!monthCursor.isAfter(endMonth)) {

                val existingSnapshot =
                    snapshotMap[monthCursor]

                val isOldPaymentMonth =
                    oldMonth == monthCursor

                val isNewPaymentMonth =
                    newMonth == monthCursor

                val monthSnapshot: ClientBalanceSnapshot

                if (isOldPaymentMonth || isNewPaymentMonth) {

                    // -------------------------------------------------
                    // Affected Payment month
                    // -------------------------------------------------

                    if (existingSnapshot != null) {

                        // -------------------------------------------------
                        // FAST PATH
                        //
                        // Snapshot exists.
                        // No full Payment/Order reads required.
                        // -------------------------------------------------

                        println(
                            "DEBUG: Applying Payment delta " +
                                    "for Client=$clientId " +
                                    "Month=$monthCursor " +
                                    "oldPayment=${existingPayment?.id} " +
                                    "newPayment=${payment.id}"
                        )

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = existingSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.applyPaymentDelta(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance,
                                oldPayment =
                                if (isOldPaymentMonth) {
                                    existingPayment
                                } else {
                                    null
                                },
                                newPayment =
                                if (isNewPaymentMonth) {
                                    payment
                                } else {
                                    null
                                }
                            )

                    } else {

                        // -------------------------------------------------
                        // RECOVERY PATH
                        //
                        // Snapshot is missing.
                        // Recover OLD state first, then apply Payment delta.
                        // -------------------------------------------------

                        println(
                            "DEBUG: Missing affected snapshot for " +
                                    "Client=$clientId " +
                                    "Month=$monthCursor. " +
                                    "Recovering OLD state first."
                        )

                        val recoveredSnapshot =
                            getOrRecalculateMonthSnapshot(
                                transaction = transaction,
                                clientId = clientId,
                                yearMonth = monthCursor,
                                openingBalance = runningClosingBalance,
                                client = client
                            )

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = recoveredSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.applyPaymentDelta(
                                existingSnapshot = recoveredSnapshot,
                                openingBalance = runningClosingBalance,
                                oldPayment =
                                if (isOldPaymentMonth) {
                                    existingPayment
                                } else {
                                    null
                                },
                                newPayment =
                                if (isNewPaymentMonth) {
                                    payment
                                } else {
                                    null
                                }
                            )
                    }

                } else {

                    // -------------------------------------------------
                    // Unaffected month
                    // -------------------------------------------------

                    if (existingSnapshot != null) {

                        // -------------------------------------------------
                        // Existing snapshot:
                        // only propagate opening balance.
                        // -------------------------------------------------

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = existingSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.propagateBalance(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance
                            )

                    } else {

                        // -------------------------------------------------
                        // Missing snapshot:
                        // recover from actual transactions.
                        // -------------------------------------------------

                        val recoveredSnapshot =
                            getOrRecalculateMonthSnapshot(
                                transaction = transaction,
                                clientId = clientId,
                                yearMonth = monthCursor,
                                openingBalance = runningClosingBalance,
                                client = client
                            )

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = recoveredSnapshot

                        monthSnapshot =
                            recoveredSnapshot
                    }
                }

                calculatedSnapshots.add(
                    monthSnapshot
                )

                runningClosingBalance =
                    monthSnapshot.closingBalance

                println(
                    "DEBUG: Client=$clientId " +
                            "Month=$monthCursor " +
                            "Closing=${monthSnapshot.closingBalance}"
                )

                monthCursor =
                    monthCursor.plusMonths(1)
            }

            // ---------------------------------------------------------
            // 8. Read existing Global Summaries
            // ---------------------------------------------------------

            val existingGlobalSummaries =
                calculatedSnapshots.associate { snapshot ->

                    snapshot.yearMonth to
                            getGlobalSummaryData(
                                transaction = transaction,
                                yearMonth = snapshot.yearMonth
                            )
                }

            // ---------------------------------------------------------
            // 9. Calculate updated Global Summaries
            // ---------------------------------------------------------

            val updatedGlobalSummaries =
                calculatedSnapshots.associate { newSnapshot ->

                    val yearMonth =
                        newSnapshot.yearMonth

                    val existingData =
                        existingGlobalSummaries[
                            yearMonth
                        ] ?: emptyMap()

                    val oldSnapshot =
                        oldSnapshotForGlobalByMonth[
                            yearMonth
                        ] ?: getEffectiveSnapshot(
                            clientId = clientId,
                            yearMonth = yearMonth,
                            snapshots = snapshots
                        )

                    yearMonth to
                            globalSummaryService.calculateGlobalSummaryDelta(
                                yearMonth = yearMonth,
                                existingData = existingData,
                                oldSnapshot = oldSnapshot,
                                newSnapshot = newSnapshot
                            )
                }

            // ---------------------------------------------------------
            // 10. All reads/calculations completed.
            //     Start writes.
            // ---------------------------------------------------------

            println(
                "DEBUG: All reads/calculations completed. " +
                        "Starting writes."
            )

            // ---------------------------------------------------------
            // 11. Save Payment
            // ---------------------------------------------------------

            paymentRepository.save(
                transaction,
                payment,
                existingPayment
            )

            // ---------------------------------------------------------
            // 12. Save client snapshots
            // ---------------------------------------------------------

            calculatedSnapshots.forEach { snapshot ->

                snapshotRepository.saveSnapshot(
                    transaction,
                    snapshot
                )
            }

            // ---------------------------------------------------------
            // 13. Save Global Summaries
            // ---------------------------------------------------------

            updatedGlobalSummaries.values.forEach { summary ->

                saveGlobalSummary(
                    transaction,
                    summary
                )
            }

            // ---------------------------------------------------------
            // 14. Update Client lastClosingBalance
            // ---------------------------------------------------------

            if (client != null) {

                clientRepository.save(
                    transaction,
                    client.copy(
                        lastClosingBalance =
                        runningClosingBalance
                    ),
                    client
                )
            }

            // ---------------------------------------------------------
            // 15. Return result
            // ---------------------------------------------------------

            LedgerTransactionResult(
                calculationResult = LedgerCalculationResult(
                    snapshots = calculatedSnapshots,
                    affectedMonths =
                    calculatedSnapshots
                        .map {
                            it.yearMonth
                        }
                        .toSet(),
                    finalClosingBalance =
                    runningClosingBalance
                ),
                clientsById = mapOf(
                    clientId to client
                )
            )
        }
        clientMonthlyReportingService.saveAll(
            clientsById = transactionResult.clientsById,
            snapshots = transactionResult.calculationResult.snapshots
        )

        return transactionResult.calculationResult
    }


    fun deletePaymentAndRecalculateLedger(paymentId: String): LedgerCalculationResult {
        val existingPayment = paymentRepository.findById(paymentId)
            ?: throw IllegalArgumentException("Payment not found: $paymentId")

        val transactionResult = transactionExecutor.execute(existingPayment.clientId) { transaction ->
            val existingPayment = paymentRepository.findById(transaction, paymentId) ?: throw IllegalStateException("Payment not found: $paymentId")
            val client = clientRepository.findById(transaction, existingPayment.clientId)
            val clientId = existingPayment.clientId
            val startMonth = YearMonth.from(existingPayment.paymentDate)

            // ---------------------------------------------------------
            // 1. Determine latest snapshot month
            // ---------------------------------------------------------
            val latestSnapshot = snapshotRepository.findLatestSnapshotForClient(transaction = transaction, clientId = clientId)
            val latestSnapshotMonth = latestSnapshot?.yearMonth
            // ---------------------------------------------------------
            // 2. Determine latest transaction month
            // ---------------------------------------------------------
            val latestStoredTransactionMonth = getEndMonth(transaction = transaction, clientId = clientId)
            // ---------------------------------------------------------
            // 3. Determine end month
            // ---------------------------------------------------------
            val endMonth = listOfNotNull(startMonth, latestSnapshotMonth, latestStoredTransactionMonth).maxOrNull() ?: startMonth
            // ---------------------------------------------------------
            // 4. Read only snapshots required for calculation range
            // ---------------------------------------------------------

            val snapshots = snapshotRepository.getSnapshotsForClientInRange(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    endMonth = endMonth
                )
            //convert list to map base on month --yyyy-mm
            val snapshotMap = snapshots.associateBy { it.yearMonth }
            // ---------------------------------------------------------
            // 5. Determine opening balance
            // ---------------------------------------------------------
            val openingBalance = snapshotRepository.getOpeningBalanceSeed(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    fallbackOpeningBalance = client?.lastClosingBalance ?: 0L
                )

            val calculatedSnapshots = mutableListOf<ClientBalanceSnapshot>()
            val oldSnapshotForGlobalByMonth = mutableMapOf<YearMonth, ClientBalanceSnapshot>()
            var runningClosingBalance = openingBalance
            /*
             * Recalculate affected month and propagate
             * through subsequent months.
             */
            var monthCursor = startMonth

            while (!monthCursor.isAfter(endMonth)) {
                val existingSnapshot = snapshotMap[monthCursor]
                val monthSnapshot: ClientBalanceSnapshot
                if (monthCursor == startMonth) {
                    /*
                     * Affected Payment month
                     */
                    if (existingSnapshot != null) {
                        /*
                         * FAST PATH
                         *
                         * Snapshot already contains the Payment.
                         * Remove it using delta.
                         *
                         * No Orders/Payments collection reads.
                         */
                        oldSnapshotForGlobalByMonth[monthCursor] = existingSnapshot
                        monthSnapshot = ledgerCalculationEngine.applyPaymentDelta(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance,
                                oldPayment = existingPayment,
                                newPayment = null
                            )
                    } else {
                        /*
                         * RECOVERY PATH
                         *
                         * Snapshot is missing.
                         *
                         * First reconstruct the OLD state from
                         * actual Orders/Payments/Invoices, then
                         * remove the Payment using delta.
                         */
                        val recoveredSnapshot = getOrRecalculateMonthSnapshot(
                                transaction = transaction,
                                clientId = clientId,
                                yearMonth = monthCursor,
                                openingBalance = runningClosingBalance,
                                client = client
                            )

                        oldSnapshotForGlobalByMonth[monthCursor] = recoveredSnapshot
                        monthSnapshot = ledgerCalculationEngine.applyPaymentDelta(
                                existingSnapshot = recoveredSnapshot,
                                openingBalance = runningClosingBalance,
                                oldPayment = existingPayment,
                                newPayment = null
                            )
                    }

                } else {
                    /*
                     * Subsequent month
                     */
                    if (existingSnapshot != null) {
                        /*
                         * Snapshot exists.
                         * No transaction reads required.
                         */
                        oldSnapshotForGlobalByMonth[monthCursor] = existingSnapshot

                        monthSnapshot = ledgerCalculationEngine.propagateBalance(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance
                            )

                    } else {
                        /*
                         * Snapshot missing.
                         * Recover from actual transactions.
                         */
                        val recoveredSnapshot = getOrRecalculateMonthSnapshot(
                                transaction = transaction,
                                clientId = clientId,
                                yearMonth = monthCursor,
                                openingBalance = runningClosingBalance,
                                client = client
                            )
                        oldSnapshotForGlobalByMonth[monthCursor] = recoveredSnapshot
                        monthSnapshot = recoveredSnapshot
                    }
                }
                calculatedSnapshots.add(monthSnapshot)
                runningClosingBalance = monthSnapshot.closingBalance
                monthCursor = monthCursor.plusMonths(1)
            }

            /*
             * Global summaries
             */

            val existingGlobalSummaries =
                calculatedSnapshots.associate { snapshot ->
                    snapshot.yearMonth to
                            getGlobalSummaryData(
                                transaction = transaction,
                                yearMonth = snapshot.yearMonth
                            )
                }

            val updatedGlobalSummaries =
                calculatedSnapshots.associate { newSnapshot ->

                    val existingData = existingGlobalSummaries[newSnapshot.yearMonth] ?: emptyMap()
                    val oldSnapshot = oldSnapshotForGlobalByMonth[newSnapshot.yearMonth] ?: getEffectiveSnapshot(
                            clientId = clientId,
                            yearMonth = newSnapshot.yearMonth,
                            snapshots = snapshots
                        )

                    newSnapshot.yearMonth to
                            globalSummaryService.calculateGlobalSummaryDelta(
                                yearMonth = newSnapshot.yearMonth,
                                existingData = existingData,
                                oldSnapshot = oldSnapshot,
                                newSnapshot = newSnapshot
                            )
                }

            /*
             * ---------------------------------------------------------
             * ================================All reads completed.==============================================
             * Start writes.
             * ---------------------------------------------------------
             */
            paymentRepository.deleteById(transaction, existingPayment.id)
            /*
             * Save client snapshots
             */

            calculatedSnapshots.forEach { snapshot ->
                snapshotRepository.saveSnapshot(
                    transaction,
                    snapshot
                )
            }

            /*
             * Save global summaries
             */

            updatedGlobalSummaries.values.forEach { summary ->
                saveGlobalSummary(
                    transaction,
                    summary
                )
            }

            /*
             * Update client's final closing balance
             */

            if (client != null) {
                clientRepository.save(
                    transaction,
                    client.copy(lastClosingBalance = runningClosingBalance), client
                )
            }

            LedgerTransactionResult(
                calculationResult = LedgerCalculationResult(
                    snapshots = calculatedSnapshots,
                    affectedMonths =
                    calculatedSnapshots
                        .map { it.yearMonth }
                        .toSet(),
                    finalClosingBalance =
                    runningClosingBalance
                ),
                clientsById = mapOf(
                    clientId to client
                )
            )
        }
        clientMonthlyReportingService.saveAll(
            clientsById = transactionResult.clientsById,
            snapshots = transactionResult.calculationResult.snapshots
        )

        return transactionResult.calculationResult
    }


    fun saveInvoiceAndRecalculateLedger(
        invoice: Invoice
    ): LedgerCalculationResult {

        val transactionResult = transactionExecutor.execute(invoice.clientId) { transaction ->

            /*
             * Load only the Orders linked to this Invoice.
             */
            val orders =
                invoice.orderIds.map { orderId ->

                    orderRepository.findById(
                        transaction,
                        orderId
                    ) ?: throw IllegalStateException(
                        "Order $orderId not found."
                    )
                }

            val client =
                clientRepository.findById(
                    transaction,
                    invoice.clientId
                )

            val clientId =
                invoice.clientId

            require(orders.isNotEmpty()) {
                "At least one Order is required to create an Invoice."
            }

            /*
             * Validate Orders.
             */
            orders.forEach { order ->

                require(order.clientId == clientId) {
                    "Order ${order.id} does not belong to Client $clientId."
                }

                require(order.invoiceId == null) {
                    "Order ${order.id} is already linked to Invoice ${order.invoiceId}."
                }
            }

            /*
             * All Orders in one Invoice must belong
             * to the same month.
             */
            val ordersMonth =
                orders
                    .map {
                        YearMonth.from(it.orderDate)
                    }
                    .toSet()

            require(ordersMonth.size == 1) {
                "All Orders in a single Invoice must belong to the same month."
            }

            val startMonth =
                ordersMonth.first()

            // ---------------------------------------------------------
            // 1. Determine latest snapshot month
            //
            // Read only the latest snapshot instead of all snapshots.
            // ---------------------------------------------------------

            val latestSnapshot =
                snapshotRepository.findLatestSnapshotForClient(
                    transaction = transaction,
                    clientId = clientId
                )

            val latestSnapshotMonth =
                latestSnapshot?.yearMonth

            // ---------------------------------------------------------
            // 2. Determine latest transaction month
            // ---------------------------------------------------------

            val latestStoredTransactionMonth =
                getEndMonth(
                    transaction = transaction,
                    clientId = clientId
                )

            // ---------------------------------------------------------
            // 3. Determine end month
            // ---------------------------------------------------------

            val endMonth =
                listOfNotNull(
                    startMonth,
                    latestSnapshotMonth,
                    latestStoredTransactionMonth
                ).maxOrNull()
                    ?: startMonth

            // ---------------------------------------------------------
            // 4. Read only snapshots required for calculation range
            // ---------------------------------------------------------

            val snapshots =
                snapshotRepository.getSnapshotsForClientInRange(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    endMonth = endMonth
                )

            val snapshotMap =
                snapshots.associateBy {
                    it.yearMonth
                }

            // ---------------------------------------------------------
            // 5. Determine opening balance
            // ---------------------------------------------------------

            val openingBalance =
                snapshotRepository.getOpeningBalanceSeed(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    fallbackOpeningBalance =
                    client?.lastClosingBalance ?: 0L
                )

            val calculatedSnapshots =
                mutableListOf<ClientBalanceSnapshot>()

            val oldSnapshotForGlobalByMonth =
                mutableMapOf<YearMonth, ClientBalanceSnapshot>()

            var runningClosingBalance =
                openingBalance

            /*
             * Recalculate affected month and propagate
             * through subsequent months.
             */
            var monthCursor =
                startMonth

            while (!monthCursor.isAfter(endMonth)) {

                val existingSnapshot =
                    snapshotMap[monthCursor]

                val monthSnapshot: ClientBalanceSnapshot

                if (monthCursor == startMonth) {

                    /*
                     * Affected Invoice month.
                     */
                    if (existingSnapshot != null) {

                        /*
                         * FAST PATH
                         *
                         * Existing snapshot already contains all
                         * previous invoice discounts.
                         *
                         * Only apply the new Invoice discount.
                         *
                         * No Orders/Payments collection reads.
                         */

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = existingSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.applyDiscountDelta(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance,
                                discountDelta = invoice.discount
                            )

                    } else {

                        /*
                         * RECOVERY PATH
                         *
                         * Snapshot is missing.
                         *
                         * Reconstruct the OLD state first.
                         * The new Invoice has not been saved yet,
                         * so its discount is not included.
                         */

                        val recoveredSnapshot =
                            getOrRecalculateMonthSnapshot(
                                transaction = transaction,
                                clientId = clientId,
                                yearMonth = monthCursor,
                                openingBalance = runningClosingBalance,
                                client = client
                            )

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = recoveredSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.applyDiscountDelta(
                                existingSnapshot = recoveredSnapshot,
                                openingBalance = runningClosingBalance,
                                discountDelta = invoice.discount
                            )
                    }

                } else {

                    /*
                     * Subsequent month.
                     */
                    if (existingSnapshot != null) {

                        /*
                         * Existing snapshot:
                         * only propagate opening balance.
                         */

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = existingSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.propagateBalance(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance
                            )

                    } else {

                        /*
                         * Missing snapshot:
                         * recover from actual transactions.
                         */

                        val recoveredSnapshot =
                            getOrRecalculateMonthSnapshot(
                                transaction = transaction,
                                clientId = clientId,
                                yearMonth = monthCursor,
                                openingBalance = runningClosingBalance,
                                client = client
                            )

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = recoveredSnapshot

                        monthSnapshot =
                            recoveredSnapshot
                    }
                }

                calculatedSnapshots.add(
                    monthSnapshot
                )

                runningClosingBalance =
                    monthSnapshot.closingBalance

                monthCursor =
                    monthCursor.plusMonths(1)
            }

            /*
             * Global summaries.
             */
            val existingGlobalSummaries =
                calculatedSnapshots.associate { snapshot ->

                    snapshot.yearMonth to
                            getGlobalSummaryData(
                                transaction = transaction,
                                yearMonth = snapshot.yearMonth
                            )
                }

            val updatedGlobalSummaries =
                calculatedSnapshots.associate { newSnapshot ->

                    val yearMonth =
                        newSnapshot.yearMonth

                    val existingData =
                        existingGlobalSummaries[
                            yearMonth
                        ] ?: emptyMap()

                    val oldSnapshot =
                        oldSnapshotForGlobalByMonth[
                            yearMonth
                        ] ?: getEffectiveSnapshot(
                            clientId = clientId,
                            yearMonth = yearMonth,
                            snapshots = snapshots
                        )

                    yearMonth to
                            globalSummaryService.calculateGlobalSummaryDelta(
                                yearMonth = yearMonth,
                                existingData = existingData,
                                oldSnapshot = oldSnapshot,
                                newSnapshot = newSnapshot
                            )
                }

            /*
             * ---------------------------------------------------------
             * All reads/calculations completed.
             * Start writes.
             * ---------------------------------------------------------
             */

            /*
             * Save Invoice.
             */
            invoiceRepository.save(
                transaction,
                invoice,
                null
            )

            /*
             * Link Orders to Invoice.
             */
            orders.forEach { order ->

                orderRepository.save(
                    transaction,
                    order.copy(
                        invoiceId = invoice.id
                    ),
                    order
                )
            }

            /*
             * Save snapshots.
             */
            calculatedSnapshots.forEach { snapshot ->

                snapshotRepository.saveSnapshot(
                    transaction,
                    snapshot
                )
            }

            /*
             * Save global summaries.
             */
            updatedGlobalSummaries.values.forEach { summary ->

                saveGlobalSummary(
                    transaction,
                    summary
                )
            }

            /*
             * Update client's final closing balance.
             */
            if (client != null) {

                clientRepository.save(
                    transaction,
                    client.copy(
                        lastClosingBalance =
                        runningClosingBalance
                    ),
                    client
                )
            }

            LedgerTransactionResult(
                calculationResult = LedgerCalculationResult(
                    snapshots = calculatedSnapshots,
                    affectedMonths =
                    calculatedSnapshots
                        .map {
                            it.yearMonth
                        }
                        .toSet(),
                    finalClosingBalance =
                    runningClosingBalance
                ),
                clientsById = mapOf(
                    clientId to client
                )
            )
        }

        clientMonthlyReportingService.saveAll(
            clientsById = transactionResult.clientsById,
            snapshots = transactionResult.calculationResult.snapshots
        )

        return transactionResult.calculationResult
    }


    fun updateInvoiceAndRecalculateLedger(
        invoice: Invoice,
        existingInvoice: Invoice
    ): LedgerCalculationResult {

        val transactionResult = transactionExecutor.execute(invoice.clientId) { transaction ->

            val clientId =
                invoice.clientId

            /*
             * Load only Orders directly involved with this Invoice.
             */
            val oldOrders =
                existingInvoice.orderIds.map { orderId ->

                    orderRepository.findById(
                        transaction,
                        orderId
                    ) ?: throw IllegalStateException(
                        "Order $orderId not found."
                    )
                }

            val newOrders =
                invoice.orderIds.map { orderId ->

                    orderRepository.findById(
                        transaction,
                        orderId
                    ) ?: throw IllegalStateException(
                        "Order $orderId not found."
                    )
                }

            val client =
                clientRepository.findById(
                    transaction,
                    clientId
                )

            require(newOrders.isNotEmpty()) {
                "At least one Order is required for an Invoice."
            }

            /*
             * Validate new Orders.
             */
            newOrders.forEach { order ->

                require(order.clientId == clientId) {
                    "Order ${order.id} does not belong to Client $clientId."
                }

                require(
                    order.invoiceId == null ||
                            order.invoiceId == existingInvoice.id
                ) {
                    "Order ${order.id} is already linked to Invoice ${order.invoiceId}."
                }
            }

            /*
             * All Orders in the updated Invoice must belong
             * to the same month.
             */
            val newOrdersMonths =
                newOrders
                    .map {
                        YearMonth.from(it.orderDate)
                    }
                    .toSet()

            require(newOrdersMonths.size == 1) {
                "All Orders in a single Invoice must belong to the same month."
            }

            /*
             * Determine old and new affected months.
             */
            val oldOrderMonths =
                oldOrders
                    .map {
                        YearMonth.from(it.orderDate)
                    }
                    .toSet()

            val newOrderMonths =
                newOrdersMonths

            val affectedMonths =
                oldOrderMonths + newOrderMonths

            val startMonth =
                affectedMonths.minOrNull()
                    ?: YearMonth.from(invoice.invoiceDate)

            // ---------------------------------------------------------
            // 1. Find latest snapshot only.
            // ---------------------------------------------------------

            val latestSnapshot =
                snapshotRepository.findLatestSnapshotForClient(
                    transaction = transaction,
                    clientId = clientId
                )

            val latestSnapshotMonth =
                latestSnapshot?.yearMonth

            // ---------------------------------------------------------
            // 2. Determine latest stored transaction month.
            // ---------------------------------------------------------

            val latestStoredTransactionMonth =
                getEndMonth(
                    transaction = transaction,
                    clientId = clientId
                )

            // ---------------------------------------------------------
            // 3. Determine the last month that needs propagation.
            // ---------------------------------------------------------

            val endMonth =
                listOfNotNull(
                    startMonth,
                    latestSnapshotMonth,
                    latestStoredTransactionMonth
                ).maxOrNull()
                    ?: startMonth

            // ---------------------------------------------------------
            // 4. Load only snapshots required for this calculation.
            // ---------------------------------------------------------

            val snapshots =
                snapshotRepository.getSnapshotsForClientInRange(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    endMonth = endMonth
                )

            val snapshotMap =
                snapshots.associateBy {
                    it.yearMonth
                }

            // ---------------------------------------------------------
            // 5. Calculate opening balance for first affected month.
            // ---------------------------------------------------------

            val openingBalance =
                snapshotRepository.getOpeningBalanceSeed(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    fallbackOpeningBalance =
                    client?.lastClosingBalance ?: 0L
                )

            val calculatedSnapshots =
                mutableListOf<ClientBalanceSnapshot>()

            val oldSnapshotForGlobalByMonth =
                mutableMapOf<YearMonth, ClientBalanceSnapshot>()

            var runningClosingBalance =
                openingBalance

            var monthCursor =
                startMonth

            while (!monthCursor.isAfter(endMonth)) {

                val existingSnapshot =
                    snapshotMap[monthCursor]

                val isAffectedMonth =
                    monthCursor in affectedMonths

                val monthSnapshot: ClientBalanceSnapshot

                if (isAffectedMonth) {

                    /*
                     * This month has Invoice changes.
                     *
                     * Financial Order/Payment totals have NOT changed.
                     * Only the Invoice discount may have changed.
                     */

                    val oldDiscountForMonth =
                        if (monthCursor in oldOrderMonths) {
                            existingInvoice.discount
                        } else {
                            0L
                        }

                    val newDiscountForMonth =
                        if (monthCursor in newOrderMonths) {
                            invoice.discount
                        } else {
                            0L
                        }

                    val discountDelta =
                        newDiscountForMonth -
                                oldDiscountForMonth

                    if (existingSnapshot != null) {

                        /*
                         * FAST PATH
                         *
                         * Existing snapshot already contains the
                         * old Invoice discount.
                         *
                         * Apply only the discount delta.
                         *
                         * No Orders/Payments collection reads.
                         */

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = existingSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.applyDiscountDelta(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance,
                                discountDelta = discountDelta
                            )

                    } else {

                        /*
                         * RECOVERY PATH
                         *
                         * Snapshot is missing.
                         *
                         * Recover the OLD state first, then apply
                         * the Invoice discount delta.
                         */

                        val recoveredSnapshot =
                            getOrRecalculateMonthSnapshot(
                                transaction = transaction,
                                clientId = clientId,
                                yearMonth = monthCursor,
                                openingBalance = runningClosingBalance,
                                client = client
                            )

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = recoveredSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.applyDiscountDelta(
                                existingSnapshot = recoveredSnapshot,
                                openingBalance = runningClosingBalance,
                                discountDelta = discountDelta
                            )
                    }

                } else if (existingSnapshot != null) {

                    /*
                     * No Invoice change in this month.
                     *
                     * Reuse snapshot totals and only propagate
                     * the new opening balance.
                     */

                    oldSnapshotForGlobalByMonth[
                        monthCursor
                    ] = existingSnapshot

                    monthSnapshot =
                        ledgerCalculationEngine.propagateBalance(
                            existingSnapshot = existingSnapshot,
                            openingBalance = runningClosingBalance
                        )

                } else {

                    /*
                     * Rare recovery case:
                     * snapshot is missing, so rebuild from
                     * actual transactions.
                     */

                    val recoveredSnapshot =
                        getOrRecalculateMonthSnapshot(
                            transaction = transaction,
                            clientId = clientId,
                            yearMonth = monthCursor,
                            openingBalance = runningClosingBalance,
                            client = client
                        )

                    oldSnapshotForGlobalByMonth[
                        monthCursor
                    ] = recoveredSnapshot

                    monthSnapshot =
                        recoveredSnapshot
                }

                calculatedSnapshots.add(
                    monthSnapshot
                )

                runningClosingBalance =
                    monthSnapshot.closingBalance

                monthCursor =
                    monthCursor.plusMonths(1)
            }

            /*
             * Global summaries.
             */
            val existingGlobalSummaries =
                calculatedSnapshots.associate { snapshot ->

                    snapshot.yearMonth to
                            getGlobalSummaryData(
                                transaction = transaction,
                                yearMonth = snapshot.yearMonth
                            )
                }

            val updatedGlobalSummaries =
                calculatedSnapshots.associate { newSnapshot ->

                    val yearMonth =
                        newSnapshot.yearMonth

                    val existingData =
                        existingGlobalSummaries[
                            yearMonth
                        ] ?: emptyMap()

                    val oldSnapshot =
                        oldSnapshotForGlobalByMonth[
                            yearMonth
                        ] ?: getEffectiveSnapshot(
                            clientId = clientId,
                            yearMonth = yearMonth,
                            snapshots = snapshots
                        )

                    yearMonth to
                            globalSummaryService.calculateGlobalSummaryDelta(
                                yearMonth = yearMonth,
                                existingData = existingData,
                                oldSnapshot = oldSnapshot,
                                newSnapshot = newSnapshot
                            )
                }

            /*
             * ---------------------------------------------------------
             * All reads/calculations completed.
             * Start writes.
             * ---------------------------------------------------------
             */

            /*
             * Save Invoice.
             */
            invoiceRepository.save(
                transaction,
                invoice,
                existingInvoice
            )

            /*
             * Remove Invoice relationship from Orders
             * that were removed from the Invoice.
             */
            val oldOrderIds =
                existingInvoice.orderIds.toSet()

            val newOrderIds =
                invoice.orderIds.toSet()

            val removedOrderIds =
                oldOrderIds - newOrderIds

            removedOrderIds.forEach { orderId ->

                val order =
                    oldOrders.find {
                        it.id == orderId
                    }

                if (
                    order != null &&
                    order.invoiceId == existingInvoice.id
                ) {

                    orderRepository.save(
                        transaction,
                        order.copy(
                            invoiceId = null
                        ),
                        order
                    )
                }
            }

            /*
             * Add Invoice relationship to new Orders.
             */
            newOrders.forEach { order ->

                orderRepository.save(
                    transaction,
                    order.copy(
                        invoiceId = invoice.id
                    ),
                    order
                )
            }

            /*
             * Save snapshots.
             */
            calculatedSnapshots.forEach { snapshot ->

                snapshotRepository.saveSnapshot(
                    transaction,
                    snapshot
                )
            }

            /*
             * Save global summaries.
             */
            updatedGlobalSummaries.values.forEach { summary ->

                saveGlobalSummary(
                    transaction,
                    summary
                )
            }

            /*
             * Update Client closing balance.
             */
            if (client != null) {

                clientRepository.save(
                    transaction,
                    client.copy(
                        lastClosingBalance =
                        runningClosingBalance
                    ),
                    client
                )
            }

            LedgerTransactionResult(
                calculationResult = LedgerCalculationResult(
                    snapshots = calculatedSnapshots,
                    affectedMonths =
                    calculatedSnapshots
                        .map {
                            it.yearMonth
                        }
                        .toSet(),
                    finalClosingBalance =
                    runningClosingBalance
                ),
                clientsById = mapOf(
                    clientId to client
                )
            )
        }

        clientMonthlyReportingService.saveAll(
            clientsById = transactionResult.clientsById,
            snapshots = transactionResult.calculationResult.snapshots
        )

        return transactionResult.calculationResult
    }

    fun deleteInvoiceAndRecalculateLedger(
        invoiceId: String
    ): LedgerCalculationResult {
        val existingInvoice = invoiceRepository.findById(invoiceId)
            ?: throw IllegalArgumentException("Invoice not found: $invoiceId")
        val transactionResult = transactionExecutor.execute(existingInvoice.clientId) { transaction ->

            val existingInvoice =
                invoiceRepository.findById(
                    transaction,
                    invoiceId
                ) ?: throw IllegalStateException(
                    "Invoice not found: $invoiceId"
                )

            /*
             * Load only Orders linked to this Invoice.
             */
            val orders =
                existingInvoice.orderIds.map { orderId ->

                    orderRepository.findById(
                        transaction,
                        orderId
                    ) ?: throw IllegalStateException(
                        "Order $orderId not found."
                    )
                }

            require(orders.isNotEmpty()) {
                "Invoice ${existingInvoice.id} has no linked Orders."
            }

            val clientId =
                existingInvoice.clientId

            val client =
                clientRepository.findById(
                    transaction,
                    clientId
                )

            /*
             * Invoice discount belongs to the Order month,
             * not invoiceDate.
             */
            val affectedMonths =
                orders
                    .map {
                        YearMonth.from(it.orderDate)
                    }
                    .toSet()

            require(affectedMonths.size == 1) {
                "All Orders in a single Invoice must belong to the same month."
            }

            val startMonth =
                affectedMonths.first()

            // ---------------------------------------------------------
            // 1. Find latest snapshot only.
            // ---------------------------------------------------------

            val latestSnapshot =
                snapshotRepository.findLatestSnapshotForClient(
                    transaction = transaction,
                    clientId = clientId
                )

            val latestSnapshotMonth =
                latestSnapshot?.yearMonth

            // ---------------------------------------------------------
            // 2. Determine latest stored transaction month.
            // ---------------------------------------------------------

            val latestStoredTransactionMonth =
                getEndMonth(
                    transaction = transaction,
                    clientId = clientId
                )

            // ---------------------------------------------------------
            // 3. Determine last month that needs propagation.
            // ---------------------------------------------------------

            val endMonth =
                listOfNotNull(
                    startMonth,
                    latestSnapshotMonth,
                    latestStoredTransactionMonth
                ).maxOrNull()
                    ?: startMonth

            // ---------------------------------------------------------
            // 4. Load only snapshots required for this calculation.
            // ---------------------------------------------------------

            val snapshots =
                snapshotRepository.getSnapshotsForClientInRange(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    endMonth = endMonth
                )

            val snapshotMap =
                snapshots.associateBy {
                    it.yearMonth
                }

            // ---------------------------------------------------------
            // 5. Calculate opening balance for affected month.
            // ---------------------------------------------------------

            val openingBalance =
                snapshotRepository.getOpeningBalanceSeed(
                    transaction = transaction,
                    clientId = clientId,
                    startMonth = startMonth,
                    fallbackOpeningBalance =
                    client?.lastClosingBalance ?: 0L
                )

            val calculatedSnapshots =
                mutableListOf<ClientBalanceSnapshot>()

            val oldSnapshotForGlobalByMonth =
                mutableMapOf<YearMonth, ClientBalanceSnapshot>()

            var runningClosingBalance =
                openingBalance

            /*
             * Recalculate affected month and propagate
             * through subsequent months.
             */
            var monthCursor =
                startMonth

            while (!monthCursor.isAfter(endMonth)) {

                val existingSnapshot =
                    snapshotMap[monthCursor]

                val monthSnapshot: ClientBalanceSnapshot

                if (monthCursor == startMonth) {

                    /*
                     * Affected Invoice month.
                     */

                    if (existingSnapshot != null) {

                        /*
                         * FAST PATH
                         *
                         * Existing snapshot already contains
                         * this Invoice's discount.
                         *
                         * Remove only this Invoice's discount.
                         *
                         * No Orders/Payments collection reads.
                         */

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = existingSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.applyDiscountDelta(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance,
                                discountDelta =
                                -existingInvoice.discount
                            )

                    } else {

                        /*
                         * RECOVERY PATH
                         *
                         * Snapshot is missing.
                         *
                         * Recover OLD state first, then remove
                         * this Invoice's discount.
                         */

                        val recoveredSnapshot =
                            getOrRecalculateMonthSnapshot(
                                transaction = transaction,
                                clientId = clientId,
                                yearMonth = monthCursor,
                                openingBalance = runningClosingBalance,
                                client = client
                            )

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = recoveredSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.applyDiscountDelta(
                                existingSnapshot = recoveredSnapshot,
                                openingBalance = runningClosingBalance,
                                discountDelta =
                                -existingInvoice.discount
                            )
                    }

                } else {

                    /*
                     * Subsequent month.
                     */

                    if (existingSnapshot != null) {

                        /*
                         * Existing snapshot:
                         * only propagate opening balance.
                         */

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = existingSnapshot

                        monthSnapshot =
                            ledgerCalculationEngine.propagateBalance(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance
                            )

                    } else {

                        /*
                         * Missing snapshot:
                         * recover from actual transactions.
                         */

                        val recoveredSnapshot =
                            getOrRecalculateMonthSnapshot(
                                transaction = transaction,
                                clientId = clientId,
                                yearMonth = monthCursor,
                                openingBalance = runningClosingBalance,
                                client = client
                            )

                        oldSnapshotForGlobalByMonth[
                            monthCursor
                        ] = recoveredSnapshot

                        monthSnapshot =
                            recoveredSnapshot
                    }
                }

                calculatedSnapshots.add(
                    monthSnapshot
                )

                runningClosingBalance =
                    monthSnapshot.closingBalance

                monthCursor =
                    monthCursor.plusMonths(1)
            }

            /*
             * Global summaries.
             */
            val existingGlobalSummaries =
                calculatedSnapshots.associate { snapshot ->

                    snapshot.yearMonth to
                            getGlobalSummaryData(
                                transaction = transaction,
                                yearMonth = snapshot.yearMonth
                            )
                }

            val updatedGlobalSummaries =
                calculatedSnapshots.associate { newSnapshot ->

                    val yearMonth =
                        newSnapshot.yearMonth

                    val existingData =
                        existingGlobalSummaries[
                            yearMonth
                        ] ?: emptyMap()

                    val oldSnapshot =
                        oldSnapshotForGlobalByMonth[
                            yearMonth
                        ] ?: getEffectiveSnapshot(
                            clientId = clientId,
                            yearMonth = yearMonth,
                            snapshots = snapshots
                        )

                    yearMonth to
                            globalSummaryService.calculateGlobalSummaryDelta(
                                yearMonth = yearMonth,
                                existingData = existingData,
                                oldSnapshot = oldSnapshot,
                                newSnapshot = newSnapshot
                            )
                }

            /*
             * ---------------------------------------------------------
             * All reads/calculations completed.
             * Start writes.
             * ---------------------------------------------------------
             */

            /*
             * Delete Invoice.
             */
            invoiceRepository.deleteById(
                transaction,
                existingInvoice.id
            )

            /*
             * Remove Invoice relationship from linked Orders.
             */
            orders.forEach { order ->

                if (order.invoiceId == existingInvoice.id) {

                    orderRepository.save(
                        transaction,
                        order.copy(
                            invoiceId = null
                        ),
                        order
                    )
                }
            }

            /*
             * Save snapshots.
             */
            calculatedSnapshots.forEach { snapshot ->

                snapshotRepository.saveSnapshot(
                    transaction,
                    snapshot
                )
            }

            /*
             * Save global summaries.
             */
            updatedGlobalSummaries.values.forEach { summary ->

                saveGlobalSummary(
                    transaction,
                    summary
                )
            }

            /*
             * Update client's final closing balance.
             */
            if (client != null) {

                clientRepository.save(
                    transaction,
                    client.copy(
                        lastClosingBalance =
                        runningClosingBalance
                    ),
                    client
                )
            }

            LedgerTransactionResult(
                calculationResult = LedgerCalculationResult(
                    snapshots = calculatedSnapshots,
                    affectedMonths =
                    calculatedSnapshots
                        .map {
                            it.yearMonth
                        }
                        .toSet(),
                    finalClosingBalance =
                    runningClosingBalance
                ),
                clientsById = mapOf(
                    clientId to client
                )
            )
        }

        clientMonthlyReportingService.saveAll(
            clientsById = transactionResult.clientsById,
            snapshots = transactionResult.calculationResult.snapshots
        )

        return transactionResult.calculationResult
    }

    private fun getMonthSnapshot(
        transaction: Transaction,
        clientId: String,
        yearMonth: YearMonth
    ): ClientBalanceSnapshot? {

        return snapshotRepository.getSnapshot(
            transaction,
            clientId,
            yearMonth
        )
    }

    private fun getOrRecalculateMonthSnapshot(
        transaction: Transaction,
        clientId: String,
        yearMonth: YearMonth,
        openingBalance: Long,
        client: Client?
    ): ClientBalanceSnapshot {
        println("DEBUG: getOrRecalculateMonthSnapshot - Checking for existing snapshot for $clientId in $yearMonth")
        val existingSnapshot = getMonthSnapshot(
            transaction = transaction,
            clientId = clientId,
            yearMonth = yearMonth
        )

        // Snapshot exists → cheap propagation.
        if (existingSnapshot != null) {
            return ledgerCalculationEngine.propagateBalance(
                existingSnapshot = existingSnapshot,
                openingBalance = openingBalance
            )
        }

        // Snapshot is missing → recovery case.
        // Read only this month's actual transactions.
        val orders = orderRepository.findAllForClientAndMonth(
            transaction,
            clientId,
            yearMonth
        )
        val payments = paymentRepository.findAllForClientAndMonth(
            transaction,
            clientId,
            yearMonth
        )
        val orderIds = orders.map { it.id }.toSet()
        val invoices = invoiceRepository.findAllForClientAndOrderIds(
            transaction,
            clientId,
            orderIds
        )
        val discountByMonth = invoices.sumOf { invoice -> invoice.discount }
        val totalOrdersCount = orders.size
        val totalInvoiceAmountWithGst = orders.sumOf { it.totalInvoiceAmount }
        val totalExpenses = orders.sumOf { it.totalExpense }
        val totalGstAmounts = orders.sumOf { it.totalGstAmount }
        val totalPayments = payments.sumOf { it.amount }
        val totalInvoiceAmount = totalInvoiceAmountWithGst - totalGstAmounts
        val closingBalance = openingBalance +
                    totalInvoiceAmountWithGst -
                    totalPayments -
                    discountByMonth

        println("DEBUG: getOrRecalculateMonthSnapshot - Recalculated snapshot for $clientId in $yearMonth: openingBalance=$openingBalance, totalOrdersCount=$totalOrdersCount, totalInvoiceAmount=$totalInvoiceAmount, totalInvoiceAmountWithGst=$totalInvoiceAmountWithGst, totalPayments=$totalPayments, totalExpenses=$totalExpenses, closingBalance=$closingBalance, totalGstAmounts=$totalGstAmounts, totalDiscount=$discountByMonth")
        return ClientBalanceSnapshot(
            clientId = clientId,
            yearMonth = yearMonth,
            openingBalance = openingBalance,
            totalOrdersCount = totalOrdersCount,
            totalInvoiceAmount = totalInvoiceAmount,
            totalInvoiceAmountWithGst = totalInvoiceAmountWithGst,
            totalPayments = totalPayments,
            totalExpenses = totalExpenses,
            closingBalance = closingBalance,
            totalGstAmounts = totalGstAmounts,
            totalDiscount = discountByMonth,
            updatedAt = Instant.now()
        )
    }


    private fun getEndMonth(transaction: Transaction, clientId: String): YearMonth? {
        val latestOrderMonth = orderRepository.findLatestMonthForClient(transaction, clientId)
        val latestPaymentMonth = paymentRepository.findLatestMonthForClient(transaction, clientId)
        return listOfNotNull(latestOrderMonth, latestPaymentMonth).maxOrNull()
    }

    private fun getOpeningBalance(startMonth: YearMonth,
        snapshots: List<ClientBalanceSnapshot>,
        client: Client?
    ): Long {
        val exactStartSnapshot = snapshots.find { it.yearMonth == startMonth }

        if (exactStartSnapshot != null) {
            return exactStartSnapshot.openingBalance
        }

        if (snapshots.isNotEmpty()) {
            val nearestSnapshot =
                snapshots.minByOrNull {
                    kotlin.math.abs(
                        ChronoUnit.MONTHS.between(
                            it.yearMonth.atDay(1),
                            startMonth.atDay(1)
                        )
                    )
                }

            if (nearestSnapshot != null) {

                return if (
                    nearestSnapshot.yearMonth.isBefore(startMonth)
                ) {
                    nearestSnapshot.closingBalance
                } else {
                    nearestSnapshot.openingBalance
                }
            }
        }

        return client?.lastClosingBalance ?: 0L
    }


    private fun globalSummaryToMap(
        summary: GlobalSummary
    ): Map<String, Any> {
        return mapOf(
            "yearMonth" to summary.yearMonth.toString(),
            "totalOrdersCount" to summary.totalOrdersCount,
            "totalInvoiceAmount" to summary.totalInvoiceAmount,
            "totalInvoiceAmountWithGst" to summary.totalInvoiceAmountWithGst,
            "totalPayments" to summary.totalPayments,
            "totalExpenses" to summary.totalExpenses,
            "totalDiscount" to summary.totalDiscount,
            "totalGstAmount" to summary.totalGstAmount,
            "totalReceivableAmount" to summary.totalReceivableAmount,
            "totalAdvanceAmount" to summary.totalAdvanceAmount,
            "netProfit" to summary.netProfit,
            "cashFlow" to summary.cashFlow,
            "updatedAt" to summary.updatedAt
        )
    }


    fun updateCurrentMonthOnly(clientId: String, transactionDate: LocalDate) {
        val yearMonth = YearMonth.from(transactionDate)
        val snapshot = snapshotRepository.getSnapshot(clientId, yearMonth) ?: return

        val orders = orderRepository.findAllForMonth(yearMonth.year, yearMonth.monthValue)
            .filter {
                it.clientId == clientId
            }

        val totalExpenses = orders.sumOf { it.totalExpense }

        val updatedSnapshot = snapshot.copy(
            totalExpenses = totalExpenses,
            updatedAt = Instant.now()
        )

        snapshotRepository.saveSnapshot(updatedSnapshot)
        globalSummaryService.recomputeGlobalSummaryForMonth(yearMonth)
    }
}

