package com.clientledger.core.service.summary

import com.clientledger.core.domain.ClientType
import com.clientledger.core.domain.GlobalSummary
import com.clientledger.core.domain.SummaryClientIndex
import com.clientledger.core.pagination.PageResult
import com.clientledger.core.repository.summary.GlobalSummaryRepository
import com.clientledger.core.repository.summary.SummaryClientIndexRepository
import org.springframework.stereotype.Service

@Service
class SummaryService(
    private val globalSummaryRepository: GlobalSummaryRepository,
    private val summaryClientIndexRepository: SummaryClientIndexRepository
) {

    fun getMonthSummary(
        ownerId: String,
        yearMonth: String
    ): GlobalSummary? {

        return globalSummaryRepository.find(
            ownerId = ownerId,
            yearMonth = yearMonth
        )
    }

    fun getYearSummary(
        ownerId: String,
        year: Int
    ): GlobalSummary {

        require(ownerId.isNotBlank()) {
            "ownerId must not be blank"
        }

        require(year in 1..9999) {
            "year must be between 1 and 9999"
        }

        var lastAvailableSummary: GlobalSummary? = null

        var totalCashFlow = 0L
        var totalNetProfit = 0L
        var totalInvoiceAmount = 0L
        var totalInvoiceAmountWithGst = 0L
        var totalGstAmount = 0L
        var totalExpenseAmount = 0L
        var totalPaymentAmount = 0L

        for (month in 1..12) {

            val yearMonth =
                "%04d-%02d".format(year, month)

            val summary =
                globalSummaryRepository.find(
                    ownerId = ownerId,
                    yearMonth = yearMonth
                )
                    ?: continue

            lastAvailableSummary = summary

            totalCashFlow += summary.cashFlow
            totalNetProfit += summary.netProfit
            totalInvoiceAmount += summary.totalInvoiceAmount
            totalInvoiceAmountWithGst += summary.totalInvoiceAmountWithGst
            totalGstAmount += summary.totalGstAmount
            totalExpenseAmount += summary.totalExpenseAmount
            totalPaymentAmount += summary.totalPaymentAmount
        }

        val lastSummary = lastAvailableSummary
            ?: return GlobalSummary(
                ownerId = ownerId,
                yearMonth = year.toString()
            )

        return GlobalSummary(
            ownerId = ownerId,
            yearMonth = year.toString(),

            receivableClientCount =
            lastSummary.receivableClientCount,

            advanceClientCount =
            lastSummary.advanceClientCount,

            settledClientCount =
            lastSummary.settledClientCount,

            totalReceivableAmount =
            lastSummary.totalReceivableAmount,

            totalAdvanceAmount =
            lastSummary.totalAdvanceAmount,

            cashFlow = totalCashFlow,
            netProfit = totalNetProfit,

            totalInvoiceAmount =
            totalInvoiceAmount,

            totalInvoiceAmountWithGst =
            totalInvoiceAmountWithGst,

            totalGstAmount =
            totalGstAmount,

            totalExpenseAmount =
            totalExpenseAmount,

            totalPaymentAmount =
            totalPaymentAmount
        )
    }

    fun getClients(
        ownerId: String,
        yearMonth: String,
        status: ClientType,
        size: Int,
        cursor: String?
    ): PageResult<SummaryClientIndex> {

        require(
            status == ClientType.RECEIVABLE ||
                    status == ClientType.ADVANCE
        ) {
            "status must be RECEIVABLE or ADVANCE"
        }

        return summaryClientIndexRepository.findPage(
            ownerId = ownerId,
            yearMonth = yearMonth,
            size = size,
            cursor = cursor
        )
    }

}
