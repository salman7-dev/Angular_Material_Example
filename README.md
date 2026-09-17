Our Current Problem and What We Are Trying to Build

We have a client-ledger system with approximately 300â€“500 clients and
around 2â€“3 years of transaction data.

We need to generate:

-   Monthly global reports
-   Yearly global reports
-   Total receivables
-   Total advances
-   Payment, order, GST, expense, and profit summaries
-   Client-wise drill-down details

The main problem is performance and Firestore read cost.

If we calculate every report by reading all client snapshots and
transactions each time, we may need to read a large amount of data
repeatedly. This makes reporting slow and increases Firestore
document-read costs.

OUR EXISTING COLLECTIONS

1.  Global Summary Report

The Global Summary Report contains monthly aggregate information, such
as:

-   Total orders
-   Total invoice amount
-   Total payments
-   Total expenses
-   Total GST
-   Total profit
-   Total receivables
-   Total advances
-   Other global totals

It mainly represents the overall business summary.

Currently, it contains data for clients who were active during that
month.

For example:

Total clients: 500 Active clients in February: 100 Inactive clients in
February: 400

The February summary contains information for the 100 active clients.
The remaining 400 clients do not have activity records for that month.

An inactive client means that the client did not create any order,
payment, or other relevant transaction during that month.

2.  Global Summary Report with Client Information

The Global Summary Report with Client Information contains monthly
client-level information for clients who had activity during that month.

It can contain:

-   Client ID
-   Month
-   Client activity
-   Transaction closing balance
-   Initial opening balance
-   Actual balance
-   Receivable amount
-   Advance amount
-   Client status
-   Other client-specific reporting information

This collection is mainly useful for drill-down reporting.

For example, the top-level report may show:

February Receivable = â‚¹10,00,000

When the user clicks the receivable amount, the system can search the
client-information report and show:

Client A â†’ â‚¹80,000 receivable Client B â†’ â‚¹60,000 receivable Client C â†’
â‚¹1,20,000 receivable

For a particular client, the drill-down can show:

Transaction balance â‚¹50,000 Initial opening â‚¹30,000 Actual receivable
â‚¹80,000

Therefore, the user can understand how the final receivable amount was
formed.

IMPORTANT BALANCE RULE

The transaction ledger balance and the initial opening balance are
separate concepts.

Actual Client Balance = Transaction Closing Balance + Initial Opening
Balance

The final balance is classified as follows:

Actual balance > 0 â†’ Receivable Actual balance < 0 â†’ Advance Actual
balance = 0 â†’ Settled

The Global Summary Report should remain transaction-oriented where
required, while the client-level drill-down can show the initial opening
balance separately.

For example:

Transaction closing balance = â‚¹70,000 Initial opening balance = â‚¹30,000
Actual receivable = â‚¹1,00,000

WHAT WE ARE TRYING TO BUILD

We want to convert the reporting system into a pre-calculated and
incrementally updated reporting system.

Instead of recalculating everything whenever the user opens a report:

User opens report â†“ Read transactions â†“ Read snapshots â†“ Loop through
clients â†“ Calculate everything again â†“ Display report

We want the system to calculate and update reporting data when business
data changes:

Order / Payment / Invoice changes â†“ Ledger calculation â†“ Update affected
snapshots â†“ Update Global Summary â†“ Update Client Information report â†“
Store ready-to-read values

Then, when the user opens the report:

User opens report â†“ Read prepared Global Summary â†“ Display report
immediately

This avoids repeatedly scanning years of transaction and snapshot data.

DESIRED TOP-DOWN REPORTING FLOW

When the user selects a month, such as February, the system should
immediately show the already-calculated summary:

February

-   Total receivable
-   Total advance
-   Total payments
-   Total orders
-   Total GST
-   Total expenses
-   Total profit

These values should already be available in the Global Summary Report.

The user should not have to wait for the system to recalculate all
clients and all historical snapshots.

DESIRED DRILL-DOWN FLOW

The drill-down should use the existing Global Summary Report with Client
Information.

The flow should be:

Global Summary Report â†“ User clicks Receivable amount â†“ Find
corresponding month and balance category â†“ Search Client Information
report â†“ Show client-wise receivable details

For example:

February Receivable = â‚¹10,00,000

The drill-down may show:

Client A Initial opening â‚¹30,000 Transaction balance â‚¹50,000 Actual
receivable â‚¹80,000

Client B Initial opening â‚¹20,000 Transaction balance â‚¹40,000 Actual
receivable â‚¹60,000

The user can then identify how much of the total came from:

-   Initial opening balances
-   Transaction balances
-   Individual clients

The drill-down should not recalculate the entire ledger again.

HANDLING ACTIVE AND INACTIVE CLIENTS

Suppose there are 500 clients:

Total clients: 500 Active clients in February: 100 Inactive clients in
February: 400

The active clients already have entries in the monthly
client-information report.

For the top-level summary, the system should still be able to show the
correct total receivable and advance values, including the applicable
initial opening balances of inactive clients.

However, we should avoid creating unnecessary individual monthly records
for every inactive client only to represent an unchanged balance.

The goal is:

Global Summary â†’ Fast aggregate totals

Client Information Report â†’ Client-level drill-down information

Snapshots â†’ Authoritative historical transaction state

UPDATE BEHAVIOR

