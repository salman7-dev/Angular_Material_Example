================================================================================
1. WHY WE ARE DOING THIS
================================================================================

The goal is a month-wise client balance report plus drill-down, for ~300-500
clients over a 5-year reporting horizon, without rescanning years of snapshots
on every report open.

Three problems with the original system:

  a) global_summaries receivable/advance totals were built from only the
     snapshots that existed in that month. A client with no activity in
     February contributed nothing, so the February receivable total silently
     excluded ~400 of 500 clients.

  b) global_reporting/{yyyy-MM}/clients only has rows for clients active that
     month, so drill-down could not reconcile with the top-level total.

  c) Reports recalculated by looping clients and re-reading snapshots, which is
     slow and expensive in Firestore document reads.

Target architecture (agreed):

    Ledger + Snapshots      = source of truth
    global_summaries        = fast aggregate read model (1 doc per month)
    global_reporting/...    = per-client drill-down read model
    old state vs new state  = safe incremental update mechanism
    8-month window          = controlled recalculation boundary

IMPORTANT: this is Firestore, not MongoDB. There is no aggregation pipeline,
no $lookup, no joins, no server-side GROUP BY. Only firebase-admin 9.2.0 is on
the classpath. Any plan that assumes a one-query aggregation is not buildable
here. This was checked, not assumed.


================================================================================
2. DECISIONS MADE (and the reasoning, which is the part that gets lost)
================================================================================

2.1 THREE WINDOWS, NOT ONE
    MONTHS_EDITABLE                 = 8   gates order/payment/invoice mutations
    MONTHS_REPORTABLE               = 60  reporting horizon (5 years)
    MONTHS_TO_CHECK_IN_LEDGER_ENGINE = 36  repository scan bounds (UNCHANGED)

    One constant used to do two unrelated jobs across 15 call sites. Only the
    mutation gate became 8.

    DO NOT shrink MONTHS_TO_CHECK_IN_LEDGER_ENGINE to 8. The recovery path
    (getOrRecalculateMonthSnapshot) rebuilds a month from actual orders and
    payments, and a dormant client's opening balance is seeded from snapshots
    that can be years old. An 8-month scan would find nothing and rebuild those
    months as ZERO. There is a comment on the constant and a test asserting
    MONTHS_TO_CHECK_IN_LEDGER_ENGINE > MONTHS_EDITABLE to stop someone
    "tidying" it.

2.2 THE EDIT WINDOW GATES THE DISCOUNT, NOT THE INVOICE
    closing = opening + totalInvoiceAmountWithGst - totalPayments - totalDiscount

    The ORDER contributes the amount. The INVOICE contributes only the discount.
    So a zero-discount invoice is ledger-neutral and is safe outside the window.
    A discount change moves a balance and is gated.

    Also gated: moving a NON-ZERO discount to an order in a different month.
    The amount is unchanged but it moves between two months' balances, so
    "discount unchanged" alone was not a sufficient test.

2.3 THE DATE CHECK IS THE FIRST GATE IN THE APPLICATION
    Rejected before an id is generated, before any read, before the ledger.
    For invoices the order must be read first to learn its date, so "first"
    means first validation, not before any read.

2.4 FLOW vs STOCK (the key insight of Phase 2)
    FLOW  fields = activity in the month. A new month starts at ZERO.
          orders, invoice, gst, payments, expenses, discount, netProfit, cashFlow
    STOCK fields = point-in-time balances. A new month starts at the PREVIOUS
          MONTH'S VALUE, not zero, because 400 inactive clients still owe what
          they owed last month.
          receivable, advance, and the client counts

    The old code treated receivable/advance as flow. That is the root cause of
    problem (a) above.

2.5 WHY THE OLD/NEW-STATE DELTA IS EXACT, NOT AN APPROXIMATION
    receivable(M) = receivable(M-1) + SUM over affected clients of
                    [ max(newActual,0) - max(oldActual,0) ]

    An inactive client's balance does not move between M-1 and M, so their
    bucket contribution is unchanged and their delta is exactly ZERO. That is
    why one mutation can touch one client and leave the totals correct for all
    500. It is not a heuristic.

    A single signed delta does NOT work. +5,000 -> -3,000 is -5,000 receivable
    AND +3,000 advance; a combined -8,000 cannot express that.

2.6 ACTUAL BALANCE IS THE HEADLINE
    actualBalance = initialOpeningBalance + transactionClosingBalance
      > 0 -> receivable,  < 0 -> advance,  = 0 -> settled

    global_summaries now stores the ACTUAL figure plus the two components:
      totalReceivableAmount = transactionReceivableAmount + initialReceivableComponent
      totalAdvanceAmount    = transactionAdvanceAmount    + initialAdvanceComponent

    Advance components are stored sign-flipped so the identity holds. An
    individual component may be negative (positive opening balance + deeply
    negative ledger balance = advance overall). That is correct, not a bug.

    CONSEQUENCE: /api/global-summary/monthly receivable/advance numbers CHANGE.
    They now include initialOpeningBalance. This was a deliberate decision.

2.7 netProfit INCLUDES DISCOUNT
    netProfit = totalInvoiceAmount - totalExpenses - totalDiscount
    Previously discount was ignored. This changes previously reported numbers.

2.8 initialOpeningBalance EDITS PROPAGATE 8 MONTHS ONLY
    ClientMonthlyReporting.initialOpeningBalance is stored PER MONTH. Bounding
    propagation to 8 months means older rows keep their old value, so a 2-year
    receivable trend will show a step at the 8-month boundary matching no
    transaction. Accepted as a "no retroactive restatement" policy.
    NOT YET IMPLEMENTED - see pending item P4.

2.9 DENSE CLIENT ROWS (decided, not yet built - Phase 3)
    Every client in clients/ gets a row every month. Sparse rows look cheaper
    on writes but are far more expensive on reads: Firestore has no
    per-group-latest, so a sparse drill-down must read ALL rows <= target and
    reduce (~10,000+ docs), whereas dense reads only the ~118 receivable
    clients via a composite index, sorted and paginated in the DB.


================================================================================
3. FINISHED
================================================================================

PHASE 1 - EDITABLE WINDOW  (COMPLETE, TESTED)

  - Split one constant into three (see 2.1).
  - Date gate moved to first statement in OrderService.saveOrder and
    PaymentService.savePayment, above id generation.
  - InvoiceService: editable-period check moved from 5th validator to 1st;
    added a non-empty-orders guard since it no longer runs after
    validateInvoiceFields.
  - Invoice rules are now discount-driven (see 2.2).

  Resulting invoice behaviour:
    create outside window, no discount              -> allowed
    create outside window, with discount            -> 400
    update outside window, discount unchanged       -> allowed  (was 400)
    update outside window, discount added/removed   -> 400
    update outside window, non-zero discount moves month -> 400
    orders and payments outside window, any op      -> 400

  24 tests, 0 skipped. Mutation-checked: disabling the balance-neutral
  early-return fails exactly the 3 "update is allowed" tests, for the right
  reason. So the tests have teeth.

PHASE 2 - FLOW/STOCK SPLIT + MONTH SEEDING  (CODE COMPLETE, UNVERIFIED)

  - GlobalSummary: flow/stock split documented in the type; 8 new stock fields;
    receivable/advance redefined to actual.
  - BalanceStock: the sign logic in ONE place so the component identities hold
    by construction.
  - recomputeGlobalSummaryForMonth: now authoritative. Walks every client,
    carries balances forward, adds initialOpeningBalance, fills components and
    counts. This is what makes inactive clients appear. Expensive - not for the
    read path.
  - seedGlobalSummaryForMonth: NEW. Carries stock forward from M-1, zeroes
    flow. ONE read, not 500. Walks back through gaps (a month where nobody
    transacted leaves no document). Must be called in a transaction's READ
    phase, before any write.
  - calculateGlobalSummaryDelta: old-state/new-state on actual balance, plus
    components and counts. initialOpeningBalance is a REQUIRED parameter - done
    deliberately so the compiler enumerated all 7 call sites instead of relying
    on a manual hunt.
  - recomputeGlobalSummaryForRange + POST /api/global-summary/recompute:
    migration and repair.
  - Coordinator: all 7 sites changed from
        if (missing) rebuild else delta
    to  seed-then-ALWAYS-delta.
    This also FIXES the double-count bug documented in CLAUDE.md: the first
    client used to trigger a rebuild that already summed the second client, and
    the second then applied a delta on top of a total it was already part of.
  - rebuildGlobalSummaryForMonth: deprecated, now zero call sites. It omits
    inactive clients and leaves the component fields at zero, so its output no
    longer matches any other write path. Do not reintroduce it.


================================================================================
4. FILES CHANGED  (12 total: 9 modified, 3 new)
================================================================================

MODIFIED - Phase 1
  app/src/main/kotlin/com/clientledger/core/utils/CommonUtils.kt
  app/src/main/kotlin/com/clientledger/core/service/order/OrderService.kt
  app/src/main/kotlin/com/clientledger/core/service/payment/PaymentService.kt
  app/src/main/kotlin/com/clientledger/core/service/invoice/InvoiceService.kt

MODIFIED - Phase 2
  app/src/main/kotlin/com/clientledger/core/domain/summary/GlobalSummary.kt
  app/src/main/kotlin/com/clientledger/core/service/summary/GlobalSummaryService.kt
  app/src/main/kotlin/com/clientledger/core/transaction/LedgerTransactionCoordinator.kt
  app/src/main/kotlin/com/clientledger/core/api/GlobalSummaryController.kt

MODIFIED - build
  app/build.gradle.kts            (added io.mockk:mockk:1.14.11, test scope)

NEW - tests (first tests app/ has ever had)
  app/src/test/kotlin/com/clientledger/core/utils/CommonUtilsTest.kt            (9)
  app/src/test/kotlin/com/clientledger/core/service/invoice/InvoiceEditablePeriodTest.kt (10)
  app/src/test/kotlin/com/clientledger/core/service/EditableWindowGateTest.kt   (5)

mockk was required, not preferred: LedgerTransactionCoordinator and
PdfGenerationService are final Kotlin classes with large constructors, so hand
written fakes would have needed production classes opened up. It pulls from
Maven Central - relevant if building somewhere without network.


================================================================================
5. PENDING
================================================================================

BLOCKING - do these before trusting Phase 2

  P1. RUN THE MIGRATION. Existing global_summaries docs hold the OLD semantics
      (transaction-only receivable, no component fields) and seeding reads the
      previous month. Until this runs, the new path chains off stale values.
      Oldest-first is enforced by the range function.

        POST /api/global-summary/recompute
             ?startYearMonth=2024-01&endYearMonth=2026-09

      Run it against a COPY first.

  P2. VERIFY ON THE EMULATOR. Phase 2 is compile-verified and the arithmetic is
      reasoned through, but none of it has been executed. It is money math.
      Spot-check the identities after migrating:
        totalReceivableAmount == transactionReceivableAmount + initialReceivableComponent
        totalAdvanceAmount    == transactionAdvanceAmount    + initialAdvanceComponent

  P3. NO TESTS ON THE BALANCE MATH. The 24 tests cover Phase 1 only. Nothing
      covers the delta, the seed, the engine, or the coordinator.

INCOMPLETE PHASE 2 WORK

  P4. initialOpeningBalance edits not wired (decision 2.8).
      FirestoreClientMonthlyReportingRepository.updateInitialOpeningBalance
      still rewrites EVERY reporting row with no month filter, and does not
      adjust global summary receivable/advance at all, so totals drift.
      Needs: 8-month bound + old/new-state delta on the summary stock.

  P5. Client creation does not touch summary stock. A new client's
      initialOpeningBalance never enters the receivable total, and
      totalClientCount is carried forward by the seed, so a new client does not
      appear until a recompute. Same old/new delta shape, where old state is
      "client did not exist".

KNOWN BUGS STILL OPEN (all pre-existing, all listed in CLAUDE.md)

  P6. GlobalSummaryService.getGlobalSummaryForYear - reports 0 receivables when
      the final month has no doc. Disagrees with /drill-down/receivables, which
      carries balances forward.
  P7. GlobalSummaryService.getDiscountDrillDown - with month == null it targets
      December only and reports that month's discount as the whole year's.
  P8. LedgerTransactionCoordinator.updateCurrentMonthOnly - "?: return"
      silently drops expense edits when the snapshot is absent, and runs
      non-transactionally against live snapshots.
  P9. PaymentController imports com.google.api.pathtemplate.ValidationException
      (line 5), so the duplicate-id path at line 18 yields 500 instead of 400.
      NOTE: this does NOT affect the editable-window path - PaymentService
      imports the correct exception and those rejections map to 400 properly.

