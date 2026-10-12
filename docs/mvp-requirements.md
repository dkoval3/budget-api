# BudgIT API — MVP Requirements

Oct 11, 2026 · @Dane

This document defines what the first usable version (MVP) of the BudgIT API must do, and what it
deliberately leaves out. It is the input for the first Spec Kit specs in this repository; it does
not replace them. Endpoint shapes are indicative only: the OpenAPI spec written in each feature
spec is the contract.

**Sources:**

1. The API constitution, `.specify/memory/constitution.md` (v1.0.0, cited as "API §I" etc.).
2. The UI constitution, `budget-app/.specify/memory/constitution.md` (v1.2.0, cited as "UI §I").
3. The current UI code in `budget-app/src`. It predates the UI constitution and will change, so it
   is used as evidence of which features the MVP needs, not of data shapes. Notably, the UI already
   marks undo/redo, "Recent Moves", budget filters and profile/budget settings as "not part of the
   MVP" (commented out in `BudgetOperationsBar.tsx`, `TopPanel.tsx` and `ExpandedProfile.tsx`).

**MVP goal:** a signed-in user can set up accounts, enter transactions by hand, build a monthly
budget with category groups and categories, assign money, set targets, and always see correct
balances, activity, available amounts and Amount to Assign, all computed by the API.

## 1. What the UI does today, and what the API must provide

| UI feature (file) | Works today on | MVP API need |
| --- | --- | --- |
| Budget table with groups, categories, assigned, activity (`BudgetTable`, `useBudget`) | `SampleBudget.ts` | Budget month read with computed totals |
| Add, rename and delete category groups and categories (`AddItemPopup`, `useBudget`) | local state, addressed by array index | CRUD by ID with sort keys |
| Edit assigned amount; "Assign" popup, manual tab (`BudgetCellWithInput`, `ManualTab`) | local state | Set assigned amount per category per month |
| "Assign" popup, automatic tab (`AutomaticTab`) | placeholder | Not in MVP |
| Amount to Assign (`AmountToAssign`) | hard-coded `totalAvailable` minus assigned | Computed by the API (API §I, §VIII) |
| Available balance panel: left over from last month, assigned, spending (`AvailableBalance`) | "left over" hard-coded to 0 | Carryover, assigned, activity, available per category |
| Month navigator (`MonthNavigator`) | local month/year state | Any budget month as `YYYY-MM` |
| Targets: create, edit, show progress (`TargetPanel`, `CreateEditTarget`) | local state; progress computed in UI | Target CRUD; progress computed by API (API §VIII) |
| Accounts sidebar with Cash/Credit groups, balances and type totals (`ExpandedSidebar`) | sample accounts; balances summed in UI | Account list with computed balances |
| Create account with name, type, initial balance (`CreateAccountForm`) | local state | Create account with starting balance |
| Rename account (`EditNamePopup`) | local state | Rename account |
| Account transaction table: add, edit, delete (`AccountTable`, `FunctionBar`) | local state | Transaction CRUD per account |
| Age of Money (`AgeOfMoney`) | hard-coded `191` | Not in MVP |
| Undo (`UndoRedoUtil`, commented-out buttons) | local undo stack | Not in MVP (own spec) |
| Budget filters: Underfunded, Overfunded, etc. (`BudgetFilters`) | commented out | Not in MVP |
| Login/logout via Cognito (`login/`, `ExpandedProfile`) | browser OIDC (`react-oidc-context`) | Bearer-token auth from the Next.js server (API §VI) |

## 2. In scope

Requirement IDs (e.g. `USR-1`) are for referencing from specs and tasks.

### 2.1 Users, budgets and settings

- **USR-1** A user record is provisioned from the identity provider's sign-up confirmation event,
  idempotently, with an internal user ID mapped to the Cognito `sub` (API §VI). A Lambda triggered
  by the event calls an internal provisioning endpoint. That endpoint is not publicly reachable,
  is authenticated with IAM or a shared secret (not a user token), and is not part of the
  published OpenAPI contract the UI uses (Q5).
- **USR-2** Provisioning also creates the user's single budget, with its own budget ID, and its
  default user settings. The budget starts with no category groups; system categories (Ready to
  Assign, credit card payment categories) are created automatically as needed.
- **USR-3** Every request is authenticated as a Cognito access token and resolves to the internal
  user. A valid token for a user that has not been provisioned yet (e.g. the sign-up event is
  delayed) returns 401 with code `USER_NOT_PROVISIONED`, so the UI can show a "finishing setup,
  try again" message instead of sending the user back to sign-in (Q4).
