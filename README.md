import java.time.Instant

data class GlobalSummary(
    val yearMonth: String,                  // e.g., "2025-01" or "2026 (YTD)"
    val totalOrdersCount: Int = 0,          // Us mahine ke total orders ka count
    val totalInvoiceAmount: Long = 0L,      // Without GST (Total Orders - GST)
    val totalGstAmount: Long = 0L,          // Total GST collected
    val totalInvoiceAmountWithGst: Long = 0L, // Total Orders amount with GST
    val totalExpenses: Long = 0L,           // Total expenses in that month
    val totalPayments: Long = 0L,   // Total payments received
    val totalReceivableAmount: Long = 0L,   // Total closing balance > 0 (Receivables)
    val totalAdvanceAmount: Long = 0L,      // Total closing balance < 0 (Advances, absolute sum)
    val netProfit: Long = 0L,          // OverAllNetProfit (Invoice - Expenses)
    val cashFlow: Long = 0L,   // cashFlow (Payments - Expenses)
    val totalDiscount: Long = 0L,          // Total discount given
    val updatedAt: Instant = Instant.now()
)  private fun getGlobalSummaryData(
        transaction: Transaction,
        yearMonth: YearMonth
    ): Map<String, Any?> ?{

        val docRef =
            db.collection("global_summaries")
                .document(yearMonth.toString())

        val document = transaction.get(docRef).get()

        if (!document.exists()) {
            return null
        }
        return document.data ?: emptyMap()
    }

    private fun saveGlobalSummary(
        transaction: Transaction,
        summary: GlobalSummary) {
        val docRef =
            db.collection("global_summaries")
                .document(summary.yearMonth)

        transaction.set(docRef, summary)
    }



    cashFlow
-8000
(number)
netProfit
3000
(number)
totalAdvanceAmount
0
(number)
totalDiscount
0
(number)
totalExpenses
8000
(number)
totalGstAmount
550
(number)
totalInvoiceAmount
11000
(number)
totalInvoiceAmountWithGst
11550
(number)
totalOrdersCount
2
(number)
totalPayments
0
(number)
totalReceivableAmount
11550
(number)
arrow_drop_down
updatedAt
(map)
epochSecond
1789470995
(number)
nano
569877200
(number)
yearMonth
"2026-01"
(string)