OPEN DECISIONS - need an answer before the related work can proceed

  D1. Invoice delete is still blanket-rejected outside the 8-month window
      (InvoiceService line ~113, comment "Historical Invoice cannot be
      deleted"). By the discount rule in 2.2, deleting a ZERO-discount invoice
      is balance-neutral and arguably should be allowed; deleting a discounted
      one should stay blocked. Left strict deliberately - destructive path, and
      it was never explicitly requested.

  D2. CLAUDE.md is now out of date: it states "app/ has no tests". The rest of
      that note (no coverage of the balance math) is still accurate.

RISK

  R1. This project is NOT under git ("Is a git repository: false"). There is no
      diff or rollback for these 12 files, including a 3,900-line coordinator
      with no coverage of its balance math. Strongly recommend "git init" and
      committing the current green state BEFORE starting Phase 3.


================================================================================
6. UPCOMING PHASES
================================================================================

PHASE 3 - DENSE CLIENT ROWS + REPORTING UPDATE LAYER

  - Add to ClientMonthlyReporting:
        openingBalance      (MISSING today - global_reporting has no opening
                             balance field at all, and the report needs it)
        status              RECEIVABLE | ADVANCE | SETTLED
        receivable, advance both non-negative, never one signed number
        isActive            had activity this month
        carriedForwardFrom  which month an inactive client's balance came from
  - Make global_reporting/{yyyy-MM}/clients DENSE - every client in clients/.
  - Reporting Update Layer wired at the 7 EXISTING
    clientMonthlyReportingService.saveAll(...) call sites in the coordinator.
    That is already the seam; do NOT refactor the month-walk loops (see
    CLAUDE.md).
  - Must be IDEMPOTENT on re-write: a balance-neutral invoice update still
    runs the ledger walk and rewrites out-of-window months with identical
    numbers. Phase 2/3 must not assume old months are never touched - the
    frozen-history property applies to VALUES, not to writes.
  - Note: saveAll runs AFTER the transaction commits. If it fails, snapshots
    are committed and reports are silently stale. POST /recompute is the repair
    tool.
  - Firestore caps a transaction at 500 writes. With the 8-month window a
    single mutation is ~8 snapshots + 8 report rows + 8 summaries = ~24 writes,
    which is comfortable. A 500-client dense rollover is exactly AT the cap, so
    rollover must be a batched NON-transactional job, never inside the ledger
    transaction.

PHASE 4 - THE REPORT THIS WORK WAS FOR

  - One-time backfill of dense rows across the 60-month window. Walk months
    forward holding running state in memory: one collectionGroup query per
    month plus batched writes. Months older than 8 are immutable, so they are
    materialised once and never recomputed.
  - Create firestore.indexes.json - THE REPO HAS NO INDEX CONFIG AT ALL.
    firebase.json declares only "rules": "firestore.rules", and that file is
    missing too. Nothing has ever been validated against real Firestore's index
    requirements because the emulator serves any query. Needs:
      * collection-group (status ASC, actualBalance DESC) for cheap drill-down
      * the EXISTING undeclared queries: collectionGroup("months") with
        clientId == + yearMonth range + orderBy, and collectionGroup("clients")
        with clientId == + yearMonth range
  - GET /api/reports/monthly-balance/{yearMonth}
      Returns EVERY client with: openingBalance, that month's activity,
      closingBalance, initialOpeningBalance, actualBalance, receivable,
      advance, status, isActive, balanceSource, plus summary totals.
      Fallback chain per client:
        1. row/snapshot exists for target month     -> use it
        2. else latest earlier month                -> carry closing forward,
                                                       activity all zero
        3. else no snapshot ever                    -> opening = closing = 0
        4. last activity older than the window      -> clients.lastClosingBalance
                                                       as backstop
        then actualBalance = initialOpeningBalance + closing decides the bucket
      Include a balanceSource field (EXACT_MONTH / CARRIED_FORWARD /
      INITIAL_ONLY / LAST_CLOSING_BACKSTOP) so each branch can be verified on
      the emulator rather than trusted.
  - Drill-down showing the initial-vs-transaction split, so a 10,00,000
    receivable reconciles as "3,20,000 from opening balances, 6,80,000 from
    ledger activity".

RECOMMENDED SEQUENCING
  P1 + P2 + P3 (migrate, verify, then tests for the delta/seed arithmetic)
  BEFORE Phase 3, because Phase 3 builds directly on Phase 2's correctness.
  Also do R1 (git init) first.


================================================================================
7. COMMANDS
================================================================================

Two processes are always needed.

  # 1. Firestore emulator (port 8080, UI 4000) - must be running first
  firebase emulators:start --project client-ledger-new-code-2026-09

  # 2. App (port 8081). JDK 17 or 21.
  $env:JAVA_HOME="C:\Program Files\Java\jdk-17"
  ./gradlew :app:bootRun

  ./gradlew build                  # compile + test everything
  ./gradlew :app:compileKotlin     # fast type check
  ./gradlew :app:test              # the 24 tests

Do not trust "BUILD SUCCESSFUL" alone as proof tests ran - it passes with zero
tests. Check the counts:

  Get-ChildItem app\build\test-results\test\*.xml | ForEach-Object {
    $x=[xml](Get-Content $_.FullName)
    "{0}: tests={1} failures={2} skipped={3}" -f `
      $x.testsuite.name,$x.testsuite.tests,$x.testsuite.failures,$x.testsuite.skipped }

Project id and emulator host are HARD-CODED in FirestoreConfig.kt
(127.0.0.1:8080, NoCredentials). There is no profile or env-var switch.


================================================================================
8. MANUAL CHECKS FOR PHASE 1 (no automated emulator coverage)
================================================================================

  1. Order dated ~6 months ago                      -> succeeds
  2. Order dated ~12 months ago                     -> 400
  3. Invoice, no discount, on ~12-month-old order    -> succeeds
  4. Invoice with discount on ~12-month-old order    -> 400
  5. Update that invoice keeping discount 0          -> succeeds (was 400)
  6. Set a discount on that old invoice              -> 400

DATA WARNING: anything dated more than 8 months ago is now READ-ONLY. If any of
the existing 2-3 years of data still needs correcting, do it BEFORE this goes
live, or those months become uneditable. POST /recompute can repair summaries
but cannot reopen the edit window.
Written to progress.txt (~370 lines), structured for a cold restart:

Section	Contents
1. Why	The three original problems; the Firestore-not-MongoDB constraint
2. Decisions	9 decisions with reasoning — the part that gets lost
3. Finished	Phase 1 (complete, tested) and Phase 2 (code complete, unverified)
4. Files changed	All 12, grouped by phase
5. Pending	P1–P9 + open decisions D1–D2 + risk R1
6. Upcoming	Phases 3 and 4 in implementable detail
7. Commands	Build/run, plus the test-count check
8. Manual checks	The 6 Phase 1 emulator cases
I deliberately recorded the reasoning, not just the outcomes, since that's what won't survive a few days — specifically: why the scan-bound constant must stay at 36 (silent zero-rebuild for dormant clients), why a zero-discount invoice is ledger-neutral, why the inactive-client delta is exactly zero rather than approximately, and why a single signed delta can't express a receivable→advance crossing.

Also captured explicitly, because they're easy to lose and expensive to rediscover:

P1 is blocking — the migration must run before Phase 2 is correct at all, since seeding reads the previous month and existing docs hold old semantics.
Two APIs change numbers — netProfit now subtracts discount, and receivable/advance now include initialOpeningBalance.
The frozen-history caveat — a balance-neutral invoice update still rewrites out-of-window months, so Phase 3's reporting layer must be idempotent rather than assuming old months are untouched.
The 500-write cap — why dense rollover must be a batched non-transactional job.
The data warning — anything older than 8 months is now read-only, so corrections to existing data must happen before go-live.
One thing I'd do before anything else: git init and commit the current green state (R1 in the file). 12 files changed including the 3,900-line coordinator, no version control, and no coverage of the balance math — Phase 3 builds straight on top of that.



CODE:

package com.clientledger.core.utils

import com.clientledger.core.utils.CommonUtils.MONTHS_EDITABLE
import com.clientledger.core.utils.CommonUtils.MONTHS_REPORTABLE
import com.clientledger.core.utils.CommonUtils.MONTHS_TO_CHECK_IN_LEDGER_ENGINE
import com.clientledger.core.utils.CommonUtils.isInsideModificationMonthRange
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.Test
import java.time.LocalDate

/**
 * The editable window is relative to YearMonth.now(), so every case here is
 * expressed as an offset from today rather than a fixed date.
 */
class CommonUtilsTest {

    private fun monthsAgo(months: Long): LocalDate =
        LocalDate.now().minusMonths(months)

    // -------------------------------------------------------------------------
    // Inside the window
    // -------------------------------------------------------------------------

    @Test
    fun currentMonthIsEditable() {
        assertTrue(isInsideModificationMonthRange(LocalDate.now()))
    }

    @Test
    fun recentMonthsAreEditable() {
        for (months in 1L..7L) {
            assertTrue(
                isInsideModificationMonthRange(monthsAgo(months)),
                "$months months ago should be editable"
            )
        }
    }

    @Test
    fun oldestEditableMonthIsIncluded() {
        // The boundary is inclusive: !isBefore(oldestEditableMonth).
        assertTrue(isInsideModificationMonthRange(monthsAgo(MONTHS_EDITABLE)))
    }

    // -------------------------------------------------------------------------
    // Outside the window
    // -------------------------------------------------------------------------

    @Test
    fun monthJustBeyondWindowIsRejected() {
        assertFalse(isInsideModificationMonthRange(monthsAgo(MONTHS_EDITABLE + 1)))
    }

    @Test
    fun oneYearAgoIsRejected() {
        assertFalse(isInsideModificationMonthRange(monthsAgo(12)))
    }

    @Test
    fun formerThirtySixMonthWindowIsNowRejected() {
        // Regression guard for the Phase 1 split: these months used to be
        // editable when the gate read MONTHS_TO_CHECK_IN_LEDGER_ENGINE.
        for (months in (MONTHS_EDITABLE + 1)..MONTHS_TO_CHECK_IN_LEDGER_ENGINE) {
            assertFalse(
                isInsideModificationMonthRange(monthsAgo(months)),
                "$months months ago should no longer be editable"
            )
        }
    }

    @Test
    fun futureMonthIsRejected() {
        assertFalse(isInsideModificationMonthRange(LocalDate.now().plusMonths(1)))
    }

    // -------------------------------------------------------------------------
    // The three windows are distinct
    // -------------------------------------------------------------------------

    @Test
    fun windowsHaveTheirIntendedValues() {
        assertEquals(8L, MONTHS_EDITABLE)
        assertEquals(60L, MONTHS_REPORTABLE)
        assertEquals(36L, MONTHS_TO_CHECK_IN_LEDGER_ENGINE)
    }

    @Test
    fun repositoryScanWindowStaysWiderThanEditWindow() {
        /*
         * Recovering the month at the start of the edit window needs the older
         * snapshots that seed its opening balance, and a dormant client's last
         * snapshot can be years old. If someone "tidies" the scan bound down to
         * the edit window, the recovery path rebuilds those months as zero.
         */
        assertTrue(
            MONTHS_TO_CHECK_IN_LEDGER_ENGINE > MONTHS_EDITABLE,
            "Repository scan window must stay wider than the edit window"
        )
        assertTrue(
            MONTHS_REPORTABLE >= MONTHS_TO_CHECK_IN_LEDGER_ENGINE,
            "Reports must be able to look back at least as far as the ledger scans"
        )
    }
}
package com.clientledger.core.service.order

import com.clientledger.core.domain.order.BalanceRelevantItem
import com.clientledger.core.domain.order.BalanceRelevantOrder
import com.clientledger.core.domain.order.Order
import com.clientledger.core.exception.ResourceNotFoundException
import com.clientledger.core.exception.ValidationException
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.order.OrderRepository
import com.clientledger.core.transaction.LedgerTransactionCoordinator
import com.clientledger.core.utils.CommonUtils.isInsideModificationMonthRange
import com.clientledger.core.utils.IdGenerator
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.stereotype.Service

@Service
class OrderService(
    @Qualifier("firestoreOrderRepository")
    private val orderRepository: OrderRepository,
    @Qualifier("firestoreClientRepository")
    private val clientRepository: ClientRepository,
    private val ledgerTransactionCoordinator: LedgerTransactionCoordinator
) {

    fun saveOrder(saveOrder: Order): Order {

        // ---------------------------------------------------------
        // First gate. Reject an out-of-window date before an id is
        // generated, before any read, and before the ledger is touched.
        // ---------------------------------------------------------

        if (!isInsideModificationMonthRange(saveOrder.orderDate)) {
            throw ValidationException("Order is outside the editable period ")
        }

        val order = if (saveOrder.id.isBlank()) {
            saveOrder.copy(id = IdGenerator.generateOrderId())
        } else {
            saveOrder
        }

        val existingOrder = orderRepository.findById(order.id)

        // ---------------------------------------------------------
        // Prevent direct modification of an invoiced order
        // ---------------------------------------------------------

        if (existingOrder?.invoiceId != null) {
            throw IllegalStateException("Order ${existingOrder.id} is already linked to Invoice " + "${existingOrder.invoiceId}. " + "Please modify the Order through the Invoice.")
        }

        // ---------------------------------------------------------
        // Also prevent manually assigning an invoiceId through
        // Order API/UI.
        // InvoiceService will be responsible for linking orders.
        // ---------------------------------------------------------

        if (existingOrder == null && order.invoiceId != null
        ) {
            throw IllegalStateException("A new Order cannot be directly linked to an Invoice. " + "Please create the Invoice first.")
        }

        if (existingOrder != null && order.invoiceId != null) {
            throw IllegalStateException("Order cannot be directly linked to an Invoice. " + "InvoiceService must handle the Invoice relationship.")
        }

        // ---------------------------------------------------------
        // New order
        // ---------------------------------------------------------

        if (existingOrder == null) {

            if (clientRepository.findById(order.clientId) == null) {
                throw ResourceNotFoundException("Client ${order.clientId} does not exist. " + "Cannot create Order.")
            }
            ledgerTransactionCoordinator.saveOrderAndRecalculateLedger(order = order, transactionDate = order.orderDate)
            return order
        }

        // ---------------------------------------------------------
        // Client changed
        // ---------------------------------------------------------

        if (existingOrder.clientId != order.clientId) {
            ledgerTransactionCoordinator.deleteOrderAndRecalculateLedger(orderId = existingOrder.id)
            ledgerTransactionCoordinator.saveOrderAndRecalculateLedger(
                order = order,
                transactionDate = order.orderDate
            )

            return order
        }

        // ---------------------------------------------------------
        // Check balance-related fields
        // ---------------------------------------------------------

        val balanceUnaffected =
            isBalanceUnaffectedEdit(
                old = existingOrder,
                new = order
            )

        if (balanceUnaffected) {
            // -----------------------------------------------------
            // Only expense / deliveryDate / name / quantity / unit
            // etc. changed.
            // -----------------------------------------------------

            orderRepository.save(order)

            ledgerTransactionCoordinator.updateCurrentMonthOnly(
                order.clientId,
                order.orderDate
            )
            return order
        }

        // ---------------------------------------------------------
        // Balance-affecting change
        // ---------------------------------------------------------
        val earliestDate = minOf(existingOrder.orderDate, order.orderDate)

        ledgerTransactionCoordinator.saveOrderAndRecalculateLedger(
            order = order,
            transactionDate = earliestDate
        )
        return order
    }


    private fun isBalanceUnaffectedEdit(old: Order, new: Order): Boolean {

        val oldBalanceData = BalanceRelevantOrder(
            clientId = old.clientId,
            orderDate = old.orderDate,
            totalInvoiceAmount = old.totalInvoiceAmount,
            totalGstAmount = old.totalGstAmount,
            items = old.items.map {
                BalanceRelevantItem(
                    baseAmount = it.baseAmount,
                    gstRate = it.gstRate,
                    gstAmount = it.gstAmount,
                    lineTotal = it.lineTotal
                )
            }
        )

        val newBalanceData = BalanceRelevantOrder(
            clientId = new.clientId,
            orderDate = new.orderDate,
            totalInvoiceAmount = new.totalInvoiceAmount,
            totalGstAmount = new.totalGstAmount,
            items = new.items.map {
                BalanceRelevantItem(
                    baseAmount = it.baseAmount,
                    gstRate = it.gstRate,
                    gstAmount = it.gstAmount,
                    lineTotal = it.lineTotal
                )
            }
        )

        return oldBalanceData == newBalanceData
    }

    fun deleteOrder(orderId: String) {

        val order = orderRepository.findById(orderId) ?: return

        if (!isInsideModificationMonthRange(order.orderDate)) {
            throw ValidationException("Order is outside the editable period ")
        }

        // ---------------------------------------------------------
        // Prevent direct deletion of an invoiced order
        // ---------------------------------------------------------
        if (order.invoiceId != null) {
            throw IllegalStateException("Order ${order.id} is already linked to Invoice " + "${order.invoiceId}. " + "Please delete or modify the Order through the Invoice.")
        }
        // ---------------------------------------------------------
        // Delete the order
        // ---------------------------------------------------------
        ledgerTransactionCoordinator.deleteOrderAndRecalculateLedger(orderId = orderId)
    }

    fun getOrderById(orderId: String): Order? {
        return orderRepository.findById(orderId)
    }
}package com.clientledger.core.service.payment

import com.clientledger.core.domain.order.BalanceRelevantPayment
import com.clientledger.core.domain.payment.Payment
import com.clientledger.core.exception.ValidationException
import com.clientledger.core.repository.payment.PaymentRepository
import com.clientledger.core.transaction.LedgerTransactionCoordinator
import com.clientledger.core.utils.CommonUtils.isInsideModificationMonthRange
import com.clientledger.core.utils.IdGenerator
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.stereotype.Service

@Service
class PaymentService(
    @Qualifier("firestorePaymentRepository")
    private val paymentRepository: PaymentRepository,
    private val ledgerTransactionCoordinator: LedgerTransactionCoordinator
) {

    fun savePayment(savePayment: Payment): Payment {

        // ---------------------------------------------------------
        // First gate. Reject an out-of-window date before an id is
        // generated, before any read, and before the ledger is touched.
        // ---------------------------------------------------------

        if (!isInsideModificationMonthRange(savePayment.paymentDate)) {
            throw ValidationException("Payment is outside the editable period ")
        }

        val payment = if (savePayment.id.isBlank()) {
            savePayment.copy(id = IdGenerator.generatePaymentId())
        } else {
            savePayment
        }

        val existingPayment = paymentRepository.findById(payment.id)

        // ---------------------------------------------------------
        // New payment
        // ---------------------------------------------------------
        if (existingPayment == null) {

            ledgerTransactionCoordinator.savePaymentAndRecalculateLedger(
                payment = payment,
                transactionDate = payment.paymentDate
            )
            return payment
        }

        // ---------------------------------------------------------
        // Client changed
        // ---------------------------------------------------------
        if (existingPayment.clientId != payment.clientId) {
            // Old client loses this payment
            ledgerTransactionCoordinator.deletePaymentAndRecalculateLedger(paymentId = existingPayment.id)
            // New client receives this payment
            ledgerTransactionCoordinator.savePaymentAndRecalculateLedger(
                payment = payment,
                transactionDate = payment.paymentDate
            )
            return payment
        }

        // ---------------------------------------------------------
        // Balance unaffected
        // ---------------------------------------------------------
        if (isPaymentBalanceUnaffectedEdit(old = existingPayment, new = payment)) {
            // Payment mode / notes / etc.
            // Balance remains exactly the same.
            //
            // No ledger recalculation required.
            paymentRepository.save(payment)
            return payment
        }

        // ---------------------------------------------------------
        // Balance-affecting change
        // ---------------------------------------------------------
        val earliestDate = minOf(existingPayment.paymentDate, payment.paymentDate)
        ledgerTransactionCoordinator.savePaymentAndRecalculateLedger(
            payment = payment,
            transactionDate = earliestDate
        )
        return payment
    }

    private fun isPaymentBalanceUnaffectedEdit(old: Payment, new: Payment): Boolean {
        val oldBalanceData =
            BalanceRelevantPayment(
                clientId = old.clientId,
                paymentDate = old.paymentDate,
                amount = old.amount
            )

        val newBalanceData =
            BalanceRelevantPayment(
                clientId = new.clientId,
                paymentDate = new.paymentDate,
                amount = new.amount
            )
        return oldBalanceData == newBalanceData
    }

    fun deletePayment(paymentId: String) {
        val payment =
            paymentRepository.findById(paymentId) ?: throw ValidationException("Payment not found for deletion")

        if (!isInsideModificationMonthRange(payment.paymentDate)) {
            throw ValidationException("Payment is outside the editable period ")
        }

        ledgerTransactionCoordinator.deletePaymentAndRecalculateLedger(paymentId = paymentId)
    }

    fun getPaymentById(paymentId: String): Payment? {
        return paymentRepository.findById(paymentId)
    }

}
package com.clientledger.core.service.invoice

import com.clientledger.core.domain.invoice.Invoice
import com.clientledger.core.domain.order.Order
import com.clientledger.core.domain.owner.Owner
import com.clientledger.core.exception.ValidationException
import com.clientledger.core.repository.invoice.InvoiceRepository
import com.clientledger.core.repository.order.OrderRepository
import com.clientledger.core.service.pdf.PdfGenerationService
import com.clientledger.core.transaction.LedgerTransactionCoordinator
import com.clientledger.core.utils.CommonUtils.isInsideModificationMonthRange
import com.clientledger.core.utils.IdGenerator
import org.slf4j.LoggerFactory
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.stereotype.Service
import java.time.YearMonth

@Service
class InvoiceService(
    @Qualifier("firestoreInvoiceRepository")
    private val invoiceRepository: InvoiceRepository,
    @Qualifier("firestoreOrderRepository")
    private val orderRepository: OrderRepository,
    private val ledgerTransactionCoordinator: LedgerTransactionCoordinator,
    private val pdfGenerationService: PdfGenerationService
) {
    private val log = LoggerFactory.getLogger(InvoiceService::class.java);

    fun createInvoice(createInvoice: Invoice, primaryColor: String? = null, secondaryColor: String? = null): ByteArray {
        val invoice = if (createInvoice.id.isBlank()) {
            createInvoice.copy(id = IdGenerator.generateInvoiceId())
        } else {
            createInvoice
        }
        val orders = getOrders(invoice.orderIds)

        validateInvoice(invoice = invoice, orders = orders)
        // ---------------------------------------------------------
        // Save Invoice + link Orders + recalculate Ledger
        // ---------------------------------------------------------
        ledgerTransactionCoordinator.saveInvoiceAndRecalculateLedger(invoice = invoice)
        // ---------------------------------------------------------
        // Generate PDF and store PDF JSON
        // ---------------------------------------------------------
        val owner = createTemporaryOwner()

        return generatePdf(
            invoice = invoice,
            owner = owner,
            primaryColor = primaryColor,
            secondaryColor = secondaryColor
        )
    }


// =========================================================
// UPDATE INVOICE
// =========================================================

    fun updateInvoice(invoice: Invoice, primaryColor: String? = null, secondaryColor: String? = null): ByteArray {

        val existingInvoice = invoiceRepository.findById(invoice.id)
            ?: throw ValidationException("Invoice ${invoice.id} not found.")

        val existingOrders = getOrders(existingInvoice.orderIds)

        val newOrders = getOrders(invoice.orderIds)

        //-------Validation---------------//
        validateInvoice(
            invoice = invoice,
            orders = newOrders,
            existingInvoice = existingInvoice,
            existingOrders = existingOrders
        )
        // ---------------------------------------------------------
        // Save Invoice + update Orders + recalculate Ledger
        // ---------------------------------------------------------
        ledgerTransactionCoordinator.updateInvoiceAndRecalculateLedger(
            invoice = invoice,
            existingInvoice = existingInvoice
        )
        // ---------------------------------------------------------
        // Generate updated PDF and replace PDF JSON
        // ---------------------------------------------------------

        val owner = createTemporaryOwner()

        return generatePdf(
            invoice = invoice,
            owner = owner,
            primaryColor = primaryColor,
            secondaryColor = secondaryColor
        )
    }


    // =========================================================
    // DELETE INVOICE
    // =========================================================

    fun deleteInvoice(invoiceId: String) {

        val invoice =
            invoiceRepository.findById(invoiceId) ?: throw ValidationException("Invoice $invoiceId not found.")

        val orders = getOrders(invoice.orderIds)

        // ---------------------------------------------------------
        // Historical Invoice cannot be deleted
        // ---------------------------------------------------------
        val orderDate = orders.firstOrNull()?.orderDate
        if (orderDate != null && !isInsideModificationMonthRange(orderDate)) {
            log.warn(
                "Invoice delete rejected. invoiceId={}, clientId={}, reason=outside editable period",
                invoiceId,
                invoice.clientId
            )
            throw ValidationException(
                "Invoice $invoiceId is outside the editable period and cannot be deleted."
            )
        }
        // ---------------------------------------------------------
        // Delete Invoice + unlock Orders + recalculate Ledger
        // ---------------------------------------------------------
        ledgerTransactionCoordinator.deleteInvoiceAndRecalculateLedger(invoiceId = invoiceId)
        // ---------------------------------------------------------
        // Delete generated PDF JSON
        // ---------------------------------------------------------
        pdfGenerationService.deletePdfJson(invoiceId)
    }


// =========================================================
// GET INVOICE
// =========================================================

    fun getInvoiceById(invoiceId: String): Invoice? {
        return invoiceRepository.findById(invoiceId)
    }

// =========================================================
// VALIDATION
// =========================================================

    /**
     * Single validation entry point for both create and update.
     *
     * Pass [existingInvoice] and [existingOrders] only on the update path.
     * When they are null/empty the create rules apply.
     *
     * Throws:
     *  - IllegalArgumentException  -> invalid client input        (map to 400)
     *  - IllegalStateException     -> conflicts with current state (map to 409)
     *  - ValidationException       -> business rule violation
     */
    private fun validateInvoice(
        invoice: Invoice,
        orders: List<Order>,
        existingInvoice: Invoice? = null,
        existingOrders: List<Order> = emptyList()
    ) {
        try {
            // Editable period is checked first: an out-of-window date is
            // rejected before anything else is considered or created.
            validateEditablePeriod(invoice, orders, existingInvoice, existingOrders)
            validateInvoiceFields(invoice)
            validateClientUnchanged(invoice, existingInvoice)
            validateOrders(invoice, orders, existingInvoice?.id)
            validateOrdersSameMonth(orders)
        } catch (ex: RuntimeException) {
            log.warn(
                "Invoice validation failed. invoiceId={}, clientId={}, orderIds={}, reason={}",
                invoice.id,
                invoice.clientId,
                invoice.orderIds,
                ex.message
            )
            throw ex
        }
    }

    /**
     * Field-level rules. Pure — needs nothing from the database.
     */
    private fun validateInvoiceFields(invoice: Invoice) {
        require(invoice.id.isNotBlank()) { "Invoice ID cannot be empty" }
        require(invoice.clientId.isNotBlank()) { "Client ID cannot be empty" }
        require(invoice.orderIds.isNotEmpty()) { "At least one Order is required to create an Invoice" }
        require(invoice.orderIds.size <= 1) { "Kindly Select one order to generate invoice" }
        require(invoice.orderIds.distinct().size == invoice.orderIds.size) { "Duplicate Orders are not allowed in an Invoice" }
        require(invoice.totalInvoiceAmount >= 0) { "Invoice amount cannot be negative" }
        require(invoice.discount >= 0) { "Discount cannot be negative" }
        require(invoice.discount <= invoice.totalInvoiceAmount) { "Discount cannot be greater than Invoice amount" }
        require(invoice.finalAmount == invoice.totalInvoiceAmount - invoice.discount) { "Final amount must equal totalInvoiceAmount - discount" }
    }

    /**
     * Client cannot be reassigned on an existing Invoice.
     */
    private fun validateClientUnchanged(invoice: Invoice, existingInvoice: Invoice?) {
        if (existingInvoice == null) return
        require(invoice.clientId == existingInvoice.clientId) {
            "Invoice client cannot be changed."
        }
    }

    /**
     * Orders must belong to the Invoice client, must not already be linked to a
     * different Invoice, and must add up to the Invoice total.
     */
    private fun validateOrders(
        invoice: Invoice, orders: List<Order>, currentInvoiceId: String?
    ) {
        orders.forEach { order ->
            require(order.clientId == invoice.clientId) {
                "Order ${order.id} does not belong to Client ${invoice.clientId}."
            }

            val linkedInvoiceId = order.invoiceId
            check(linkedInvoiceId == null || linkedInvoiceId == currentInvoiceId) {
                "Order ${order.id} is already linked to Invoice $linkedInvoiceId."
            }
        }

        val ordersTotal = orders.sumOf { it.totalInvoiceAmount }
        require(ordersTotal == invoice.totalInvoiceAmount) {
            "Invoice amount ${invoice.totalInvoiceAmount} does not match order total $ordersTotal."
        }
    }

    /**
     * All Orders in a single Invoice must belong to the same month.
     */
    private fun validateOrdersSameMonth(orders: List<Order>) {
        val months = orders.map { YearMonth.from(it.orderDate) }.toSet()
        require(months.size == 1) {
            "All Orders in a single Invoice must belong to the same month."
        }
    }

    /**
     * Editable-period rules.
     *
     * An Invoice only reaches the ledger through its discount: the Order
     * contributes totalInvoiceAmountWithGst, the Invoice contributes
     * totalDiscount. So the editable period gates the discount, not the
     * Invoice itself.
     *
     * Create: an Invoice may be generated for an Order outside the editable
     *         period, but no discount is allowed.
     * Update: an Invoice outside the editable period may still be updated as
     *         long as the discount stays put. Changing the discount amount, or
     *         moving a non-zero discount to an Order in another month, moves a
     *         balance and is only allowed inside the period.
     */
    private fun validateEditablePeriod(
        invoice: Invoice,
        orders: List<Order>,
        existingInvoice: Invoice?,
        existingOrders: List<Order>
    ) {
        // This runs before validateInvoiceFields, so it cannot assume an Order
        // is present.
        require(orders.isNotEmpty()) {
            "At least one Order is required to generate an Invoice"
        }

        val newOrderEditable = isInsideModificationMonthRange(orders.first().orderDate)
        if (existingInvoice == null) {
            if (!newOrderEditable) {
                require(invoice.discount == 0L) {
                    "Discount is not allowed for an Invoice containing Orders " +
                            "outside the editable period."
                }
            }
            return
        }

        val existingOrderDate = existingOrders.firstOrNull()?.orderDate

        val existingOrderEditable =
            existingOrderDate?.let { isInsideModificationMonthRange(it) } ?: true

        val discountChanged = invoice.discount != existingInvoice.discount

        val discountMonthMoved =
            invoice.discount != 0L &&
                    existingOrderDate?.let { YearMonth.from(it) } !=
                    YearMonth.from(orders.first().orderDate)

        // Balance-neutral update: leave it alone regardless of age.
        if (!discountChanged && !discountMonthMoved) {
            return
        }

        if (!existingOrderEditable) {
            throw ValidationException(
                "Invoice ${invoice.id} has a discount on Orders outside the " +
                        "editable period, so the discount cannot be changed."
            )
        }

        if (!newOrderEditable) {
            throw ValidationException(
                "Invoice discount cannot be applied because one or more selected " +
                        "Orders are outside the editable period."
            )
        }
    }


// =========================================================
// PRIVATE HELPERS
// =========================================================

    private fun getOrders(orderIds: List<String>): List<Order> {
        return orderIds.map { orderId ->
            orderRepository.findById(orderId)
                ?: throw ValidationException("Order $orderId not found.")
        }
    }

    private fun generatePdf(invoice: Invoice, owner: Owner, primaryColor: String?, secondaryColor: String?): ByteArray {
        return pdfGenerationService.generateAndSavePdf(
            invoice = invoice,
            owner = owner,
            primaryColor = primaryColor,
            secondaryColor = secondaryColor
        )
    }

    /**
     * Temporary Owner information.
     *
     * This will later be replaced with the
     * logged-in owner's actual business information.
     */
    private fun createTemporaryOwner(): Owner {
        return Owner(
            name = "Owner Name",
            organization = "My Organization",
            email = "owner@example.com",
            phone = "0000000000",
            gstNumber = "GSTIN",
            address = "Business Address"
        )
    }
}


package com.clientledger.core.domain.summary

import java.time.Instant

/**
 * Company-wide totals for one month: global_summaries/{yyyy-MM}.
 *
 * The fields fall into two groups that behave oppositely at a month boundary,
 * and keeping them straight is the whole point of this class:
 *
 * FLOW fields are activity that happened *in* the month. A fresh month starts
 * at zero and they accumulate as transactions land.
 *
 * STOCK fields are point-in-time balances. A fresh month starts at the previous
 * month's value, NOT at zero, because a client with no activity this month still
 * owes exactly what they owed last month. Rebuilding stock from only the
 * current month's snapshots is what used to make inactive clients disappear
 * from the receivable total.
 */
data class GlobalSummary(
    val yearMonth: String,                  // e.g., "2025-01" or "2026 (YTD)"

    // -------------------------------------------------------------------------
    // FLOW - this month's activity. Zero in a freshly seeded month.
    // -------------------------------------------------------------------------

    val totalOrdersCount: Int = 0,            // Us mahine ke total orders ka count
    val totalInvoiceAmount: Long = 0L,        // Without GST (Total Orders - GST)
    val totalGstAmount: Long = 0L,            // Total GST collected
    val totalInvoiceAmountWithGst: Long = 0L, // Total Orders amount with GST
    val totalExpenses: Long = 0L,             // Total expenses in that month
    val totalPayments: Long = 0L,             // Total payments received
    val totalDiscount: Long = 0L,             // Total discount given
    val netProfit: Long = 0L,                 // Invoice - Expenses - Discount
    val cashFlow: Long = 0L,                  // Payments - Expenses

    // -------------------------------------------------------------------------
    // STOCK - point-in-time balances. Carried forward into a new month.
    //
    // A client's actual balance is initialOpeningBalance + closingBalance, and
    // the sign of that decides which bucket they land in. Each headline splits
    // into the two components it was built from, so a drill-down can show how
    // much of the total came from opening balances and how much from ledger
    // activity:
    //
    //   totalReceivableAmount = transactionReceivableAmount + initialReceivableComponent
    //   totalAdvanceAmount    = transactionAdvanceAmount    + initialAdvanceComponent
    //
    // The advance components are stored sign-flipped (as positive magnitudes)
    // so that identity holds. An individual component may still be negative -
    // a client with a positive opening balance and a deeply negative ledger
    // balance is in advance overall - which is correct, not a bug.
    // -------------------------------------------------------------------------

    val totalReceivableAmount: Long = 0L,     // Actual receivable across ALL clients
    val totalAdvanceAmount: Long = 0L,        // Actual advance across ALL clients

    val transactionReceivableAmount: Long = 0L,
    val initialReceivableComponent: Long = 0L,
    val transactionAdvanceAmount: Long = 0L,
    val initialAdvanceComponent: Long = 0L,

    val receivableClientCount: Int = 0,
    val advanceClientCount: Int = 0,
    val settledClientCount: Int = 0,
    val totalClientCount: Int = 0,

    val updatedAt: Instant = Instant.now()
)

package com.clientledger.core.service.summary

import com.clientledger.core.domain.snapshot.ClientBalanceSnapshot
import com.clientledger.core.domain.summary.*
import com.clientledger.core.repository.client.ClientRepository
import com.clientledger.core.repository.order.FirestoreOrderRepository
import com.clientledger.core.repository.payment.FirestorePaymentRepository
import com.clientledger.core.repository.snapshot.SnapshotRepository
import com.clientledger.core.utils.CommonUtils.MONTHS_REPORTABLE
import com.google.cloud.firestore.Firestore
import com.google.cloud.firestore.Transaction
import org.springframework.stereotype.Service
import java.time.Instant
import java.time.YearMonth
import java.util.Date
import kotlin.math.abs

@Service
class GlobalSummaryService(
    private val clientRepository: ClientRepository,
    private val snapshotRepository: SnapshotRepository,
    private val db: Firestore,
    private val orderRepository: FirestoreOrderRepository,
    private val paymentRepository: FirestorePaymentRepository
) {

    /**
     * Point-in-time balance buckets for a set of clients.
     *
     * A client's actual balance is initialOpeningBalance + closingBalance and
     * the sign of that decides the bucket. Each total also carries the two
     * components it was built from, so these identities hold by construction:
     *
     *   totalReceivable = transactionReceivable + initialReceivable
     *   totalAdvance    = transactionAdvance    + initialAdvance
     *
     * Advance components are accumulated sign-flipped so they stay positive
     * magnitudes like the total they belong to. An individual component can
     * still be negative - a client with a positive opening balance and a deeply
     * negative ledger balance is in advance overall - which is correct.
     */
    private class BalanceStock {
        var totalReceivable = 0L
        var transactionReceivable = 0L
        var initialReceivable = 0L
        var totalAdvance = 0L
        var transactionAdvance = 0L
        var initialAdvance = 0L
        var receivableClients = 0
        var advanceClients = 0
        var settledClients = 0

        fun add(initialOpeningBalance: Long, closingBalance: Long) {

            val actualBalance = initialOpeningBalance + closingBalance

            when {
                actualBalance > 0L -> {
                    totalReceivable += actualBalance
                    transactionReceivable += closingBalance
                    initialReceivable += initialOpeningBalance
                    receivableClients++
                }

                actualBalance < 0L -> {
                    totalAdvance += -actualBalance
                    transactionAdvance += -closingBalance
                    initialAdvance += -initialOpeningBalance
                    advanceClients++
                }

                else -> settledClients++
            }
        }
    }

    /**
     * Recalculate and store the global summary for a month from scratch.
     *
     * Source of truth: Client Balance Snapshots.
     *
     * This is the authoritative-but-expensive path. It walks every client, so a
     * client with no activity in the month still contributes their carried
     * forward balance, and a client with no snapshots at all still contributes
     * their initialOpeningBalance. That is what makes inactive clients show up
     * in the receivable and advance totals.
     *
     * Use it for migration and repair, not on the read path.
     */
    fun recomputeGlobalSummaryForMonth(
        yearMonth: YearMonth
    ): GlobalSummary {

        val clients =
            clientRepository.findAll()

        var totalOrdersCount = 0L
        var totalInvoiceAmount = 0L
        var totalGstAmount = 0L
        var totalInvoiceAmountWithGst = 0L
        var totalExpenses = 0L
        var totalPayments = 0L
        var totalDiscount = 0L

        val stock = BalanceStock()

        clients.forEach { client ->

            val snapshot =
                getEffectiveSnapshotForMonth(
                    clientId = client.id,
                    yearMonth = yearMonth
                )

            totalOrdersCount +=
                snapshot.totalOrdersCount.toLong()

            totalInvoiceAmount +=
                snapshot.totalInvoiceAmount

            totalGstAmount +=
                snapshot.totalGstAmounts

            totalInvoiceAmountWithGst +=
                snapshot.totalInvoiceAmountWithGst

            totalExpenses += snapshot.totalExpenses

            totalPayments += snapshot.totalPayments

            totalDiscount += snapshot.totalDiscount

            stock.add(
                initialOpeningBalance = client.initialOpeningBalance,
                closingBalance = snapshot.closingBalance
            )
        }

        val globalSummary =
            GlobalSummary(
                yearMonth = yearMonth.toString(),
                totalOrdersCount = totalOrdersCount.toInt(),
                totalInvoiceAmount = totalInvoiceAmount,
                totalGstAmount = totalGstAmount,
                totalInvoiceAmountWithGst = totalInvoiceAmountWithGst,
                totalExpenses = totalExpenses,
                totalPayments = totalPayments,
                totalDiscount = totalDiscount,

                netProfit = totalInvoiceAmount - totalExpenses - totalDiscount,
                cashFlow = totalPayments - totalExpenses,

                totalReceivableAmount = stock.totalReceivable,
                totalAdvanceAmount = stock.totalAdvance,
                transactionReceivableAmount = stock.transactionReceivable,
                initialReceivableComponent = stock.initialReceivable,
                transactionAdvanceAmount = stock.transactionAdvance,
                initialAdvanceComponent = stock.initialAdvance,
                receivableClientCount = stock.receivableClients,
                advanceClientCount = stock.advanceClients,
                settledClientCount = stock.settledClients,
                totalClientCount = clients.size,

                updatedAt = Instant.now()
            )

        val docRef =
            db.collection("global_summaries")
                .document(yearMonth.toString())

        /*
         * Written through the same explicit map as every other write path.
         * Handing the POJO straight to transaction.set stored updatedAt as a
         * raw Instant while saveGlobalSummary stored a Firestore timestamp, so
         * the field's shape depended on whichever path wrote last.
         */
        db.runTransaction { transaction ->
            transaction.set(
                docRef,
                globalSummaryToMap(globalSummary)
            )
            null
        }.get()

        return globalSummary
    }

    /**
     * Recompute a span of months, oldest first.
     *
     * Migration and repair entry point. The stock fields only chain correctly
     * when earlier months are already correct, so the order matters.
     */
    fun recomputeGlobalSummaryForRange(
        startMonth: YearMonth,
        endMonth: YearMonth
    ): List<GlobalSummary> {

        require(!startMonth.isAfter(endMonth)) {
            "startMonth $startMonth cannot be after endMonth $endMonth"
        }

        val results = mutableListOf<GlobalSummary>()

        var cursor = startMonth
        while (!cursor.isAfter(endMonth)) {
            results.add(recomputeGlobalSummaryForMonth(cursor))
            cursor = cursor.plusMonths(1)
        }

        return results
    }

    /**
     * Create a month's summary by carrying the previous month's balances
     * forward.
     *
     * Stock fields are copied across: a client with no activity this month
     * still owes exactly what they owed last month, so receivable and advance
     * start where they left off instead of at zero. Flow fields start at zero
     * because no activity has landed in this month yet.
     *
     * This costs one read rather than one per client, which is the whole reason
     * the totals can include inactive clients cheaply. It walks back through
     * gaps because a month in which nobody transacted leaves no document at all.
     *
     * Must be called during a transaction's read phase, before any write.
     */
    fun seedGlobalSummaryForMonth(
        transaction: Transaction,
        yearMonth: YearMonth
    ): GlobalSummary {

        val oldestMonth = yearMonth.minusMonths(MONTHS_REPORTABLE)
        var cursor = yearMonth.minusMonths(1)

        while (!cursor.isBefore(oldestMonth)) {

            val docRef =
                db.collection("global_summaries")
                    .document(cursor.toString())

            val document = transaction.get(docRef).get()

            if (document.exists()) {

                val data = document.data ?: emptyMap()

                fun carriedLong(key: String): Long =
                    (data[key] as? Number)?.toLong() ?: 0L

                fun carriedInt(key: String): Int =
                    (data[key] as? Number)?.toInt() ?: 0

                return GlobalSummary(
                    yearMonth = yearMonth.toString(),

                    // Flow deliberately left at its zero default.

                    totalReceivableAmount =
                        carriedLong("totalReceivableAmount"),
                    totalAdvanceAmount =
                        carriedLong("totalAdvanceAmount"),
                    transactionReceivableAmount =
                        carriedLong("transactionReceivableAmount"),
                    initialReceivableComponent =
                        carriedLong("initialReceivableComponent"),
                    transactionAdvanceAmount =
                        carriedLong("transactionAdvanceAmount"),
                    initialAdvanceComponent =
                        carriedLong("initialAdvanceComponent"),
                    receivableClientCount =
                        carriedInt("receivableClientCount"),
                    advanceClientCount =
                        carriedInt("advanceClientCount"),
                    settledClientCount =
                        carriedInt("settledClientCount"),
                    totalClientCount =
                        carriedInt("totalClientCount"),

                    updatedAt = Instant.now()
                )
            }

            cursor = cursor.minusMonths(1)
        }

        // Nothing inside the reportable window to carry forward.
        return GlobalSummary(yearMonth = yearMonth.toString())
    }

    fun globalSummaryToMap(
        summary: GlobalSummary
    ): Map<String, Any?> {

        return mapOf(
            "yearMonth" to summary.yearMonth,

            "totalOrdersCount" to summary.totalOrdersCount,
            "totalInvoiceAmount" to summary.totalInvoiceAmount,
            "totalGstAmount" to summary.totalGstAmount,
            "totalInvoiceAmountWithGst" to summary.totalInvoiceAmountWithGst,
            "totalExpenses" to summary.totalExpenses,
            "totalPayments" to summary.totalPayments,
            "totalDiscount" to summary.totalDiscount,
            "netProfit" to summary.netProfit,
            "cashFlow" to summary.cashFlow,

            "totalReceivableAmount" to summary.totalReceivableAmount,
            "totalAdvanceAmount" to summary.totalAdvanceAmount,
            "transactionReceivableAmount" to summary.transactionReceivableAmount,
            "initialReceivableComponent" to summary.initialReceivableComponent,
            "transactionAdvanceAmount" to summary.transactionAdvanceAmount,
            "initialAdvanceComponent" to summary.initialAdvanceComponent,
            "receivableClientCount" to summary.receivableClientCount,
            "advanceClientCount" to summary.advanceClientCount,
            "settledClientCount" to summary.settledClientCount,
            "totalClientCount" to summary.totalClientCount,

            "updatedAt" to Date.from(summary.updatedAt)
        )
    }

    /**
     * Returns the effective snapshot for a particular client/month.
     *
     * If an exact snapshot exists, it is returned.
     *
     * If the exact month does not exist, the latest previous
     * snapshot is used and its closing balance is carried forward.
     *
     * IMPORTANT:
     * initialOpeningBalance is NOT added here.
     */
    private fun getEffectiveSnapshotForMonth(
        clientId: String,
        yearMonth: YearMonth
    ): ClientBalanceSnapshot {

        /*
         * 1. Exact month snapshot.
         */
        val exactSnapshot =
            snapshotRepository.getSnapshot(
                clientId = clientId,
                yearMonth = yearMonth
            )

        if (exactSnapshot != null) {
            return exactSnapshot
        }

        /*
         * 2. Find latest snapshot before requested month.
         */
        val previousSnapshot =
            snapshotRepository.findLatestSnapshotBefore(
                clientId = clientId,
                yearMonth = yearMonth
            )

        val balance =
            previousSnapshot?.closingBalance ?: 0L

        /*
         * 3. Create a virtual snapshot.
         *
         * This is only used for calculation.
         * It is NOT written to Firestore.
         */
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

    /**
     * Returns the stored global summary for a month.
     *
     * IMPORTANT:
     * This method reads ONLY from global_summaries.
     *
     * ClientMonthlyReporting is NOT used here.
     */
    fun getGlobalSummaryForMonth(
        yearMonth: YearMonth
    ): Map<String, Any?> {

        val docRef =
            db.collection("global_summaries")
                .document(yearMonth.toString())

        val snapshot =
            docRef.get().get()

        if (!snapshot.exists()) {
            return emptyMap()
        }

        return snapshot.data ?: emptyMap()
    }

    /**
     * Get complete yearly global summary.
     *
     * Monthly transaction metrics are summed.
     *
     * Receivable/Advance is taken only from the
     * final month of the requested year.
     */
    fun getGlobalSummaryForYear(
        year: Int
    ): GlobalSummary {

        val currentYear =
            YearMonth.now().year

        val currentMonthValue =
            YearMonth.now().monthValue

        val lastMonth =
            if (year == currentYear) {
                currentMonthValue
            } else {
                12
            }

        var totalOrdersCount = 0L
        var totalInvoiceAmount = 0L
        var totalGstAmount = 0L
        var totalInvoiceAmountWithGst = 0L
        var totalExpenses = 0L
        var totalPayments = 0L
        var totalDiscount = 0L
        var netProfit = 0L
        var cashFlow = 0L

        for (month in 1..lastMonth) {

            val yearMonth =
                YearMonth.of(year, month)

            val summary =
                getGlobalSummaryForMonth(yearMonth)

            totalOrdersCount +=
                (summary["totalOrdersCount"] as? Number)
                    ?.toLong() ?: 0L

            totalInvoiceAmount +=
                (summary["totalInvoiceAmount"] as? Number)
                    ?.toLong() ?: 0L

            totalGstAmount +=
                (summary["totalGstAmount"] as? Number)
                    ?.toLong() ?: 0L

            totalInvoiceAmountWithGst +=
                (summary["totalInvoiceAmountWithGst"] as? Number)
                    ?.toLong() ?: 0L

            totalExpenses +=
                (summary["totalExpenses"] as? Number)
                    ?.toLong() ?: 0L

            totalPayments +=
                (summary["totalPayments"] as? Number)
                    ?.toLong() ?: 0L

            totalDiscount +=
                (summary["totalDiscount"] as? Number)
                    ?.toLong() ?: 0L

            netProfit +=
                (summary["netProfit"] as? Number)
                    ?.toLong() ?: 0L

            cashFlow +=
                (summary["cashFlow"] as? Number)
                    ?.toLong() ?: 0L
        }

        /*
         * Receivable/Advance is a point-in-time balance.
         *
         * Therefore do NOT sum it across months.
         */
        val finalMonth =
            YearMonth.of(year, lastMonth)

        val finalSummary =
            getGlobalSummaryForMonth(finalMonth)

        val totalReceivableAmount =
            (finalSummary["totalReceivableAmount"] as? Number)
                ?.toLong() ?: 0L

        val totalAdvanceAmount =
            (finalSummary["totalAdvanceAmount"] as? Number)
                ?.toLong() ?: 0L

        return GlobalSummary(
            yearMonth = year.toString(),

            totalOrdersCount =
                totalOrdersCount.toInt(),

            totalInvoiceAmount =
                totalInvoiceAmount,

            totalGstAmount =
                totalGstAmount,

            totalInvoiceAmountWithGst =
                totalInvoiceAmountWithGst,

            totalExpenses =
                totalExpenses,

            totalPayments =
                totalPayments,

            totalReceivableAmount =
                totalReceivableAmount,

            totalAdvanceAmount =
                totalAdvanceAmount,

            totalDiscount =
                totalDiscount,

            netProfit =
                netProfit,

            cashFlow =
                cashFlow
        )
    }

    /**
     * Invoice drilldown.
     */
    fun getInvoiceDrillDown(
        year: Int,
        month: Int?
    ): InvoiceDrillDownResponse {

        val allClients =
            clientRepository
                .findAll()
                .associateBy { it.id }

        val orders =
            if (month != null) {
                orderRepository.findAllForMonth(year, month)
            } else {
                orderRepository.findAllForYear(year)
            }

        val items =
            orders.map { order ->

                val client =
                    allClients[order.clientId]

                val totalWithGst =
                    order.totalInvoiceAmount

                val gst =
                    order.totalGstAmount

                val withoutGst =
                    totalWithGst - gst

                InvoiceDrillDownDto(
                    orderId = order.id,
                    clientId = order.clientId,
                    clientName = client?.name ?: "Unknown Client",
                    orderDate = order.orderDate,
                    invoiceAmountWithoutGst = withoutGst,
                    gstAmount = gst,
                    totalAmountWithGst = totalWithGst
                )
            }

        return InvoiceDrillDownResponse(
            totalCount = items.size,
            totalInvoiceAmountWithOutGst =
                items.sumOf { it.invoiceAmountWithoutGst },
            totalInvoiceAmountWithGst =
                items.sumOf { it.totalAmountWithGst },
            totalGstAmount =
                items.sumOf { it.gstAmount },
            items = items
        )
    }

    /**
     * Expense drilldown.
     */
    fun getExpenseDrillDown(
        year: Int,
        month: Int?
    ): BalanceDrillDownResponse<ExpenseDrillDownDto> {

        val allClients =
            clientRepository
                .findAll()
                .associateBy { it.id }

        val orders =
            if (month != null) {
                orderRepository.findAllForMonth(year, month)
            } else {
                orderRepository.findAllForYear(year)
            }

        val expenseOrders =
            orders.filter { order ->
                order.totalExpense > 0
            }

        val items =
            expenseOrders.map { order ->

                val client =
                    allClients[order.clientId]

                ExpenseDrillDownDto(
                    orderId = order.id,
                    clientId = order.clientId,
                    clientName = client?.name ?: "Unknown Client",
                    orderDate = order.orderDate,
                    totalExpense = order.totalExpense
                )
            }

        return BalanceDrillDownResponse(
            totalCount = items.size,
            totalAmount =
                items.sumOf { it.totalExpense },
            items = items
        )
    }

    /**
     * Payment drilldown.
     */
    fun getPaymentDrillDown(
        year: Int,
        month: Int?
    ): BalanceDrillDownResponse<PaymentDrillDownDto> {

        val allClients =
            clientRepository
                .findAll()
                .associateBy { it.id }

        val payments =
            if (month != null) {
                paymentRepository.findAllForMonth(year, month)
            } else {
                paymentRepository.findAllForYear(year)
            }

        val items =
            payments.map { payment ->

                val client =
                    allClients[payment.clientId]

                PaymentDrillDownDto(
                    paymentId = payment.id,
                    clientId = payment.clientId,
                    clientName = client?.name ?: "Unknown Client",
                    paymentDate = payment.paymentDate,
                    amount = payment.amount,
                    paymentType = payment.paymentMode
                )
            }

        return BalanceDrillDownResponse(
            totalCount = items.size,
            totalAmount =
                items.sumOf { it.amount },
            items = items
        )
    }

    /**
     * Receivable drilldown.
     *
     * Uses Client Balance Snapshots only.
     *
     * month != null:
     * requested month
     *
     * month == null:
     * December
     */
    fun getReceivableDrillDown(
        year: Int,
        month: Int?
    ): BalanceDrillDownResponse<BalanceDrillDownDto> {

        val targetYearMonth =
            if (month != null) {
                YearMonth.of(year, month)
            } else {
                YearMonth.of(year, 12)
            }

        val clients =
            clientRepository.findAll()

        val items =
            clients.mapNotNull { client ->

                val snapshot =
                    getEffectiveSnapshotForMonth(
                        clientId = client.id,
                        yearMonth = targetYearMonth
                    )

                if (snapshot.closingBalance > 0L) {
                    BalanceDrillDownDto(
                        clientId = client.id,
                        clientName = client.name,
                        balanceAmount = snapshot.closingBalance
                    )
                } else {
                    null
                }
            }

        return BalanceDrillDownResponse(
            totalCount = items.size,
            totalAmount =
                items.sumOf { it.balanceAmount },
            items = items
        )
    }

    /**
     * Advance drilldown.
     *
     * Uses Client Balance Snapshots only.
     */
    fun getAdvanceDrillDown(
        year: Int,
        month: Int?
    ): BalanceDrillDownResponse<BalanceDrillDownDto> {

        val targetYearMonth =
            if (month != null) {
                YearMonth.of(year, month)
            } else {
                YearMonth.of(year, 12)
            }

        val clients =
            clientRepository.findAll()

        val items =
            clients.mapNotNull { client ->

                val snapshot =
                    getEffectiveSnapshotForMonth(
                        clientId = client.id,
                        yearMonth = targetYearMonth
                    )

                if (snapshot.closingBalance < 0L) {
                    BalanceDrillDownDto(
                        clientId = client.id,
                        clientName = client.name,
                        balanceAmount =
                            abs(snapshot.closingBalance)
                    )
                } else {
                    null
                }
            }

        return BalanceDrillDownResponse(
            totalCount = items.size,
            totalAmount =
                items.sumOf { it.balanceAmount },
            items = items
        )
    }

    /**
     * Discount drilldown.
     *
     * Uses Client Balance Snapshots only.
     */
    fun getDiscountDrillDown(
        year: Int,
        month: Int?
    ): List<DiscountDrillDownDto> {

        val targetYearMonth =
            if (month != null) {
                YearMonth.of(year, month)
            } else {
                YearMonth.of(year, 12)
            }

        val clients =
            clientRepository.findAll()

        return clients.mapNotNull { client ->

            val snapshot =
                getEffectiveSnapshotForMonth(
                    clientId = client.id,
                    yearMonth = targetYearMonth
                )

            if (snapshot.totalDiscount != 0L) {
                DiscountDrillDownDto(
                    clientId = client.id,
                    clientName = client.name,
                    totalDiscount = snapshot.totalDiscount
                )
            } else {
                null
            }
        }
    }

    /**
     * Calculate Global Summary using old/new snapshot delta.
     *
     * This method is used by the transactional ledger update flow.
     *
     * ClientMonthlyReporting is NOT involved.
     */
    /**
     * Apply one client's change to a month's summary.
     *
     * Flow fields move by the difference between the old and new snapshot.
     *
     * Stock fields replace old state with new state rather than applying a
     * single signed delta, because a client crossing zero leaves one bucket and
     * enters the other by different amounts. A balance going from +5,000 to
     * -3,000 is -5,000 receivable and +3,000 advance; a combined -8,000 cannot
     * express that.
     *
     * Every other client contributes nothing here, and that is exact rather
     * than an approximation: their balance did not move, so their bucket
     * contribution is unchanged and cancels. This is what lets a mutation touch
     * one client and still leave the totals correct for all of them.
     */
    fun calculateGlobalSummaryDelta(
        yearMonth: YearMonth,
        existingData: Map<String, Any?>,
        oldSnapshot: ClientBalanceSnapshot?,
        newSnapshot: ClientBalanceSnapshot,
        initialOpeningBalance: Long
    ): GlobalSummary {

        val oldOrdersCount = oldSnapshot?.totalOrdersCount ?: 0
        val oldInvoiceAmount = oldSnapshot?.totalInvoiceAmount ?: 0L
        val oldGstAmount = oldSnapshot?.totalGstAmounts ?: 0L
        val oldInvoiceAmountWithGst = oldSnapshot?.totalInvoiceAmountWithGst ?: 0L
        val oldExpenses = oldSnapshot?.totalExpenses ?: 0L
        val oldPayments = oldSnapshot?.totalPayments ?: 0L
        val oldDiscount = oldSnapshot?.totalDiscount ?: 0L
        val oldClosingBalance = oldSnapshot?.closingBalance ?: 0L
        val newOrdersCount = newSnapshot.totalOrdersCount
        val newInvoiceAmount = newSnapshot.totalInvoiceAmount
        val newGstAmount = newSnapshot.totalGstAmounts
        val newInvoiceAmountWithGst = newSnapshot.totalInvoiceAmountWithGst
        val newExpenses = newSnapshot.totalExpenses
        val newPayments = newSnapshot.totalPayments
        val newDiscount = newSnapshot.totalDiscount
        val newClosingBalance = newSnapshot.closingBalance

        val totalOrdersCount =
            ((existingData["totalOrdersCount"] as? Number)?.toLong() ?: 0L) -
                    oldOrdersCount +
                    newOrdersCount

        val totalInvoiceAmount =
            ((existingData["totalInvoiceAmount"] as? Number)?.toLong() ?: 0L) -
                    oldInvoiceAmount +
                    newInvoiceAmount

        val totalGstAmount =
            ((existingData["totalGstAmount"] as? Number)?.toLong() ?: 0L) -
                    oldGstAmount +
                    newGstAmount

        val totalInvoiceAmountWithGst =
            ((existingData["totalInvoiceAmountWithGst"] as? Number)?.toLong() ?: 0L) -
                    oldInvoiceAmountWithGst +
                    newInvoiceAmountWithGst

        val totalExpenses =
            ((existingData["totalExpenses"] as? Number)?.toLong() ?: 0L) -
                    oldExpenses +
                    newExpenses

        val totalPayments =
            ((existingData["totalPayments"] as? Number)?.toLong() ?: 0L) -
                    oldPayments +
                    newPayments

        val totalDiscount =
            ((existingData["totalDiscount"] as? Number)?.toLong() ?: 0L) -
                    oldDiscount +
                    newDiscount

        val oldStock = BalanceStock().apply {
            add(initialOpeningBalance, oldClosingBalance)
        }

        val newStock = BalanceStock().apply {
            add(initialOpeningBalance, newClosingBalance)
        }

        fun existingLong(key: String): Long =
            (existingData[key] as? Number)?.toLong() ?: 0L

        fun existingInt(key: String): Int =
            (existingData[key] as? Number)?.toInt() ?: 0

        return GlobalSummary(
            yearMonth = yearMonth.toString(),
            totalOrdersCount = totalOrdersCount.toInt(),
            totalInvoiceAmount = totalInvoiceAmount,
            totalGstAmount = totalGstAmount,
            totalInvoiceAmountWithGst = totalInvoiceAmountWithGst,
            totalExpenses = totalExpenses,
            totalPayments = totalPayments,
            totalDiscount = totalDiscount,

            netProfit = totalInvoiceAmount - totalExpenses - totalDiscount,
            cashFlow = totalPayments - totalExpenses,

            totalReceivableAmount =
                existingLong("totalReceivableAmount") -
                        oldStock.totalReceivable + newStock.totalReceivable,

            totalAdvanceAmount =
                existingLong("totalAdvanceAmount") -
                        oldStock.totalAdvance + newStock.totalAdvance,

            transactionReceivableAmount =
                existingLong("transactionReceivableAmount") -
                        oldStock.transactionReceivable + newStock.transactionReceivable,

            initialReceivableComponent =
                existingLong("initialReceivableComponent") -
                        oldStock.initialReceivable + newStock.initialReceivable,

            transactionAdvanceAmount =
                existingLong("transactionAdvanceAmount") -
                        oldStock.transactionAdvance + newStock.transactionAdvance,

            initialAdvanceComponent =
                existingLong("initialAdvanceComponent") -
                        oldStock.initialAdvance + newStock.initialAdvance,

            receivableClientCount =
                existingInt("receivableClientCount") -
                        oldStock.receivableClients + newStock.receivableClients,

            advanceClientCount =
                existingInt("advanceClientCount") -
                        oldStock.advanceClients + newStock.advanceClients,

            settledClientCount =
                existingInt("settledClientCount") -
                        oldStock.settledClients + newStock.settledClients,

            // A transaction never adds or removes a client.
            totalClientCount = existingInt("totalClientCount"),

            updatedAt = Instant.now()
        )
    }

    /**
     * Superseded by [seedGlobalSummaryForMonth] plus
     * [calculateGlobalSummaryDelta].
     *
     * This sums only the snapshots that exist for the month, so clients with no
     * activity that month contribute nothing and the receivable total silently
     * excludes them. It also produces transaction-only balances and leaves the
     * initial/transaction component fields at zero, so its output no longer
     * matches what every other write path stores.
     *
     * Kept only so any external caller still compiles.
     */
    @Deprecated(
        message =
            "Omits inactive clients and does not populate the component " +
                    "fields. Seed from the previous month and apply a delta " +
                    "instead.",
        level = DeprecationLevel.WARNING
    )
    fun rebuildGlobalSummaryForMonth(
        transaction: Transaction,
        yearMonth: YearMonth,
        replacementSnapshots: List<ClientBalanceSnapshot>
    ): GlobalSummary {

        val snapshots = snapshotRepository.getAllSnapshotsForMonth(transaction = transaction, yearMonth = yearMonth)
        val snapshotByClient = snapshots.associateBy { it.clientId }.toMutableMap()
        replacementSnapshots.filter { it.yearMonth.equals(yearMonth) }
            .forEach { snapshot -> snapshotByClient[snapshot.clientId] = snapshot }
        val effectiveSnapshots = snapshotByClient.values

        var totalOrdersCount = 0L
        var totalInvoiceAmount = 0L
        var totalGstAmount = 0L
        var totalInvoiceAmountWithGst = 0L
        var totalExpenses = 0L
        var totalPayments = 0L
        var totalReceivable = 0L
        var totalAdvance = 0L
        var totalDiscount = 0L

        effectiveSnapshots.forEach { snapshot ->
            totalOrdersCount += snapshot.totalOrdersCount.toLong()
            totalInvoiceAmount += snapshot.totalInvoiceAmount
            totalGstAmount += snapshot.totalGstAmounts
            totalInvoiceAmountWithGst += snapshot.totalInvoiceAmountWithGst
            totalExpenses += snapshot.totalExpenses
            totalPayments += snapshot.totalPayments
            totalDiscount += snapshot.totalDiscount

            when {
                snapshot.closingBalance > 0L -> {
                    totalReceivable += snapshot.closingBalance
                }

                snapshot.closingBalance < 0L -> {
                    totalAdvance += abs(snapshot.closingBalance)
                }
            }
        }

        return GlobalSummary(
            yearMonth = yearMonth.toString(),
            totalOrdersCount = totalOrdersCount.toInt(),
            totalInvoiceAmount = totalInvoiceAmount,
            totalGstAmount = totalGstAmount,
            totalInvoiceAmountWithGst = totalInvoiceAmountWithGst,
            totalExpenses = totalExpenses,
            totalPayments = totalPayments,
            totalReceivableAmount = totalReceivable,
            totalAdvanceAmount = totalAdvance,
            netProfit = totalInvoiceAmount - totalExpenses,
            cashFlow = totalPayments - totalExpenses,
            totalDiscount = totalDiscount,
            updatedAt = Instant.now()
        )
    }

}

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
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Component
import java.time.Instant
import java.time.LocalDate
import java.time.YearMonth
import java.time.temporal.ChronoUnit
import java.util.*
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
    private val log = LoggerFactory.getLogger(LedgerTransactionCoordinator::class.java);

    fun saveOrderAndRecalculateLedger(order: Order, transactionDate: LocalDate): LedgerCalculationResult {
        val transactionResult = transactionExecutor.execute(order.clientId) { transaction ->
            log.info("============================================================")
            log.info("DEBUG: saveOrderAndRecalculateLedger - START")
            log.info("DEBUG: orderId=${order.id}")
            log.info("DEBUG: transactionDate=$transactionDate")
            log.info("============================================================")

            // =============================================================
            // 1. READ EXISTING ORDER
            // =============================================================

            val existingOrder =
                orderRepository.findById(
                    transaction,
                    order.id
                )

            log.info("DEBUG: Existing Order = $existingOrder")

            // =============================================================
            // 2. DETERMINE OLD / NEW CLIENT AND MONTH
            // =============================================================

            val oldClientId = existingOrder?.clientId
            val newClientId = order.clientId
            val oldMonth = existingOrder?.let { YearMonth.from(it.orderDate) }
            val newMonth = YearMonth.from(order.orderDate)
            log.info("DEBUG: oldClientId=$oldClientId, " + "newClientId=$newClientId, " + "oldMonth=$oldMonth, " + "newMonth=$newMonth")
            // =============================================================
            // 3. DETERMINE AFFECTED CLIENTS
            // =============================================================

            val affectedClientIds =
                linkedSetOf<String>().apply {
                    oldClientId?.let { add(it) }
                    add(newClientId)
                }
            log.info("DEBUG: affectedClientIds=$affectedClientIds")
            // =============================================================
            // 4. READ ALL CLIENTS
            // =============================================================

            val clientsById = affectedClientIds.associateWith { clientId ->
                log.info("DEBUG: Reading Client $clientId")
                clientRepository.findById(transaction, clientId)
            }
            // =============================================================
            // 5. SNAPSHOTS ARE NOW READ TARGETED
            // =============================================================

            // =============================================================
            // 6. CALCULATE ALL CLIENT SNAPSHOTS IN MEMORY
            // =============================================================
            val allCalculatedSnapshots = mutableListOf<ClientBalanceSnapshot>()
            val finalClosingBalanceByClient = mutableMapOf<String, Long>()
            val oldSnapshotForGlobalByClientMonth = mutableMapOf<Pair<String, YearMonth>, ClientBalanceSnapshot>()
            val snapshotsByClient = mutableMapOf<String, List<ClientBalanceSnapshot>>()
            affectedClientIds.forEach { clientId ->
                log.info("------------------------------------------------------------")
                log.info("DEBUG: Processing Client $clientId")
                log.info("------------------------------------------------------------")
                val client = clientsById[clientId]
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
                require(affectedMonths.isNotEmpty()) { "No affected month found for Client $clientId." }
                val startMonth = affectedMonths.minOrNull()
                    ?: throw IllegalStateException("Unable to determine start month for Client $clientId.")
                log.info("DEBUG: Client $clientId startMonth=$startMonth")
                // ---------------------------------------------------------
                // Determine latest stored transaction month
                // ---------------------------------------------------------
                val latestStoredTransactionMonth = getEndMonth(
                    transaction = transaction,
                    clientId = clientId
                )
                log.info("DEBUG: Client $clientId " + "latestStoredTransactionMonth=$latestStoredTransactionMonth")
                // ---------------------------------------------------------
                // Determine latest stored snapshot month
                // ---------------------------------------------------------
                val latestSnapshot =
                    snapshotRepository.findLatestSnapshotForClient(
                        transaction = transaction,
                        clientId = clientId
                    )
                val latestSnapshotMonth = latestSnapshot?.yearMonth
                log.info("DEBUG: Client $clientId " + "latestSnapshotMonth=$latestSnapshotMonth")
                val endMonth =
                    listOfNotNull(
                        startMonth,
                        latestSnapshotMonth,
                        latestStoredTransactionMonth,
                        if (isNewClient) newMonth else null
                    ).maxOrNull()
                        ?: startMonth

                log.info(
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

                log.info(
                    "DEBUG: Client $clientId has " +
                            "${snapshots.size} snapshots in range " +
                            "$startMonth -> $endMonth"
                )

                snapshots.forEach { snapshot ->
                    log.info(
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

                log.info(
                    "DEBUG: Client $clientId openingBalance=$openingBalance"
                )

                // ---------------------------------------------------------
                // Process months
                // ---------------------------------------------------------

                var runningClosingBalance = openingBalance
                var monthCursor = startMonth
                while (!monthCursor.isAfter(endMonth)) {

                    log.info("------------------------------------------------------------")
                    log.info("DEBUG: PROCESSING MONTH " + "client=$clientId " + "month=$monthCursor")
                    log.info("------------------------------------------------------------")
                    val existingSnapshot = snapshotMap[monthCursor]
                    val isAffectedMonth = monthCursor in affectedMonths
                    log.info("DEBUG: existingSnapshot=$existingSnapshot")
                    log.info("DEBUG: isAffectedMonth=$isAffectedMonth")
                    log.info("DEBUG: runningClosingBalance BEFORE=$runningClosingBalance")
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

                    log.info("DEBUG: oldOrderForMonth=" + "${oldOrderForMonth?.id}")
                    log.info("DEBUG: newOrderForMonth=" + "${newOrderForMonth?.id}")
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

                            log.info("DEBUG: CASE 1 - " + "Existing snapshot + affected month")
                            log.info(
                                "DEBUG: Applying Order delta " +
                                        "for Client=$clientId " +
                                        "Month=$monthCursor " +
                                        "oldOrder=${oldOrderForMonth?.id} " +
                                        "newOrder=${newOrderForMonth?.id}"
                            )

                            oldSnapshotForGlobalByClientMonth[clientId to monthCursor] = existingSnapshot
                            monthSnapshot =
                                ledgerCalculationEngine.applyOrderDelta(
                                    existingSnapshot = existingSnapshot,
                                    openingBalance = runningClosingBalance,
                                    oldOrder = oldOrderForMonth,
                                    newOrder = newOrderForMonth
                                )
                        }

                        // =================================================
                        // CASE 2
                        // Existing snapshot + unaffected month
                        // =================================================

                        existingSnapshot != null -> {
                            log.info("DEBUG: CASE 2 - " + "Existing snapshot + unaffected month")
                            oldSnapshotForGlobalByClientMonth[clientId to monthCursor] = existingSnapshot
                            monthSnapshot =
                                ledgerCalculationEngine.propagateBalance(
                                    existingSnapshot = existingSnapshot,
                                    openingBalance = runningClosingBalance
                                )
                        }

                        // =================================================
                        // CASE 3
                        // Missing snapshot + affected month
                        // =================================================

                        isAffectedMonth -> {
                            log.info("DEBUG: CASE 3 - " + "Missing snapshot + affected month")
                            val recoveredSnapshot =
                                getOrRecalculateMonthSnapshot(
                                    transaction = transaction,
                                    clientId = clientId,
                                    yearMonth = monthCursor,
                                    openingBalance = runningClosingBalance,
                                    client = client
                                )

                            oldSnapshotForGlobalByClientMonth[clientId to monthCursor] = recoveredSnapshot
                            log.info(
                                "DEBUG: Recovered OLD snapshot " +
                                        "Client=$clientId " +
                                        "Month=$monthCursor " +
                                        "orders=${recoveredSnapshot.totalOrdersCount} " +
                                        "invoice=${recoveredSnapshot.totalInvoiceAmount} " +
                                        "invoiceWithGst=${recoveredSnapshot.totalInvoiceAmountWithGst} " +
                                        "payments=${recoveredSnapshot.totalPayments} " +
                                        "closing=${recoveredSnapshot.closingBalance}"
                            )

                            log.info(
                                "DEBUG: Applying Order delta to recovered " +
                                        "snapshot for Client=$clientId " +
                                        "Month=$monthCursor"
                            )

                            monthSnapshot =
                                ledgerCalculationEngine.applyOrderDelta(
                                    existingSnapshot = recoveredSnapshot,
                                    openingBalance = runningClosingBalance,
                                    oldOrder = oldOrderForMonth,
                                    newOrder = newOrderForMonth
                                )
                        }

                        // =================================================
                        // CASE 4
                        // Missing snapshot + unaffected month
                        // =================================================

                        else -> {

                            log.info(
                                "DEBUG: CASE 4 - " +
                                        "Missing snapshot + unaffected month"
                            )

                            val recoveredSnapshot = getOrRecalculateMonthSnapshot(
                                transaction = transaction,
                                clientId = clientId,
                                yearMonth = monthCursor,
                                openingBalance = runningClosingBalance,
                                client = client
                            )

                            oldSnapshotForGlobalByClientMonth[clientId to monthCursor] = recoveredSnapshot

                            log.info(
                                "DEBUG: Recovered snapshot " +
                                        "Client=$clientId " +
                                        "Month=$monthCursor " +
                                        "orders=${recoveredSnapshot.totalOrdersCount} " +
                                        "invoice=${recoveredSnapshot.totalInvoiceAmount} " +
                                        "invoiceWithGst=${recoveredSnapshot.totalInvoiceAmountWithGst} " +
                                        "payments=${recoveredSnapshot.totalPayments} " +
                                        "closing=${recoveredSnapshot.closingBalance}"
                            )
                            monthSnapshot = recoveredSnapshot
                        }
                    }

                    allCalculatedSnapshots.add(monthSnapshot)
                    runningClosingBalance = monthSnapshot.closingBalance
                    monthCursor = monthCursor.plusMonths(1)
                }

                finalClosingBalanceByClient[clientId] = runningClosingBalance

            }

            // =============================================================
            // 7. READ ALL GLOBAL SUMMARIES
            // =============================================================

            log.info("============================================================")
            log.info("DEBUG: READING GLOBAL SUMMARIES")
            log.info("============================================================")

            val uniqueGlobalMonths =
                allCalculatedSnapshots
                    .map { it.yearMonth }
                    .toSet()

            val existingGlobalSummaries =
                uniqueGlobalMonths.associateWith { yearMonth ->
                    getGlobalSummaryData(
                        transaction = transaction,
                        yearMonth = yearMonth
                    )
                }

            val updatedGlobalSummaries =
                mutableMapOf<YearMonth, GlobalSummary>()

            affectedClientIds.forEach { clientId ->
                val clientSnapshots =
                    allCalculatedSnapshots.filter {
                        it.clientId == clientId
                    }

                clientSnapshots.forEach { newSnapshot ->
                    val yearMonth =
                        newSnapshot.yearMonth

                    val existingGlobalData =
                        updatedGlobalSummaries[
                            yearMonth
                        ]?.let { summary ->
                            globalSummaryToMap(summary)
                        } ?: existingGlobalSummaries[
                            yearMonth
                        ]

                    val oldSnapshot =
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

                    /*
                     * A month with no summary document yet is seeded from the
                     * previous month, so the carried-forward balances of every
                     * inactive client are already in place, and then each
                     * affected client's own change is applied on top.
                     *
                     * This branch used to rebuild the month by summing all
                     * clients. On the two-client path that meant the first
                     * client triggered a full rebuild - which already included
                     * the second client - and the second client then applied
                     * its delta on top of a total it was already part of,
                     * double-counting. Seeding carries only the untouched
                     * clients forward, so each affected client is applied
                     * exactly once.
                     */
                    val baseData =
                        existingGlobalData
                            ?: globalSummaryService.globalSummaryToMap(
                                globalSummaryService.seedGlobalSummaryForMonth(
                                    transaction = transaction,
                                    yearMonth = yearMonth
                                )
                            )

                    val updatedSummary =
                        globalSummaryService.calculateGlobalSummaryDelta(
                            yearMonth = yearMonth,
                            existingData = baseData,
                            oldSnapshot = oldSnapshot,
                            newSnapshot = newSnapshot,
                            initialOpeningBalance =
                                clientsById[clientId]
                                    ?.initialOpeningBalance ?: 0L
                        )

                    updatedGlobalSummaries[
                        yearMonth
                    ] = updatedSummary
                }
            }


            // =============================================================
            // GLOBAL SUMMARY FINAL DEBUG
            // =============================================================

            log.info("============================================================")
            log.info("DEBUG: FINAL UPDATED GLOBAL SUMMARIES")
            log.info("============================================================")

            updatedGlobalSummaries.forEach { (month, summary) ->

                log.info(
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

            log.info("============================================================")
            log.info("DEBUG: All reads/calculations completed. " + "Starting writes.")
            log.info("============================================================")

            // -------------------------------------------------------------
            // Save Order
            // -------------------------------------------------------------

            log.info("DEBUG: Saving Order ${order.id}")
            orderRepository.save(transaction, order, existingOrder)
            // -------------------------------------------------------------
            // Save Snapshots
            // -------------------------------------------------------------
            allCalculatedSnapshots.forEach { snapshot ->
                log.info(
                    "DEBUG: Saving Snapshot " +
                            "client=${snapshot.clientId} " +
                            "month=${snapshot.yearMonth} " +
                            "orders=${snapshot.totalOrdersCount} " +
                            "invoice=${snapshot.totalInvoiceAmount} " +
                            "closing=${snapshot.closingBalance}"
                )
                snapshotRepository.saveSnapshot(transaction, snapshot)
            }

            // -------------------------------------------------------------
            // Save Global Summaries
            // -------------------------------------------------------------

            updatedGlobalSummaries.values.forEach { summary ->
                log.info(
                    "DEBUG: Saving Global Summary " +
                            "month=${summary.yearMonth} " +
                            "orders=${summary.totalOrdersCount} " +
                            "invoice=${summary.totalInvoiceAmount} " +
                            "receivable=${summary.totalReceivableAmount} " +
                            "advance=${summary.totalAdvanceAmount}"
                )
                saveGlobalSummary(transaction, summary)
            }

            // -------------------------------------------------------------
            // Update Clients
            // -------------------------------------------------------------

            finalClosingBalanceByClient.forEach { (clientId, closingBalance) ->

                val client =
                    clientsById[clientId]

                if (client != null) {

                    log.info(
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

            log.info("============================================================")
            log.info("DEBUG: TRANSACTION CALCULATION COMPLETE")
            log.info("============================================================")

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
//            log.info(
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
//                        log.info(
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
//                        log.info(
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
//                        log.info(
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
//                        log.info(
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
//                    log.info(
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
//            log.info(
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

            val clientId = existingOrder.clientId
            val client = clientRepository.findById(transaction, clientId)
            val startMonth = YearMonth.from(existingOrder.orderDate)

            // ---------------------------------------------------------
            // 4. Determine latest transaction month
            // ---------------------------------------------------------

            val latestStoredTransactionMonth = getEndMonth(
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

            val calculatedSnapshots = mutableListOf<ClientBalanceSnapshot>()
            val oldSnapshotForGlobalByMonth = mutableMapOf<YearMonth, ClientBalanceSnapshot>()
            var runningClosingBalance = openingBalance
            var monthCursor = startMonth
            while (!monthCursor.isAfter(endMonth)) {

                val existingSnapshot =
                    snapshotMap[monthCursor]

                val monthSnapshot: ClientBalanceSnapshot

                if (monthCursor.equals(startMonth)) {

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

                        log.info(
                            "DEBUG: Applying Order DELETE delta " +
                                    "for Client=$clientId " +
                                    "Month=$monthCursor " +
                                    "orderId=${existingOrder.id}"
                        )

                        oldSnapshotForGlobalByMonth[monthCursor] = existingSnapshot

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

                        log.info(
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

                        oldSnapshotForGlobalByMonth[monthCursor] = recoveredSnapshot

                        log.info(
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

                            oldSnapshotForGlobalByMonth[monthCursor] = existingSnapshot

                            ledgerCalculationEngine.propagateBalance(
                                existingSnapshot = existingSnapshot,
                                openingBalance = runningClosingBalance
                            )

                        } else {
                            val recoveredSnapshot =
                                getOrRecalculateMonthSnapshot(
                                    transaction = transaction,
                                    clientId = clientId,
                                    yearMonth = monthCursor,
                                    openingBalance = runningClosingBalance,
                                    client = client
                                )

                            oldSnapshotForGlobalByMonth[monthCursor] = recoveredSnapshot
                            recoveredSnapshot
                        }
                }
                calculatedSnapshots.add(monthSnapshot)
                runningClosingBalance = monthSnapshot.closingBalance
                monthCursor = monthCursor.plusMonths(1)
            }

            // ---------------------------------------------------------
            // 10. Read existing Global Summaries
            // ---------------------------------------------------------

            val existingGlobalSummaries =
                calculatedSnapshots.associateWith { snapshot ->
                    getGlobalSummaryData(
                        transaction = transaction,
                        yearMonth = snapshot.yearMonth
                    )
                }

            val updatedGlobalSummaries =
                calculatedSnapshots.associate { newSnapshot ->
                    val yearMonth = newSnapshot.yearMonth
                    val existingData = existingGlobalSummaries[newSnapshot]
                    val oldSnapshot =
                        oldSnapshotForGlobalByMonth[yearMonth] ?: getEffectiveSnapshot(
                            clientId = clientId,
                            yearMonth = yearMonth,
                            snapshots = snapshots
                        )

                    // Seed a missing month from the previous one, then apply
                    // this client's change on top. See the note in
                    // saveOrderAndRecalculateLedger.
                    val baseData =
                        existingData
                            ?: globalSummaryService.globalSummaryToMap(
                                globalSummaryService.seedGlobalSummaryForMonth(
                                    transaction = transaction,
                                    yearMonth = yearMonth
                                )
                            )

                    val updatedSummary =
                        globalSummaryService.calculateGlobalSummaryDelta(
                            yearMonth = yearMonth,
                            existingData = baseData,
                            oldSnapshot = oldSnapshot,
                            newSnapshot = newSnapshot,
                            initialOpeningBalance =
                                client?.initialOpeningBalance ?: 0L
                        )

                    yearMonth to updatedSummary
                }


            // ---------------------------------------------------------
            // 12. All reads/calculations completed
            //     Start writes
            // ---------------------------------------------------------
            log.info("DEBUG: All reads/calculations completed. " + "Starting writes.")
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
    ): Map<String, Any?>? {

        val docRef =
            db.collection("global_summaries")
                .document(yearMonth.toString())

        val document = transaction.get(docRef).get()

        if (!document.exists()) {
            return null
        }
        return document.data ?: emptyMap()
    }


    private fun saveGlobalSummary(transaction: Transaction, summary: GlobalSummary) {
        val docRef = db.collection("global_summaries")
            .document(summary.yearMonth)
        val data = hashMapOf<String, Any>(
            "yearMonth" to summary.yearMonth,
            "totalOrdersCount" to summary.totalOrdersCount,
            "totalInvoiceAmount" to summary.totalInvoiceAmount,
            "totalGstAmount" to summary.totalGstAmount,
            "totalInvoiceAmountWithGst" to summary.totalInvoiceAmountWithGst,
            "totalExpenses" to summary.totalExpenses,
            "totalPayments" to summary.totalPayments,
            "totalReceivableAmount" to summary.totalReceivableAmount,
            "totalAdvanceAmount" to summary.totalAdvanceAmount,
            "netProfit" to summary.netProfit,
            "cashFlow" to summary.cashFlow,
            "totalDiscount" to summary.totalDiscount,
            "updatedAt" to Date.from(summary.updatedAt)
        )
        transaction.set(docRef, data)
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

            val client = clientRepository.findById(transaction, clientId)

            val oldMonth = existingPayment?.let {
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

                        log.info(
                            "DEBUG: Applying Payment delta " +
                                    "for Client=$clientId " +
                                    "Month=$monthCursor " +
                                    "oldPayment=${existingPayment?.id} " +
                                    "newPayment=${payment.id}"
                        )

                        oldSnapshotForGlobalByMonth[monthCursor] = existingSnapshot
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

                        log.info(
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

                        oldSnapshotForGlobalByMonth[monthCursor] = recoveredSnapshot

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

                calculatedSnapshots.add(monthSnapshot)

                runningClosingBalance = monthSnapshot.closingBalance

                log.info(
                    "DEBUG: Client=$clientId " +
                            "Month=$monthCursor " +
                            "Closing=${monthSnapshot.closingBalance}"
                )

                monthCursor = monthCursor.plusMonths(1)
            }

            // ---------------------------------------------------------
            // 8. Read existing Global Summaries
            // ---------------------------------------------------------

            val existingGlobalSummaries =
                calculatedSnapshots.associateWith { snapshot ->
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
                            newSnapshot
                        ]

                    val oldSnapshot =
                        oldSnapshotForGlobalByMonth[
                            yearMonth
                        ] ?: getEffectiveSnapshot(
                            clientId = clientId,
                            yearMonth = yearMonth,
                            snapshots = snapshots
                        )

                    /*
                     * A month with no summary document yet is seeded from the
                     * previous month, so the carried-forward balances of every
                     * inactive client are already in place, and then this
                     * client's own change is applied on top.
                     *
                     * This used to rebuild the month from the affected
                     * snapshots and then let the next client apply a delta on
                     * top of a total that already included them, which
                     * double-counted.
                     */
                    val baseData =
                        existingData
                            ?: globalSummaryService.globalSummaryToMap(
                                globalSummaryService.seedGlobalSummaryForMonth(
                                    transaction = transaction,
                                    yearMonth = yearMonth
                                )
                            )

                    val updatedSummary =
                        globalSummaryService.calculateGlobalSummaryDelta(
                            yearMonth = yearMonth,
                            existingData = baseData,
                            oldSnapshot = oldSnapshot,
                            newSnapshot = newSnapshot,
                            initialOpeningBalance =
                                client?.initialOpeningBalance ?: 0L
                        )

                    yearMonth to updatedSummary
                }


            // ---------------------------------------------------------
            // 10. All reads/calculations completed.
            //     Start writes.
            // ---------------------------------------------------------

            log.info(
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
            val existingPayment = paymentRepository.findById(transaction, paymentId)
                ?: throw IllegalStateException("Payment not found: $paymentId")
            val client = clientRepository.findById(transaction, existingPayment.clientId)
            val clientId = existingPayment.clientId
            val startMonth = YearMonth.from(existingPayment.paymentDate)

            // ---------------------------------------------------------
            // 1. Determine latest snapshot month
            // ---------------------------------------------------------
            val latestSnapshot =
                snapshotRepository.findLatestSnapshotForClient(transaction = transaction, clientId = clientId)
            val latestSnapshotMonth = latestSnapshot?.yearMonth
            // ---------------------------------------------------------
            // 2. Determine latest transaction month
            // ---------------------------------------------------------
            val latestStoredTransactionMonth = getEndMonth(transaction = transaction, clientId = clientId)
            // ---------------------------------------------------------
            // 3. Determine end month
            // ---------------------------------------------------------
            val endMonth =
                listOfNotNull(startMonth, latestSnapshotMonth, latestStoredTransactionMonth).maxOrNull() ?: startMonth
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
                calculatedSnapshots.associateWith { snapshot ->
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
                            newSnapshot
                        ]

                    val oldSnapshot =
                        oldSnapshotForGlobalByMonth[
                            yearMonth
                        ] ?: getEffectiveSnapshot(
                            clientId = clientId,
                            yearMonth = yearMonth,
                            snapshots = snapshots
                        )

                    /*
                     * A month with no summary document yet is seeded from the
                     * previous month, so the carried-forward balances of every
                     * inactive client are already in place, and then this
                     * client's own change is applied on top.
                     *
                     * This used to rebuild the month from the affected
                     * snapshots and then let the next client apply a delta on
                     * top of a total that already included them, which
                     * double-counted.
                     */
                    val baseData =
                        existingData
                            ?: globalSummaryService.globalSummaryToMap(
                                globalSummaryService.seedGlobalSummaryForMonth(
                                    transaction = transaction,
                                    yearMonth = yearMonth
                                )
                            )

                    val updatedSummary =
                        globalSummaryService.calculateGlobalSummaryDelta(
                            yearMonth = yearMonth,
                            existingData = baseData,
                            oldSnapshot = oldSnapshot,
                            newSnapshot = newSnapshot,
                            initialOpeningBalance =
                                client?.initialOpeningBalance ?: 0L
                        )

                    yearMonth to updatedSummary
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
                calculatedSnapshots.associateWith { snapshot ->
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
                            newSnapshot
                        ]

                    val oldSnapshot =
                        oldSnapshotForGlobalByMonth[
                            yearMonth
                        ] ?: getEffectiveSnapshot(
                            clientId = clientId,
                            yearMonth = yearMonth,
                            snapshots = snapshots
                        )

                    /*
                     * A month with no summary document yet is seeded from the
                     * previous month, so the carried-forward balances of every
                     * inactive client are already in place, and then this
                     * client's own change is applied on top.
                     *
                     * This used to rebuild the month from the affected
                     * snapshots and then let the next client apply a delta on
                     * top of a total that already included them, which
                     * double-counted.
                     */
                    val baseData =
                        existingData
                            ?: globalSummaryService.globalSummaryToMap(
                                globalSummaryService.seedGlobalSummaryForMonth(
                                    transaction = transaction,
                                    yearMonth = yearMonth
                                )
                            )

                    val updatedSummary =
                        globalSummaryService.calculateGlobalSummaryDelta(
                            yearMonth = yearMonth,
                            existingData = baseData,
                            oldSnapshot = oldSnapshot,
                            newSnapshot = newSnapshot,
                            initialOpeningBalance =
                                client?.initialOpeningBalance ?: 0L
                        )

                    yearMonth to updatedSummary
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
                calculatedSnapshots.associateWith { snapshot ->
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
                            newSnapshot
                        ]

                    val oldSnapshot =
                        oldSnapshotForGlobalByMonth[
                            yearMonth
                        ] ?: getEffectiveSnapshot(
                            clientId = clientId,
                            yearMonth = yearMonth,
                            snapshots = snapshots
                        )

                    /*
                     * A month with no summary document yet is seeded from the
                     * previous month, so the carried-forward balances of every
                     * inactive client are already in place, and then this
                     * client's own change is applied on top.
                     *
                     * This used to rebuild the month from the affected
                     * snapshots and then let the next client apply a delta on
                     * top of a total that already included them, which
                     * double-counted.
                     */
                    val baseData =
                        existingData
                            ?: globalSummaryService.globalSummaryToMap(
                                globalSummaryService.seedGlobalSummaryForMonth(
                                    transaction = transaction,
                                    yearMonth = yearMonth
                                )
                            )

                    val updatedSummary =
                        globalSummaryService.calculateGlobalSummaryDelta(
                            yearMonth = yearMonth,
                            existingData = baseData,
                            oldSnapshot = oldSnapshot,
                            newSnapshot = newSnapshot,
                            initialOpeningBalance =
                                client?.initialOpeningBalance ?: 0L
                        )

                    yearMonth to updatedSummary
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

    fun deleteInvoiceAndRecalculateLedger(invoiceId: String): LedgerCalculationResult {

        val existingInvoice =
            invoiceRepository.findById(invoiceId) ?: throw IllegalArgumentException("Invoice not found: $invoiceId")

        val transactionResult = transactionExecutor.execute(existingInvoice.clientId) { transaction ->

            val existingInvoice = invoiceRepository.findById(transaction, invoiceId)
                ?: throw IllegalStateException("Invoice not found: $invoiceId")

            /*
             * Load only Orders linked to this Invoice.
             */
            val orders = existingInvoice.orderIds.map { orderId ->
                orderRepository.findById(transaction, orderId)
                    ?: throw IllegalStateException("Order $orderId not found.")
            }
            require(orders.isNotEmpty()) { "Invoice ${existingInvoice.id} has no linked Orders." }
            val clientId = existingInvoice.clientId
            val client = clientRepository.findById(transaction, clientId)

            /*
             * Invoice discount belongs to the Order month,
             * not invoiceDate.
             */
            val affectedMonths = orders
                .map { YearMonth.from(it.orderDate) }
                .toSet()

            require(affectedMonths.size == 1) { "All Orders in a single Invoice must belong to the same month." }
            val startMonth = affectedMonths.first()
            // ---------------------------------------------------------
            // 1. Find latest snapshot only.
            // ---------------------------------------------------------
            val latestSnapshot = snapshotRepository.findLatestSnapshotForClient(
                transaction = transaction,
                clientId = clientId
            )

            val latestSnapshotMonth = latestSnapshot?.yearMonth
            // ---------------------------------------------------------
            // 2. Determine latest stored transaction month.
            // ---------------------------------------------------------

            val latestStoredTransactionMonth = getEndMonth(
                transaction = transaction,
                clientId = clientId
            )

            // ---------------------------------------------------------
            // 3. Determine last month that needs propagation.
            // ---------------------------------------------------------

            val endMonth = listOfNotNull(
                startMonth,
                latestSnapshotMonth,
                latestStoredTransactionMonth
            ).maxOrNull() ?: startMonth

            // ---------------------------------------------------------
            // 4. Load only snapshots required for this calculation.
            // ---------------------------------------------------------

            val snapshots = snapshotRepository.getSnapshotsForClientInRange(
                transaction = transaction,
                clientId = clientId,
                startMonth = startMonth,
                endMonth = endMonth
            )
            val snapshotMap = snapshots.associateBy { it.yearMonth }
            // ---------------------------------------------------------
            // 5. Calculate opening balance for affected month.
            // ---------------------------------------------------------

            val openingBalance = snapshotRepository.getOpeningBalanceSeed(
                transaction = transaction,
                clientId = clientId,
                startMonth = startMonth,
                fallbackOpeningBalance =
                    client?.lastClosingBalance ?: 0L
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
                if (monthCursor.equals(startMonth)) {
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
                        oldSnapshotForGlobalByMonth[monthCursor] = existingSnapshot

                        monthSnapshot = ledgerCalculationEngine.applyDiscountDelta(
                            existingSnapshot = existingSnapshot,
                            openingBalance = runningClosingBalance,
                            discountDelta = -existingInvoice.discount
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
                        val recoveredSnapshot = getOrRecalculateMonthSnapshot(
                            transaction = transaction,
                            clientId = clientId,
                            yearMonth = monthCursor,
                            openingBalance = runningClosingBalance,
                            client = client
                        )

                        oldSnapshotForGlobalByMonth[monthCursor] = recoveredSnapshot

                        monthSnapshot = ledgerCalculationEngine.applyDiscountDelta(
                            existingSnapshot = recoveredSnapshot,
                            openingBalance = runningClosingBalance,
                            discountDelta = -existingInvoice.discount
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
                        oldSnapshotForGlobalByMonth[monthCursor] = existingSnapshot
                        monthSnapshot = ledgerCalculationEngine.propagateBalance(
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

                        oldSnapshotForGlobalByMonth[monthCursor] = recoveredSnapshot
                        monthSnapshot = recoveredSnapshot
                    }
                }

                calculatedSnapshots.add(monthSnapshot)
                runningClosingBalance = monthSnapshot.closingBalance
                monthCursor = monthCursor.plusMonths(1)
            }

            /*
             * Global summaries.
             */
            val existingGlobalSummaries =
                calculatedSnapshots.associateWith { snapshot ->
                    getGlobalSummaryData(
                        transaction = transaction,
                        yearMonth = snapshot.yearMonth
                    )
                }

            val updatedGlobalSummaries =
                calculatedSnapshots.associate { newSnapshot ->
                    val yearMonth = newSnapshot.yearMonth
                    val existingData = existingGlobalSummaries[newSnapshot]
                    val oldSnapshot = oldSnapshotForGlobalByMonth[yearMonth] ?: getEffectiveSnapshot(
                        clientId = clientId,
                        yearMonth = yearMonth,
                        snapshots = snapshots
                    )

                    // Seed a missing month from the previous one, then apply
                    // this client's change on top. See the note in
                    // saveOrderAndRecalculateLedger.
                    val baseData =
                        existingData
                            ?: globalSummaryService.globalSummaryToMap(
                                globalSummaryService.seedGlobalSummaryForMonth(
                                    transaction = transaction,
                                    yearMonth = yearMonth
                                )
                            )

                    val updatedSummary =
                        globalSummaryService.calculateGlobalSummaryDelta(
                            yearMonth = yearMonth,
                            existingData = baseData,
                            oldSnapshot = oldSnapshot,
                            newSnapshot = newSnapshot,
                            initialOpeningBalance =
                                client?.initialOpeningBalance ?: 0L
                        )

                    yearMonth to updatedSummary
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
            invoiceRepository.deleteById(transaction, existingInvoice.id)
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
                snapshotRepository.saveSnapshot(transaction, snapshot)
            }

            /*
             * Save global summaries.
             */
            updatedGlobalSummaries.values.forEach { summary ->
                saveGlobalSummary(
                    transaction, summary
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
        return snapshotRepository.getSnapshot(transaction, clientId, yearMonth)
    }

    private fun getOrRecalculateMonthSnapshot(
        transaction: Transaction,
        clientId: String,
        yearMonth: YearMonth,
        openingBalance: Long,
        client: Client?
    ): ClientBalanceSnapshot {
        log.info("DEBUG: getOrRecalculateMonthSnapshot - Checking for existing snapshot for $clientId in $yearMonth")
        val existingSnapshot = getMonthSnapshot(transaction = transaction, clientId = clientId, yearMonth = yearMonth)

        // Snapshot exists → cheap propagation.
        if (existingSnapshot != null) {
            return ledgerCalculationEngine.propagateBalance(
                existingSnapshot = existingSnapshot,
                openingBalance = openingBalance
            )
        }

        // Snapshot is missing → recovery case.
        // Read only this month's actual transactions.
        val orders = orderRepository.findAllForClientAndMonth(transaction, clientId, yearMonth)
        val payments = paymentRepository.findAllForClientAndMonth(transaction, clientId, yearMonth)
        val orderIds = orders.map { it.id }.toSet()
        val invoices = invoiceRepository.findAllForClientAndOrderIds(transaction, clientId, orderIds)
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

        log.info("DEBUG: getOrRecalculateMonthSnapshot - Recalculated snapshot for $clientId in $yearMonth: openingBalance=$openingBalance, totalOrdersCount=$totalOrdersCount, totalInvoiceAmount=$totalInvoiceAmount, totalInvoiceAmountWithGst=$totalInvoiceAmountWithGst, totalPayments=$totalPayments, totalExpenses=$totalExpenses, closingBalance=$closingBalance, totalGstAmounts=$totalGstAmounts, totalDiscount=$discountByMonth")
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

    private fun getOpeningBalance(
        startMonth: YearMonth,
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
            "yearMonth" to summary.yearMonth,
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

package com.clientledger.core.api

import com.clientledger.core.domain.summary.*
import com.clientledger.core.service.summary.GlobalSummaryService
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.RestController
import java.time.YearMonth

@RestController
@RequestMapping("/api/global-summary")
class GlobalSummaryController(private val globalSummaryService: GlobalSummaryService) {

    @GetMapping("/monthly")
    fun getMonthlySummary(@RequestParam yearMonth: String): Map<String, Any?> {
        // Example: /api/global-summary/monthly?yearMonth=2025-01
        return globalSummaryService.getGlobalSummaryForMonth(YearMonth.parse(yearMonth))
    }

    @GetMapping("/yearly")
    fun getYearlySummary(@RequestParam year: Int): GlobalSummary {
        // Example: /api/global-summary/yearly?year=2026
        return globalSummaryService.getGlobalSummaryForYear(year)
    }

    @GetMapping("/drill-down/invoices")
    fun getInvoiceDrillDown(
        @RequestParam year: Int,
        @RequestParam(required = false) month: Int?
    ): ResponseEntity<InvoiceDrillDownResponse> {
        return ResponseEntity.ok(globalSummaryService.getInvoiceDrillDown(year, month))
    }

    @GetMapping("/drill-down/expenses")
    fun getExpenseDrillDown(
        @RequestParam year: Int,
        @RequestParam(required = false) month: Int?
    ): ResponseEntity<BalanceDrillDownResponse<ExpenseDrillDownDto>> {
        return ResponseEntity.ok(globalSummaryService.getExpenseDrillDown(year, month))
    }

    @GetMapping("/drill-down/payments")
    fun getPaymentDrillDown(
        @RequestParam year: Int,
        @RequestParam(required = false) month: Int?
    ): ResponseEntity<BalanceDrillDownResponse<PaymentDrillDownDto>> {
        return ResponseEntity.ok(globalSummaryService.getPaymentDrillDown(year, month))
    }

    @GetMapping("/drill-down/receivables")
    fun getReceivableDrillDown(
        @RequestParam year: Int,
        @RequestParam(required = false) month: Int?
    ): ResponseEntity<BalanceDrillDownResponse<BalanceDrillDownDto>> {
        return ResponseEntity.ok(globalSummaryService.getReceivableDrillDown(year, month))
    }

    @GetMapping("/drill-down/advances")
    fun getAdvanceDrillDown(
        @RequestParam year: Int,
        @RequestParam(required = false) month: Int?
    ): ResponseEntity<BalanceDrillDownResponse<BalanceDrillDownDto>> {
        return ResponseEntity.ok(globalSummaryService.getAdvanceDrillDown(year, month))
    }

    @GetMapping("/drill-down/discounts")
    fun getDiscountDrillDown(
        @RequestParam year: Int,
        @RequestParam(required = false) month: Int?
    ): ResponseEntity<List<DiscountDrillDownDto>> {
        return ResponseEntity.ok(globalSummaryService.getDiscountDrillDown(year, month))
    }

    /**
     * Rebuild a span of monthly summaries from the snapshots.
     *
     * Two jobs:
     *
     * Migration. Summaries written before the flow/stock split hold
     * transaction-only receivable and advance totals and no component fields.
     * Seeding reads the previous month, so those old documents have to be
     * recomputed once, oldest first, before the incremental path can chain
     * correctly off them.
     *
     * Repair. Client reporting is written after the ledger transaction commits,
     * so a failure there leaves snapshots committed and summaries stale. This
     * is idempotent and recomputes from the snapshots, which remain the source
     * of truth.
     *
     * Walks every client per month, so it is deliberately not on a read path.
     *
     * Example: /api/global-summary/recompute?startYearMonth=2024-01&endYearMonth=2026-09
     */
    @PostMapping("/recompute")
    fun recomputeRange(
        @RequestParam startYearMonth: String,
        @RequestParam endYearMonth: String
    ): ResponseEntity<List<GlobalSummary>> {
        return ResponseEntity.ok(
            globalSummaryService.recomputeGlobalSummaryForRange(
                startMonth = YearMonth.parse(startYearMonth),
                endMonth = YearMonth.parse(endYearMonth)
            )
        )
    }
}
