package com.clientledger.core.repository.snapshot

import com.clientledger.core.domain.snapshot.ClientBalanceSnapshot
import com.google.cloud.firestore.Transaction
import java.time.YearMonth

interface SnapshotRepository {

    fun getSnapshot(clientId: String, yearMonth: YearMonth): ClientBalanceSnapshot?

    fun saveSnapshot(snapshot: ClientBalanceSnapshot)

    fun getInitialClientOpeningBalance(clientId: String): Long

    fun getAllSnapshotsForClient(clientId: String): List<ClientBalanceSnapshot>

    // Transaction-aware methods
    fun getSnapshot(
        transaction: Transaction,
        clientId: String,
        yearMonth: YearMonth
    ): ClientBalanceSnapshot?

    fun saveSnapshot(
        transaction: Transaction,
        snapshot: ClientBalanceSnapshot
    )

    fun getAllSnapshotsForClient(
        transaction: Transaction,
        clientId: String
    ): List<ClientBalanceSnapshot>

    fun findLatestSnapshotForClient(transaction: Transaction, clientId: String): ClientBalanceSnapshot?

    fun getSnapshotsForClientInRange(transaction: Transaction, clientId: String, startMonth: YearMonth, endMonth: YearMonth): List<ClientBalanceSnapshot>

    fun getOpeningBalanceSeed(transaction: Transaction, clientId: String, startMonth: YearMonth, fallbackOpeningBalance: Long): Long

    fun findLatestSnapshotBefore(
        clientId: String,
        yearMonth: YearMonth
    ): ClientBalanceSnapshot?

}