- **USR-4** User settings can be read and updated. The MVP has one setting: the profile timezone
  (`ZoneId`), used to decide "today" and the current budget month for target progress
  (API §III, §VIII). Provisioning leaves it unset; the UI sends the browser's timezone on first
  sign-in and the API stores it once. After that it changes only when the user edits it (Q8).

### 2.2 Category groups and categories

- **CAT-1** Create, rename and delete category groups.
- **CAT-2** Create, rename and delete categories within a group. Moving a category to another group
  is not in the MVP; the user deletes and recreates it (Q6).
- **CAT-3** Groups and categories are addressed by ID, never by position, and ordered by an
  explicit sort key (API §VIII). New items are placed where the UI asks (top or bottom). A
  dedicated reorder (drag-and-drop) endpoint is not in the MVP, but the schema supports it.
- **CAT-4** Deleting a category that has transactions or assigned money resolves those references
  at delete time (API §V): the request names a replacement category, and the deleted category's
  transactions and assignments move to it. Deleting a group deletes its categories under the same
  rule.
- **CAT-5** Names are required, trimmed, length-limited, and unique within their parent.

### 2.3 Budget months and assigning money

- **BUD-1** Read one budget month (`YYYY-MM`). The response contains every group and category in
  order, and for each category: assigned this month, activity this month, carryover from last
  month, available, its target (if any) with computed progress, and its version. Group subtotals
  and the month's Amount to Assign are included (API §I).
- **BUD-2** Any valid month can be read, even one with no assignments or transactions. Months are
  never created explicitly. In a month with no activity, each category's available amount is the
  amount left at the end of the previous month, and Amount to Assign reflects all earlier inflows
  and assignments.
