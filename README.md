1. Three windows, not one
MONTHS_TO_CHECK_IN_LEDGER_ENGINE = 36 currently does two unrelated jobs. Split into three constants:

Constant	Value	Drives	Call sites
MONTHS_EDITABLE	8	isInsideModificationMonthRange — order/payment/invoice create-edit-delete	8
MONTHS_REPORTABLE	60	report horizon, dense-row retention	new
MONTHS_TO_CHECK_IN_LEDGER_ENGINE	36, unchanged	repository scan bounds (recovery path)	7
Plus: balance-seed lookups for dormant clients must be unbounded — use the existing findLatestSnapshotBefore (already unbounded) rather than getOpeningBalanceSeed's ±12-month radius, which silently falls back for a client inactive longer than a year.

The property this buys you, and it's the whole basis of the design: with an 8-month edit window, any month older than 8 months can never change. Its report rows get computed once and are frozen forever. That's what makes 5-year reports cheap — you're not keeping 60 months live, you're keeping 8 live and 52 frozen.

Useful side effect: worst-case propagation drops from 36 months to 8, so a single mutation writes ~8 snapshots + 8 report rows + 8 summaries ≈ 24 writes instead of ~108. Much safer against Firestore's 500-write transaction cap.

2. global_summaries/{yyyy-MM} — split fields by flow vs stock
60 docs total. Two field classes that behave oppositely at a month boundary:

Flow — zeroed when a new month is seeded:
totalOrdersCount, totalInvoiceAmount, totalGstAmount, totalInvoiceAmountWithGst, totalPayments, totalExpenses, totalDiscount, netProfit, cashFlow

Stock — copied forward from M−1 when a new month is seeded:
actualReceivable, receivableInitialComponent, receivableTransactionComponent, actualAdvance, advanceInitialComponent, advanceTransactionComponent, receivableClientCount, advanceClientCount, settledClientCount, totalClientCount

On the component split you approved — the exact identity, taken over only the clients whose actualBalance > 0:


actualReceivable = receivableInitialComponent + receivableTransactionComponent
So your ₹10,00,000 February receivable reconciles as "₹3,20,000 came from initial opening balances, ₹6,80,000 from transaction activity" — the breakdown you asked for. Components may individually be negative (a client with initial +₹30,000 and transaction −₹10,000 is still ₹20,000 receivable); that's correct, not a bug. Same three fields mirrored for advance.

3. global_reporting/{yyyy-MM}/clients/{clientId} — dense, every client
Per your "maintain all clients which exist in clients/". Existing ClientMonthlyReporting gains:

openingBalance — missing today, and you need it for the opening column
status — RECEIVABLE / ADVANCE / SETTLED
receivable, advance — both non-negative, never one signed number
isActive — had activity this month
carriedForwardFrom — which month an inactive client's balance came from
4. Three mechanisms
Month seeding. When month M's docs don't exist, create them from M−1: copy stock forward, zero flow. Global summary = 1 read + 1 write. Client rows = 500 reads + 500 writes, batched outside any transaction (Firestore batch limit is 500, so 2 batches). Replaces rebuildGlobalSummaryForMonth, which is the current source of both the inactive-client undercount and the double-count bug.

Old/new state delta. Your formula, unchanged, applied at the seven existing clientMonthlyReportingService.saveAll(...) call sites. Inactive clients contribute a provably zero delta, so the other 400 are never read or written.

One-time backfill. Walk months forward once holding 500 clients' running state in memory — one collectionGroup("months") query per month plus batched writes. ~15,000 docs for your current 2-3 years.

5. What the read paths cost after this
Report	Doc reads
Monthly summary	1
Yearly summary	12
Receivables drill-down for a month	only receivable clients (~118), sorted by amount and paginated in the DB via composite index (status, actualBalance DESC)
Single client drill-down	1
6. Open items
netProfit — include discount? Currently totalInvoiceAmount − totalExpenses. You list "Total profit". I'd make it totalInvoiceAmount − totalExpenses − totalDiscount, but that changes numbers you're reporting today. Your call.
initialOpeningBalance edits break the frozen-history property. Changing it shifts actualBalance in every month, including the 52 frozen ones — so it must rewrite that client's rows across up to 60 months and re-delta each month's summary. updateInitialOpeningBalance rewrites the rows today but does not touch global summary receivable/advance, so totals drift. This has to be a batched job, not part of the ledger transaction. Is editing initialOpeningBalance actually a supported operation, or can we restrict it?
saveAll runs after the transaction commits. If it fails, snapshots are committed and reports are silently stale — true today too. Needs an idempotent recompute-month admin endpoint as the repair tool.
app/ has no tests, and this touches the write path at seven sites in a 3,900-line coordinator. Everything verified against the emulator by hand.
7. Sequencing
Phase 1 — split the three constants. Contained, no behaviour change except the 8-month gate.
Phase 2 — flow/stock split + seeding in global_summaries. Fixes inactive-client totals and the double-count. Correctness win, no new collections.
Phase 3 — dense client rows + Reporting Update Layer with old/new deltas at the seven sites.
Phase 4 — backfill job, firestore.indexes.json, drill-down endpoints.
Each phase is independently verifiable on the emulator, and Phase 2 delivers correct totals even if we stop there.

Confirm the approach — and answer (1) and (2) — and tell me whether to start at Phase 1 or do 1+2 together.