Suppose Client Aâ€™s February order or payment is updated.

We should not recalculate all 500 clients.

The expected flow is:

Order / Payment update â†“ Ledger propagation â†“ Identify affected client â†“
Identify affected months â†“ Calculate the updated client state â†“ Update
affected snapshot records â†“ Update affected Global Summary records â†“
Update affected Client Information records

If the change affects February through May, then only Client Aâ€™s
relevant months should be updated:

Client A - February â†’ update Client A - March â†’ update Client A - April
â†’ update Client A - May â†’ update

Other clients should remain untouched.

This will reduce unnecessary Firestore writes and calculations.

WHY SIMPLE DELTA LOGIC IS RISKY

Initially, we considered directly adding or subtracting a delta from the
receivable and advance totals.

However, this becomes risky when a clientâ€™s balance changes from
receivable to advance or from advance to receivable.

For example:

Old actual balance = +â‚¹5,000 Old status = Receivable

After an update:

New actual balance = -â‚¹3,000 New status = Advance

The summary must change as follows:

Receivable: -â‚¹5,000 Advance: +â‚¹3,000

A simple generic delta of -â‚¹8,000 is not enough because it does not
clearly explain how much should be removed from receivables and how much
should be added to advances.

SAFER OLD-STATE VERSUS NEW-STATE APPROACH

For every affected client and month, we should calculate both the old
and new reporting states.

Old state:

-   oldReceivable
-   oldAdvance
-   oldActualBalance

New state:

-   newReceivable
-   newAdvance
-   newActualBalance

Then calculate the exact changes:

receivableDelta = newReceivable - oldReceivable

advanceDelta = newAdvance - oldAdvance

Example:

Old balance = +â‚¹5,000

oldReceivable = â‚¹5,000 oldAdvance = â‚¹0

New balance = -â‚¹3,000

newReceivable = â‚¹0 newAdvance = â‚¹3,000

The summary adjustment becomes:

Receivable adjustment = â‚¹0 - â‚¹5,000 = -â‚¹5,000 Advance adjustment =
â‚¹3,000 - â‚¹0 = +â‚¹3,000

This correctly handles:

-   Receivable increasing
-   Receivable decreasing
-   Advance increasing
-   Advance decreasing
-   Receivable changing into advance
-   Advance changing into receivable
-   Settled balance changes
-   Initial opening balance changes
-   Payment or order corrections

The old state and new state make the update deterministic and safer.

ROLE OF THE EIGHT-MONTH EDIT WINDOW

The system has an approximately eight-month editable window.

We want to use this rule to limit recalculation and updates.

Within the editable window:

-   Orders can be corrected
-   Payments can be corrected
-   Initial opening balances may be updated according to business rules
-   Affected snapshots can be recalculated
-   Affected global summaries can be updated
-   Affected client-information records can be updated

Older data is generally stable and should not be recalculated during
every report request.

The eight-month window therefore acts as a practical boundary for normal
correction and propagation.

However, if an exceptional older correction is allowed, the existing
ledger rules must still handle it correctly.

PROPOSED ARCHITECTURE

Orders / Payments / Invoices â†“ Ledger Calculation Engine â†“ Client
Monthly Snapshots â†“ Reporting Update Layer â†“ â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â” â”‚ â”‚ â–¼ â–¼
Global Summary Client Information Report Report â”‚ â”‚ â–¼ â–¼ Top-level totals
Drill-down details

The responsibilities should remain separated.

Ledger and Snapshots

These remain the authoritative source of transaction truth:

-   Orders
-   Payments
-   Invoices
-   Snapshots
-   Ledger Calculation Engine

Global Summary Report

This stores ready-to-display aggregate values:

-   Monthly orders
-   Monthly payments
-   Monthly GST
-   Monthly expenses
-   Monthly profit
-   Transaction receivable
-   Transaction advance
-   Applicable initial-balance components
-   Total receivable
-   Total advance

Global Summary Report with Client Information

This stores client-level values needed for drill-down:

-   Client ID
-   Month
-   Transaction closing balance
-   Initial opening balance
-   Actual closing balance
-   Receivable amount
-   Advance amount
-   Active/inactive information

Reporting Update Layer

This layer will:

1.  Receive the final old and new client/month states.
2.  Identify the affected months.
3.  Update the relevant client-information records.
4.  Update the global aggregate summary.
5.  Handle receivable and advance changes safely.
6.  Avoid recalculating unaffected clients.

FINAL OBJECTIVE

We are trying to build a reporting system where:

1.  Ledger snapshots remain the authoritative source.
2.  Global Summary contains ready-to-read aggregate values.
3.  Client Information contains client-level drill-down values.
4.  Active and inactive clients are handled correctly.
5.  Reports do not repeatedly scan two or three years of data.
6.  Only affected client/month records are updated after a transaction
    change.
7.  Receivable and advance changes are handled using old state versus
    new state.
8.  Initial opening balance remains separately identifiable.
9.  The eight-month edit window limits normal recalculation.
10. The user can open a report quickly and drill down without triggering
    a full ledger recalculation.

In short:

Ledger = Source of Truth Global Summary = Fast Aggregate Read Model
Client Information = Drill-Down Read Model Old State vs New State = Safe
Update Mechanism Eight-Month Window = Controlled Recalculation Boundary