- **BUD-2a** Budget months can be read and assigned from the month the budget was created up to 12
  months after the current month (in the user's timezone). Requests outside that range are
  rejected with 422 (Q9).
- **BUD-3** Set the assigned amount for one category in one month (absolute value, version
  checked). The UI's "Assign X to category" popup is a client-side convenience that sends the new
  absolute value. The response returns the updated category, its group subtotal and the new
  Amount to Assign (API §IV).
- **BUD-4** Amount to Assign follows API §VIII: "Ready to Assign" inflows add to it, assignments in
  any month subtract from it, a category's available amount rolls into the next month, and cash
  overspending in a category reduces the next month's Amount to Assign.
  Amount to Assign MAY go negative, for example when the user assigns more than is available or
  when a credit account's opening debt reduces it (ACC-1a). The API allows this: it does not
  reject the write, and it returns the negative value as is without clamping it to zero. The UI
  is responsible for informing the user of the problem (e.g. highlighting the negative amount).
- **BUD-5** Totals are computed on every read and are never stored in the MVP. SQL returns
  per-category, per-month aggregates (assigned and activity), never individual transactions, and a
  pure domain function applies rollover and the cash-overspending rule to produce carryover,
  available and Amount to Assign (API §I, §VII, §X).
- **BUD-6** A benchmark integration test, seeded at the sizing targets with five years of history,
  checks the budget-month read and a write against the p95 targets in API §X. If it fails, a
  per-category-per-month summary table is added as allowed by API §I.

### 2.4 Targets

- **TGT-1** Create, update and delete the target of a category (at most one per category).
- **TGT-2** Target types: `SET_ASIDE`, `FILL_UP`, `HAVE_BALANCE`. A target is either recurring
  (weekly with a weekday, monthly with a day of month, or yearly with a date) or custom (an amount
  with an optional due date, otherwise "eventually"). Due dates are calendar days (API §III).
- **TGT-3** Progress ("needed this month", "underfunded", met or not) is computed by the API, as
  pure domain functions covered by the shared fixtures, and returned with the budget month
  (API §VII, §VIII).

### 2.5 Accounts

- **ACC-1** Create an account with a name, a type (`CASH` or `CREDIT`) and a starting balance. The
  starting balance is recorded as a transaction: for a cash account it is an inflow to Ready to
  Assign; for a credit account see ACC-1a.
- **ACC-1a** A credit account's starting balance is recorded as a negative opening transaction
  with no category and a note telling the user to select the category the debt should be paid
  from (Q7):
    - While the opening transaction has no category, it reduces Amount to Assign by the debt
      amount.
    - Once the user selects a category, it reduces that category's available amount by the debt
      amount instead.
    - Either way, the same amount moves into the card's payment category, so it is set aside to
      pay the card, as with normal credit card spending (TXN-4).
    - Amount to Assign may go negative as a result. The API returns the negative value as is and
      never clamps it to zero; the UI highlights it.
    - Paying the debt down is budgeted through the card's payment category (ACC-2).
- **ACC-2** Creating a credit account creates its credit card payment category (API §VIII).
- **ACC-3** Rename an account.
- **ACC-4** List accounts with each account's balance and totals per account type, computed by the
  API.

### 2.6 Transactions

- **TXN-1** List an account's transactions, newest first. The response is a page object; in the
  MVP a single page contains everything (API §X).
- **TXN-2** Create, update and delete a transaction: date (`YYYY-MM-DD`), payee (free text),
  category (by ID, including Ready to Assign for inflows), memo, and signed amount in cents
  (inflows positive, outflows negative; API §I). Category is required except for transfers.
- **TXN-3** The client may supply the transaction ID on create (API §V).
- **TXN-4** Spending on a credit account moves money from the spending category to that card's
  payment category (API §VIII).
- **TXN-5** Transfers between two of the user's accounts are created, updated and deleted as a
  linked pair with no category (API §VIII). They are in the MVP because they are the only way to
  record a credit card payment.
- **TXN-6** Every transaction write returns the changed transaction(s), the affected account
  balances, and the recomputed budget values for the affected month(s) and categories (API §IV).

### 2.7 Platform and non-functional requirements

These are needed for the MVP because the constitution makes them gates for any feature.

- **PLT-1** Spring Boot 4 / WebFlux / R2DBC skeleton with Maven Wrapper, Flyway migrations, and the
  layer packages and ArchUnit rules from API §VII.
- **PLT-2** Contract-first OpenAPI 3.1 spec, generated interfaces, spec validation in integration
  tests, and the spec plus shared fixtures published on every merge to `main` (API §IV, §VII).
- **PLT-3** Problem Details errors, the status codes in API §IV, `Cache-Control: no-store`, and
  `X-Request-Id` handling.
- **PLT-4** Version checks returning `409 STALE_VERSION` on every mutable record (API §V).
- **PLT-5** Owner scoping on every query, cross-user access returning 404, and an RLS-ready schema
  (API §VI).
- **PLT-6** Config files, AppConfig, Secrets Manager and startup validation (API §VI).
- **PLT-7** Structured JSON logs to stdout, the log privacy test, Micrometer metrics, OpenTelemetry
  traces, and health/info on a separate management port (API §XI).
- **PLT-8** Docker image; Docker Compose stack (Postgres, API, UI, mock OIDC) used for local
  development and the cross-repo E2E suite; a dev-only seed (API §IX).
- **PLT-9** GitHub Actions running `./mvnw verify` with every gate in the constitution, plus
  dependency and image scanning (API §VI, Workflow).
- **PLT-10** One production environment on AWS (RDS for PostgreSQL, ECS or EKS) plus `local` and
  `test` (API, Technology Constraints).

## 3. Out of scope for the MVP

| Feature | Why it's out | Where it goes |
| --- | --- | --- |
| Undo/redo and "Recent Moves" | Commented out in the UI; constitution leaves it to its own spec | Undo feature spec |
| Automatic assign (`AutomaticTab`) | Placeholder in the UI | Later spec |
| Moving money between categories | No UI for it; "assign" covers the MVP | Later spec |
| Budget filters (Underfunded, Overfunded, Money Available, Snoozed) | Commented out; can be done client-side from target progress | UI-only later |
| Snoozing targets | Only appears in the commented-out filters | Later spec |
| Age of Money | Hard-coded in the UI | Later spec |
| Split transactions | No UI for it; the rule stays in the constitution for when it's built | Later spec |
| CSV import | Not in the UI; out of v1 in both constitutions (Q1) | Import spec |
| Bank linking and automatic imports (`linked` flag) | The goal after v1 | Bank linking spec |
| Deleting or closing financial accounts | No UI for it | Later spec |
| Deleting the user account and all data | Not needed while the owner is the only user; required before anyone else signs up (API §VI, Q2) | Before public launch |
| Drag-and-drop reordering | No UI for it; sort keys are in the schema | Later spec |
| Payee management and autocomplete | Payee is free text in the UI | Later spec |
| Cleared/reconciled status | The UI's `checked` flag is row selection, not cleared status | Later spec |
| Transaction search, filters, sorting and real paging | MVP returns one page (API §X) | When performance requires it |
| Scheduled or recurring transactions | Not in the UI | Later spec |
| Budget settings and account settings beyond timezone | Commented out in the UI | Later spec |
| Multiple budgets per user, shared budgets | Out of v1 (UI Scope); budget IDs keep the door open | Later |
| Multiple currencies, investments | Out of v1 (UI Scope); `currency` is always `USD` | Later |
| Server push (SSE/WebSockets) and streaming responses | Out of v1 (UI Scope) | Later |
| Rate limiting, `Idempotency-Key`, read replicas, cached totals, RLS policies | Deferred by the constitution until needed | Amendments or specs |
| Load tests and a staging environment | Deferred by your answers (8.6, 10.6) | Later |

## 4. Changes the UI will need

These follow from the constitutions; listed here so the UI specs can reference them.

- Address groups, categories, accounts and transactions by ID instead of array index
  (`useBudget`, `UndoRedoUtil`, `ManualTab`).
- Use integer cents instead of float dollars (sample data uses values like `143.23`).
- Use `YYYY-MM-DD` and `YYYY-MM` strings instead of `Date` objects and separate month/year state
  (`Transaction.date`, `Target.due`, `currentMonth`/`currentYear`).
- Send signed amounts (sample transactions store outflows as positive numbers).
- Reference a transaction's category by ID, not by name (`Transaction.category`, `categoryIdx`).
- Treat `checked` and `isSelected` as UI-only state; they are never sent to the API.
- Read Amount to Assign, balances, activity and target progress from the API instead of computing
  them (`calculateTotalAssigned`, `calculateAccountBalance`, `TargetPanel`).
- Move sign-in to the Next.js server and call the API only from the server (UI §VI), replacing
  `react-oidc-context` in the browser.
- Handle a 401 with code `USER_NOT_PROVISIONED` with a "finishing setup, try again" message
  instead of a redirect to `/login` (USR-3, UI §VI).
- Send the browser's timezone on first sign-in (USR-4).
- Highlight a negative Amount to Assign and tell the user they have assigned more than they have
  (BUD-4).
- Show the opening-debt note on a new credit account and let the user pick the category the
  debt is paid from (ACC-1a).

## 5. Decisions

These were open questions; the answers are folded into the requirements above.

| # | Question | Suggested default | Your answer |
| --- | --- | --- | --- |
| Q1 | The UI constitution lists CSV import as part of v1, but your answer to 9.4 says it doesn't need to be in v1. Which is right? | Leave CSV import out of the MVP and amend the UI constitution's v1 scope. | Leave CSV import out of the MVP and amend the UI constitution's v1 scope. |
| Q2 | Is deleting the user account in the MVP? | Out of the MVP while you're the only user; required before anyone else signs up. | Out of the MVP while you're the only user; required before anyone else signs up. |
| Q3 | The UI constitution's E2E list includes undo, but undo is out of the MVP. | E2E covers undo once the undo spec ships; amend the UI list to say so. | E2E covers undo once the undo spec ships; amend the UI list to say so. |
| Q4 | What happens if a valid token arrives before provisioning has run (e.g. the event is delayed)? | Return 401 with a `USER_NOT_PROVISIONED` code; the UI shows a "finishing setup, try again" message. | Return 401 with a `USER_NOT_PROVISIONED` code; the UI shows a "finishing setup, try again" message. |
| Q5 | How does the provisioning event reach the API's database? | A Lambda calls an internal, non-public provisioning endpoint authenticated with IAM or a shared secret. | A Lambda calls an internal, non-public provisioning endpoint authenticated with IAM or a shared secret. |
| Q6 | Can a category be moved to another group in the MVP? | No; delete and recreate is enough for the MVP. | No; delete and recreate is enough for the MVP. |
| Q7 | How is a credit account's starting balance budgeted? | Recorded as a negative opening transaction with no budget category, so the debt shows on the account but doesn't reduce Amount to Assign; paying it down is budgeted through the payment category. | Recorded as a negative opening transaction with no budget category and a note informing the user they should select a category to pay the debt from; If a category is not selected, reduce Amount to Assign by the debt amount, else reduce category money available by the debt amount; paying it down is budgeted through the payment category. |
| Q8 | What is the default profile timezone before the user changes it? | Sent by the UI from the browser on first sign-in and stored once; editable later. | Sent by the UI from the browser on first sign-in and stored once; editable later. |
| Q9 | Which months can be budgeted? | From the month the budget was created up to 12 months ahead. | From the month the budget was created up to 12 months ahead. |

## 6. Suggested spec order

Each API spec lands, with its OpenAPI change, before the matching UI spec (API Workflow).

1. **Platform skeleton** (PLT-1 to PLT-10): runnable service, CI gates, Compose stack, health
   endpoint, empty OpenAPI spec published.
2. **Users and auth** (USR-1 to USR-4).
3. **Budget structure** (CAT-1 to CAT-5), since transactions reference categories.
4. **Accounts and transactions** (ACC-1 to ACC-4, TXN-1 to TXN-3, TXN-6).
5. **Budget months and assigning** (BUD-1 to BUD-6).
6. **Credit cards and transfers** (ACC-1a, ACC-2, TXN-4, TXN-5).
7. **Targets** (TGT-1 to TGT-3).
