# PayLink - Requirements Pack (Phase 1)

Project: PayLink bill-payment and wallet backend (Python FastAPI, PostgreSQL)
Source: PayLink Year Work Project - Student Brief (sections 2, 3 and 4)
Priorities: Must, Should, Could

---

## 1. Problem Summary

PayLink is a company in Egypt, comparable to Fawry or Vodafone Cash. Customers pay monthly utility bills (electricity, water, mobile top-up and internet) from a digital wallet, either in the mobile app or through agents, who are small shops with a PayLink terminal. Customers want to avoid queues. The billers (the utility companies) want to be paid quickly and correctly.

PayLink earns money from a small service fee on every payment and pays a commission to agents. Because PayLink holds customers' money in wallets, correctness with money is the most important property of the system: a customer must never be charged twice, never be charged for a payment the biller did not receive, and no balance may ever become negative.

Our team builds the backend. A payment moves through these steps: identify the customer and check limits, ask the biller for the bill (inquiry), add the PayLink fee, run fraud checks, hold the money, call the biller with a unique reference, confirm the result, notify the customer, and reconcile with the biller's settlement file at night. Every step can fail, so most of the system is about handling failure: duplicate requests, slow or unreachable billers, asynchronous callbacks, notification outages and differences found during reconciliation.

The system must be easy to extend with new billers (each biller has a different protocol and behaviour), must keep an auditable ledger of every money movement, and must be ready for the busiest day of the month (target: 100 payments per second sustained for one minute).

Scope of the minimum system: Accounts and Wallet, Biller Catalog, Payment Processing, Biller Integration, Fees and Commissions and Notifications (6 modules), plus the extra modules Fraud Checks and Reconciliation. Billers: ElectroCo (REST JSON), AquaCo (SOAP-style XML) and TopUpNet (asynchronous REST JSON with callback). NetCo (daily batch CSV) is optional and is out of scope for the first release.

---

## 2. Stakeholders

| Stakeholder | Interest | What they need from the system |
|---|---|---|
| Customer | Pays bills quickly from a wallet without queueing | Correct balances, clear fees and totals, clear failure reasons, receipts by SMS and push, no double charges |
| Agent | Serves customers at a PayLink terminal and earns commission | Cash top-ups, paying on behalf of a customer, commission credited when the payment completes |
| Support employee | Resolves customer complaints | Search a payment by reference with its full history, block and unblock accounts, approve refunds, review reconciliation differences |
| Admin | Operates the platform | Manage billers and services, change fee and limit values without redeploying, upgrade customers, view daily totals and the audit log |
| PayLink management and finance | Revenue, trust and regulatory standing | Exact money handling, fee and commission accounting, audit trail, no financial losses |
| Billers (ElectroCo, AquaCo, TopUpNet, NetCo) | Receive payments fast and correctly | Unique references, no duplicate payments, correct settlement files |
| Regulators | Protect customers' money | Traceable, auditable money movements |
| SMS, email and push providers | Deliver notifications | Calls with retry; their outages must not break payments |
| Development team | Builds and maintains PayLink | Modular design, new billers without changing existing modules, automated tests and CI |
| Instructors and TAs | Grade the project | Complete, testable requirements, correct diagrams, working Docker and CI |

---

## 3. Business Rules Configuration Reference

These values are fixed by section 2.6 of the brief. They are stored in configuration (the `SystemConfig` table, editable by an admin without redeploying), never hard-coded. All money is an exact decimal with 2 places, rounded ROUND_HALF_UP, in EGP. Floating point is never used.

| Configuration key | Value | Used by |
|---|---|---|
| `limits.daily.basic` | 5000.00 EGP | FR-2, FR-18 |
| `limits.daily.verified` | 30000.00 EGP | FR-2, FR-18 |
| `limits.single.min` | 5.00 EGP | FR-6, FR-8, FR-18 |
| `limits.single.max` | 10000.00 EGP | FR-6, FR-8, FR-18 |
| `fees.electricity.fixed` | 2.50 EGP | FR-8, FR-16 |
| `fees.water.fixed` | 2.50 EGP | FR-8, FR-16 |
| `fees.mobile.percent` | 1.5 percent of the amount | FR-8, FR-16 |
| `fees.mobile.min` | 1.00 EGP | FR-8, FR-16 |
| `fees.internet.tier1.max_amount` | 200.00 EGP (inclusive) | FR-16 |
| `fees.internet.tier1.fee` | 2.00 EGP (amount up to 200.00) | FR-16 |
| `fees.internet.tier2.fee` | 4.00 EGP (amount above 200.00) | FR-16 |
| `promotion.biller_id` | one chosen biller | FR-16 |
| `promotion.start_date`, `promotion.end_date` | configurable dates, both inclusive | FR-16 |
| `commission.agent.percent` | 40 percent of the service fee | FR-17 |
| `velocity.max_payments` | 5 | FR-19 |
| `velocity.window_seconds` | 60 | FR-19 |
| `velocity.block_minutes` | 10 | FR-19 |
| `biller.timeout_seconds` | 10 (live billers) | FR-13 |
| `biller.query.retry_delays_minutes` | 1, 5, 15 | FR-13 |
| `biller.async_callback_timeout_minutes` | 30 (our decision, see ambiguity 4) | FR-13, FR-14 |
| `notification.max_retries` | 3 | FR-23 |
| `business.timezone` | Africa/Cairo (our decision, see ambiguity 2) | FR-18, FR-28 |

Payment rejection and failure reason codes used below: `BELOW_MINIMUM`, `ABOVE_MAXIMUM`, `DAILY_LIMIT_EXCEEDED`, `VELOCITY_EXCEEDED`, `ACCOUNT_BLOCKED`, `INSUFFICIENT_FUNDS`, `BILLER_DISABLED`, `BILLER_REFUSED`, `INVALID_TRANSITION`.

---

## 4. User Stories and Acceptance Criteria

Conventions:

- Each story uses the form: As a [role], I want [action], so that [benefit].
- Each acceptance criterion (AC) uses Given / When / Then.
- Priority values: Must, Should, Could. Stories marked Must cannot be removed.
- "Today" and "day" always mean the calendar day in Africa/Cairo.
- Note on numbering: in the brief, FR-17 to FR-28 appear under the heading of section 3.3, but they are functional requirements (they carry the FR prefix and describe behaviour). We treat all of FR-1 to FR-28 as functional requirements. The non-functional requirements are in section 5.
- "Hard case N" refers to the numbered situations in section 2.7 of the brief.

### 4.0 Overview

| ID | Title | Priority | Module |
|---|---|---|---|
| FR-1 | Register as a customer | Must | Accounts and Wallet |
| FR-2 | Upgrade a customer to verified | Should | Accounts and Wallet |
| FR-3 | View wallet balance and transaction history | Must | Accounts and Wallet |
| FR-4 | Top up the wallet | Must | Accounts and Wallet |
| FR-5 | Ledger entry for every balance change | Must | Accounts and Wallet |
| FR-6 | Manage billers and services | Must | Biller Catalog |
| FR-7 | List billers and required input fields | Must | Biller Catalog |
| FR-8 | Bill inquiry with amount, fee and total | Must | Payment Processing |
| FR-9 | Pay a bill from the wallet | Must | Payment Processing |
| FR-10 | Idempotent payment requests | Must | Payment Processing |
| FR-11 | Payment life cycle with allowed transitions only | Must | Payment Processing |
| FR-12 | Hold funds before calling the biller and release on failure | Must | Payment Processing |
| FR-13 | Timeout becomes Unknown and is re-queried | Must | Biller Integration |
| FR-14 | TopUpNet asynchronous callbacks without duplicates | Must | Biller Integration |
| FR-15 | At least three billers with different formats | Must | Biller Integration |
| FR-16 | Fee rules including the promotion period | Must | Fees and Commissions |
| FR-17 | Agent pays on behalf of a customer and earns commission | Should | Fees and Commissions |
| FR-18 | Reject payments above the single-payment or daily limit | Must | Fraud Checks |
| FR-19 | Velocity rule blocks the account | Should | Fraud Checks |
| FR-20 | Support searches a payment by reference | Must | Payment Processing |
| FR-21 | Support blocks and unblocks an account | Should | Accounts and Wallet |
| FR-22 | SMS and push message on completion and failure | Must | Notifications |
| FR-23 | Notification retry and undelivered status | Should | Notifications |
| FR-24 | Admin edits fee and limit values without redeploying | Should | Fees and Commissions |
| FR-25 | Audit log for every money movement and admin action | Must | Accounts and Wallet |
| FR-26 | Daily reconciliation report from a settlement file | Should | Reconciliation |
| FR-27 | Refund of a completed payment approved by support | Could | Accounts and Wallet |
| FR-28 | Admin views daily totals | Could | Reconciliation |

Priority count: 19 Must (FR-1, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 20, 22, 25), 7 Should (FR-2, 17, 19, 21, 23, 24, 26) and 2 Could (FR-27, 28). Total: 28 user stories.

---

### FR-1 - Register as a customer (Must)

**User story:** As a new customer, I want to register an account, so that I get a wallet and can pay my bills from my phone.

**Acceptance criteria:**

- **AC-1.1** **Given** a phone number and an email that are not registered, **When** the customer submits valid registration details, **Then** an account with role CUSTOMER and verification level BASIC is created, one EGP wallet is created whose available balance is 0.00 and held balance is 0.00, and an audit record is written.
- **AC-1.2** **Given** a phone number that is already registered, **When** someone registers with the same phone number, **Then** the request is rejected with a DUPLICATE_ACCOUNT error and no second account or wallet is created.
- **AC-1.3** **Given** a registration request with a missing or malformed phone number or email, **When** it is submitted, **Then** it is rejected with a validation error naming the invalid field and nothing is stored.
- **AC-1.4** **Given** a newly registered BASIC customer, **When** the customer asks for their daily payment limit, **Then** the limit returned is 5,000.00 EGP, read from configuration key `limits.daily.basic`.

### FR-2 - Upgrade a customer to verified (Should)

**User story:** As a customer, I want my account upgraded to the verified level, so that my daily payment limit rises from 5,000.00 EGP to 30,000.00 EGP.

**Acceptance criteria:**

- **AC-2.1** **Given** a BASIC customer whose identity documents were checked and approved by an admin (the check is simulated), **When** the admin upgrades the account, **Then** the verification level becomes VERIFIED, the daily limit for this customer becomes 30,000.00 EGP, and an audit record stores the old level, the new level, the admin and the reason.
- **AC-2.2** **Given** a BASIC customer who has already paid 4,000.00 EGP today, **When** the account is upgraded, **Then** the remaining daily allowance for the rest of that day is 26,000.00 EGP (30,000.00 minus 4,000.00).
- **AC-2.3** **Given** a VERIFIED customer, **When** the customer submits one payment of 10,001.00 EGP, **Then** it is rejected with ABOVE_MAXIMUM because the single-payment maximum of 10,000.00 EGP still applies.
- **AC-2.4** **Given** a user who is not an admin, **When** the user calls the upgrade operation, **Then** the request is refused with 403 and the verification level is unchanged.

### FR-3 - View wallet balance and transaction history (Must)

**User story:** As a customer, I want to see my available balance, my held balance and my transaction history, so that I know how much money I can spend and what happened to it.

**Acceptance criteria:**

- **AC-3.1** **Given** a customer whose wallet has ledger entries, **When** the customer requests the wallet, **Then** the response shows available balance equal to the sum of the AVAILABLE ledger entries and held balance equal to the sum of the HELD ledger entries, both as exact decimals with 2 places in EGP.
- **AC-3.2** **Given** a payment of 352.50 EGP in state FundsHeld, **When** the customer requests the wallet, **Then** 352.50 is included in the held balance and is not included in the available balance.
- **AC-3.3** **Given** a wallet with more than one page of ledger entries, **When** the customer requests the transaction history, **Then** the entries are returned newest first, in pages, each with entry type, amount, bucket, payment reference and timestamp.
- **AC-3.4** **Given** customer A is logged in, **When** customer A requests the wallet or history of customer B, **Then** the request is refused with 403 and no data is returned.

### FR-4 - Top up the wallet (Must)

**User story:** As a customer, I want to top up my wallet by simulated card or by agent cash, so that I have money to pay bills.

**Acceptance criteria:**

- **AC-4.1** **Given** a customer with an active wallet, **When** the customer tops up 500.00 EGP by simulated card and the simulated gateway approves, **Then** one ledger transaction credits the customer's available balance by 500.00 and debits the system external wallet by 500.00, so that the entries of the transaction sum to 0.00.
- **AC-4.2** **Given** a simulated card payment that the gateway declines, **When** the top-up is submitted, **Then** no ledger entry is created, the balance is unchanged and the customer receives the decline reason.
- **AC-4.3** **Given** an authenticated agent who received 500.00 EGP in cash from a customer, **When** the agent records a cash top-up for that customer's phone number, **Then** the customer's available balance increases by 500.00 and the ledger entry and the audit record store the agent as the actor.
- **AC-4.4** **Given** a top-up amount that is zero, negative or has more than 2 decimal places, **When** it is submitted, **Then** it is rejected with a validation error and nothing is stored.

### FR-5 - Ledger entry for every balance change (Must)

**User story:** As a finance officer at PayLink, I want every balance change recorded as a ledger entry and balances computed from the ledger, so that money movements are traceable and balances can never be overwritten.

**Acceptance criteria:**

- **AC-5.1** **Given** any balance change (top-up, hold, release, final charge, fee, commission or refund), **When** the change is committed, **Then** at least one ledger entry exists with a signed DECIMAL amount, bucket (AVAILABLE or HELD), entry type, wallet, payment (if any), actor, reason and UTC timestamp, and the amounts of all entries sharing the same transaction id sum to 0.00.
- **AC-5.2** **Given** a wallet, **When** its balance is requested, **Then** each balance is computed as the sum of its ledger entries per bucket and no stored balance column is read or written.
- **AC-5.3** **Given** an existing ledger entry, **When** an update or delete is attempted through the application, **Then** it is refused and the entry is unchanged; corrections are made only by new compensating entries.
- **AC-5.4** **Given** a ledger transaction in which one of its entries cannot be saved, **When** the transaction is committed, **Then** none of its entries is saved.

### FR-6 - Manage billers and services (Must)

**User story:** As an admin, I want to add, edit and disable billers and their services, so that customers can pay the right companies and I can stop a biller when needed.

**Acceptance criteria:**

- **AC-6.1** **Given** an admin, **When** the admin adds a biller (name, service type, adapter key) with at least one service (name, required input fields, minimum amount, maximum amount), **Then** the biller and service are stored as active, they appear in the biller list, and an audit record is written.
- **AC-6.2** **Given** a service with a minimum amount below 5.00 EGP, a maximum amount above 10,000.00 EGP, or a minimum greater than the maximum, **When** the admin saves it, **Then** it is rejected with a validation error because the global single-payment limits apply.
- **AC-6.3** **Given** an existing biller or service, **When** the admin edits its name, input fields or amounts, **Then** the change applies to new inquiries and payments, payments already created keep their stored data, and an audit record stores the old and new values.
- **AC-6.4** **Given** an active biller, **When** the admin disables it, **Then** it disappears from the customer biller list, new inquiries and new payments for it are refused with BILLER_DISABLED, and payments already in progress continue to their final state.
- **AC-6.5** **Given** a user who is not an admin, **When** the user adds, edits or disables a biller, **Then** the request is refused with 403 and nothing changes.

### FR-7 - List billers and required input fields (Must)

**User story:** As a customer, I want to see the list of billers and the input fields each one requires, so that I know what to enter before paying.

**Acceptance criteria:**

- **AC-7.1** **Given** the active billers ElectroCo (meter number), AquaCo (water account number) and TopUpNet (mobile phone number), **When** a customer requests the biller list, **Then** each biller is returned with its services, its required input fields (name, label and validation pattern) and the minimum and maximum amount.
- **AC-7.2** **Given** a disabled biller, **When** a customer requests the biller list, **Then** the disabled biller is not included.
- **AC-7.3** **Given** a caller who is not authenticated, **When** the caller requests the biller list, **Then** the request is refused with 401.

### FR-8 - Bill inquiry with amount, fee and total (Must)

**User story:** As a customer, I want to run a bill inquiry and see the amount, the fee and the total before paying, so that I know exactly what will be taken from my wallet.

**Acceptance criteria:**

- **AC-8.1** **Given** ElectroCo reports 350.00 EGP due for meter number M, **When** the customer runs an inquiry for M, **Then** the response shows amount 350.00, service fee 2.50 and total 352.50, and no money is held or moved.
- **AC-8.2** **Given** AquaCo reports 120.00 EGP due for a water account, **When** the customer runs an inquiry, **Then** the response shows amount 120.00, service fee 2.50 and total 122.50.
- **AC-8.3** **Given** a TopUpNet mobile top-up of 100.00 EGP (the customer chooses the amount because there is no bill), **When** the customer requests a quote, **Then** the response shows service fee 1.50 (1.5 percent) and total 101.50.
- **AC-8.4** **Given** a TopUpNet mobile top-up of 50.00 EGP, **When** the customer requests a quote, **Then** the response shows service fee 1.00 (the minimum, because 1.5 percent is 0.75) and total 51.00.
- **AC-8.5** **Given** a requested amount of 4.99 EGP or 10,000.01 EGP, **When** the customer requests a quote, **Then** the request is rejected with BELOW_MINIMUM or ABOVE_MAXIMUM respectively.
- **AC-8.6** **Given** a promotion that is active for ElectroCo, **When** the customer runs an inquiry for a 350.00 EGP bill, **Then** the response shows service fee 0.00 and total 350.00.
- **AC-8.7** **Given** a biller that fails or does not answer an inquiry within 10 seconds, **When** the customer runs an inquiry, **Then** the customer receives a "biller unavailable" error and no payment, hold or ledger entry is created.

### FR-9 - Pay a bill from the wallet (Must)

**User story:** As a customer, I want to pay a bill from my wallet, so that I do not have to queue at the biller's office.

**Acceptance criteria:**

- **AC-9.1** **Given** a customer with available balance 1,000.00 EGP and an ElectroCo inquiry showing amount 350.00, fee 2.50 and total 352.50, **When** the customer confirms the payment, **Then** a payment is created in state Initiated, moves to FundsHeld with available balance 647.50 and held balance 352.50, is sent to ElectroCo with a unique reference, and after ElectroCo confirms moves to Completed with held balance 0.00 and available balance 647.50.
- **AC-9.2** **Given** a payment of 350.00 EGP with fee 2.50 whose biller confirmed, **When** the hold is converted to a final charge, **Then** one ledger transaction debits HELD 352.50 from the customer wallet, credits 2.50 to the fee income wallet and credits 350.00 to the biller payable wallet.
- **AC-9.3** **Given** a customer with available balance 300.00 EGP and a payment total of 352.50 EGP, **When** the customer confirms the payment, **Then** the payment becomes Rejected with INSUFFICIENT_FUNDS, the biller is not called and no ledger entry is created.
- **AC-9.4** **Given** a payment that reached any state, **When** the response is returned, **Then** it contains the payment reference, state, amount, fee, total and timestamp.

### FR-10 - Idempotent payment requests (Must)

**User story:** As a customer, I want a repeated request with the same idempotency key to return the original result, so that a slow network never makes me pay twice.

**Acceptance criteria:**

- **AC-10.1** **Given** a payment created with idempotency key K by customer C that is already Completed, **When** the same request with key K arrives again, **Then** the original result (same payment id, reference and state) is returned and no new payment, ledger entry, biller call or notification is created (Hard case 1).
- **AC-10.2** **Given** two requests with the same key K from the same customer arriving at the same moment, **When** both are processed, **Then** exactly one payment exists and both responses show that payment.
- **AC-10.3** **Given** key K was already used with a different amount, biller or input, **When** a request with key K arrives, **Then** it is rejected with 409 IDEMPOTENCY_KEY_REUSED and nothing changes.
- **AC-10.4** **Given** a payment request without an idempotency key, **When** it is submitted, **Then** it is rejected with a validation error because the key is mandatory.
- **AC-10.5** **Given** the same key value used by two different customers, **When** both submit a payment, **Then** two separate payments are created because keys are unique per customer.

### FR-11 - Payment life cycle with allowed transitions only (Must)

**User story:** As a developer on the team, I want a payment to follow the life cycle of section 2.5 and no other transitions, so that money is never moved by an impossible state change.

**Acceptance criteria:**

- **AC-11.1** **Given** a payment in any state, **When** an event arrives, **Then** only these transitions are allowed: Initiated to FundsHeld (fraud checks pass and wallet has enough funds); Initiated to Rejected (fraud check fails or funds are insufficient); FundsHeld to SentToBiller (request sent to biller); SentToBiller to Completed (biller confirms); SentToBiller to Failed (biller refuses); SentToBiller to Unknown (no answer within the timeout); Unknown to Completed (biller later confirms by callback, retry or settlement file); Unknown to Failed (biller later refuses, or reconciliation finds no payment); Failed to Refunded (hold released to the wallet).
- **AC-11.2** **Given** an event that is not in the list above (for example FundsHeld to Completed, or Completed to Failed), **When** it is applied, **Then** it is refused with INVALID_TRANSITION, the state is unchanged and the attempt is recorded in the payment history.
- **AC-11.3** **Given** a payment in a final state (Completed, Rejected or Refunded), **When** any event arrives, **Then** the state stays the same and the event is refused.
- **AC-11.4** **Given** a payment in state Unknown, **When** a refund or hold release is attempted before it moves to Failed, **Then** it is refused, because money in Unknown is never returned automatically.
- **AC-11.5** **Given** any accepted transition, **When** it happens, **Then** a payment event with from state, to state, cause and timestamp is stored in the same database transaction as the state change.

### FR-12 - Hold funds before calling the biller and release on failure (Must)

**User story:** As a finance officer at PayLink, I want funds held before the biller is called and released if the payment fails, so that customers are never charged for payments the biller did not receive.

**Acceptance criteria:**

- **AC-12.1** **Given** a payment whose fraud checks passed and whose total is 352.50 EGP, **When** the funds are held, **Then** the available balance decreases by 352.50 and the held balance increases by 352.50 in one transaction, before the request is sent to the biller.
- **AC-12.2** **Given** a payment for which the hold was not made, **When** the payment is processed, **Then** the biller is never called.
- **AC-12.3** **Given** a payment that becomes Failed, **When** the hold is released, **Then** the held balance decreases by the total, the available balance increases by the same total, the payment becomes Refunded, and no fee income and no agent commission are credited.
- **AC-12.4** **Given** an available balance of 400.00 EGP and two payments with total 352.50 EGP each arriving from the same wallet at the same moment, **When** both are processed, **Then** exactly one reaches FundsHeld, the other is Rejected with INSUFFICIENT_FUNDS, and the available balance never becomes negative (Hard case 4).
- **AC-12.5** **Given** a payment that the biller confirms, **When** it completes, **Then** the hold is converted to a final charge and is not released to the wallet.

### FR-13 - Timeout becomes Unknown and is re-queried (Must)

**User story:** As a customer, I want a payment that gets no answer from the biller to be kept safe and checked again, so that I am neither charged twice nor refunded wrongly.

**Acceptance criteria:**

- **AC-13.1** **Given** ElectroCo takes 12 seconds to answer, which is more than the 10-second timeout, **When** the timeout is reached, **Then** the payment becomes Unknown, the money stays held, the customer is told that the payment is being processed, and no refund is made (Hard case 2).
- **AC-13.2** **Given** a payment that entered Unknown at time T, **When** the retry schedule runs, **Then** the biller is re-queried at T plus 1 minute, T plus 5 minutes and T plus 15 minutes, which is at most 3 re-queries.
- **AC-13.3** **Given** a re-query that returns a confirmation, **When** it is processed, **Then** the payment becomes Completed with fee, commission and notification; **Given** a re-query that returns a refusal, **When** it is processed, **Then** the payment becomes Failed, the hold is released and the payment becomes Refunded.
- **AC-13.4** **Given** a payment whose 3 re-queries all got no answer, **When** the third re-query ends, **Then** the payment stays Unknown with the money held and is not re-queried again until a callback or reconciliation resolves it.
- **AC-13.5** **Given** scheduled re-queries and an application restart, **When** the application starts again, **Then** the pending re-queries are reloaded from the database and still run at their scheduled times.

### FR-14 - TopUpNet asynchronous callbacks without duplicates (Must)

**User story:** As a developer on the team, I want to accept TopUpNet's asynchronous callbacks and ignore duplicates, so that a payment completes exactly once.

**Acceptance criteria:**

- **AC-14.1** **Given** a TopUpNet payment in state SentToBiller after TopUpNet replied "accepted", **When** a "success" callback with the payment's reference arrives, **Then** the payment becomes Completed once, with fee and commission handled as for any completed payment.
- **AC-14.2** **Given** a TopUpNet payment in state SentToBiller, **When** a "failed" callback arrives, **Then** the payment becomes Failed, the hold is released and the payment becomes Refunded.
- **AC-14.3** **Given** a TopUpNet payment that is already Completed, **When** a second "success" callback arrives, **Then** the callback is ignored, the response is 200 OK, no ledger entry is added, the state is unchanged and a DUPLICATE_CALLBACK event is logged (Hard case 6).
- **AC-14.4** **Given** a payment in a final state, **When** a callback arrives that contradicts it (for example "failed" for a Completed payment), **Then** the callback is ignored, a CONFLICTING_CALLBACK event is logged and the payment is flagged for support review.
- **AC-14.5** **Given** a callback with an unknown reference, **When** it arrives, **Then** the response is 404, the callback is logged and no state changes.
- **AC-14.6** **Given** a TopUpNet payment in state Unknown, **When** a callback arrives, **Then** it is processed as the later confirmation or refusal allowed by the life cycle.

### FR-15 - At least three billers with different formats (Must)

**User story:** As a customer, I want to pay electricity, water and mobile top-up bills, so that I can use PayLink for my everyday payments.

**Acceptance criteria:**

- **AC-15.1** **Given** a payment for ElectroCo, **When** the integration sends it, **Then** the ElectroCo adapter sends a REST JSON request with the unique payment reference and maps the answer to the PayLink result.
- **AC-15.2** **Given** a payment for AquaCo, **When** the integration sends it, **Then** the AquaCo adapter sends SOAP-style XML over HTTP and maps AquaCo's field names and error codes to the PayLink result.
- **AC-15.3** **Given** AquaCo refuses a payment with an error code, **When** the answer is processed, **Then** the payment becomes Failed, the hold is released, the payment becomes Refunded and the customer receives an SMS with a clear reason (Hard case 3).
- **AC-15.4** **Given** an AquaCo payment amount greater than the bill amount reported by the inquiry, **When** the customer submits it, **Then** PayLink rejects it before calling AquaCo with a validation error and no payment is created.
- **AC-15.5** **Given** a TopUpNet payment, **When** TopUpNet replies "accepted", **Then** the payment stays in SentToBiller and is not completed until the callback arrives.
- **AC-15.6** **Given** the AquaCo mock server is down, **When** customers pay ElectroCo and TopUpNet, **Then** those payments are processed normally and are not delayed by the AquaCo outage.
- **AC-15.7** **Given** a request to add a new biller, **When** the biller is added, **Then** only a new adapter in the Biller Integration module and configuration are added, and no existing module changes (Hard case 8).

### FR-16 - Fee rules including the promotion period (Must)

**User story:** As a finance officer at PayLink, I want fees calculated by the rules of section 2.6 including the promotion period, so that PayLink earns the right fee on every payment.

**Acceptance criteria:**

- **AC-16.1** **Given** an electricity or water payment of any amount (for example 5.00 or 10,000.00 EGP), **When** the fee is calculated, **Then** the fee is 2.50 EGP.
- **AC-16.2** **Given** mobile top-ups of 100.00, 40.00 and 333.33 EGP, **When** the fee is calculated, **Then** the fees are 1.50, 1.00 and 5.00 EGP (1.5 percent is 1.50, 0.60 raised to the 1.00 minimum, and 4.99995 rounded half up to 5.00).
- **AC-16.3** **Given** internet payments of 5.00, 200.00 and 200.01 EGP, **When** the fee is calculated, **Then** the fees are 2.00, 2.00 and 4.00 EGP (tiered: 2 EGP up to and including 200.00, 4 EGP above).
- **AC-16.4** **Given** a promotion configured for ElectroCo from 2026-12-01 to 2026-12-07 (both dates inclusive), **When** a payment for ElectroCo is quoted on any date in that period, **Then** the fee is 0.00 and the total equals the amount.
- **AC-16.5** **Given** the same promotion, **When** a payment for AquaCo is quoted on 2026-12-03, or a payment for ElectroCo is quoted on 2026-12-08, **Then** the normal fee of 2.50 EGP applies, because the promotion covers only one chosen biller and only its dates.
- **AC-16.6** **Given** any fee or commission calculation, **When** it is computed, **Then** exact decimal arithmetic is used and the result is rounded ROUND_HALF_UP to 2 places, never floating point.

### FR-17 - Agent pays on behalf of a customer and earns commission (Should)

**User story:** As an agent, I want to pay a bill on behalf of a customer and earn commission, so that I am rewarded for serving customers at my terminal.

**Acceptance criteria:**

- **AC-17.1** **Given** an authenticated agent and a customer who confirmed with a simulated one-time code and whose wallet has available balance 1,000.00 EGP, **When** the agent submits an ElectroCo payment of 350.00 EGP with fee 2.50, **Then** the payment is created with the agent's id, the funds are held from the customer's wallet, and the same life cycle, limits and fraud checks apply.
- **AC-17.2** **Given** that agent payment, **When** it becomes Completed, **Then** the agent's wallet is credited 1.00 EGP (40 percent of the 2.50 fee) in the same ledger transaction as the final charge, and the fee income wallet keeps 1.50.
- **AC-17.3** **Given** an agent payment whose fee is 1.50 EGP (mobile top-up of 100.00), **When** it becomes Completed, **Then** the agent is credited 0.60 EGP.
- **AC-17.4** **Given** an agent payment during a promotion where the fee is 0.00, **When** it becomes Completed, **Then** the commission is 0.00 and no commission ledger entry is created.
- **AC-17.5** **Given** an agent payment that is Rejected, Failed or Unknown, **When** its state changes, **Then** no commission is credited; if an Unknown payment later becomes Completed, the commission is credited at that moment.
- **AC-17.6** **Given** a user who is not an agent, **When** the user submits a payment on behalf of a customer, **Then** the request is refused with 403.

### FR-18 - Reject payments above the single-payment or daily limit (Must)

**User story:** As a PayLink manager, I want payments above the single-payment limit or the daily limit rejected with a clear reason, so that fraud losses and mistakes are limited.

**Acceptance criteria:**

- **AC-18.1** **Given** a payment amount of 4.99 EGP, **When** it is submitted, **Then** it is Rejected with BELOW_MINIMUM; **Given** an amount of 5.00 EGP, **When** it is submitted, **Then** it is accepted for processing.
- **AC-18.2** **Given** a VERIFIED customer and an amount of 10,000.01 EGP, **When** it is submitted, **Then** it is Rejected with ABOVE_MAXIMUM; **Given** an amount of 10,000.00 EGP, **When** it is submitted, **Then** it is accepted for processing.
- **AC-18.3** **Given** a BASIC customer who has already paid 4,800.00 EGP today, **When** the customer submits a payment of 250.00 EGP, **Then** the payment is Rejected with DAILY_LIMIT_EXCEEDED (4,800.00 plus 250.00 is 5,050.00, above 5,000.00), the message shows the remaining 200.00 EGP, and no hold is made.
- **AC-18.4** **Given** a BASIC customer who has already paid 4,800.00 EGP today, **When** the customer submits a payment of 200.00 EGP, **Then** the payment is accepted, because the daily total reaches exactly 5,000.00.
- **AC-18.5** **Given** a VERIFIED customer who has already paid 29,900.00 EGP today, **When** the customer submits 200.00 EGP, **Then** it is Rejected with DAILY_LIMIT_EXCEEDED; **When** the customer submits 100.00 EGP, **Then** it is accepted.
- **AC-18.6** **Given** payments of the customer today, **When** the daily total is computed, **Then** the total counts the bill amount (not the fee) of payments in FundsHeld, SentToBiller, Unknown and Completed, and does not count payments that are Rejected, Failed or Refunded.
- **AC-18.7** **Given** a new calendar day in Africa/Cairo, **When** the first payment of the day is evaluated, **Then** the daily total starts again at 0.00.

### FR-19 - Velocity rule blocks the account (Should)

**User story:** As a PayLink manager, I want an account that makes more than 5 payments in 60 seconds blocked for 10 minutes, so that automated abuse and runaway clients are stopped.

**Acceptance criteria:**

- **AC-19.1** **Given** a customer who made 5 payments within the last 60 seconds, **When** the customer submits a 6th new payment within that window, **Then** the payment is Rejected with VELOCITY_EXCEEDED and the account is blocked for 10 minutes.
- **AC-19.2** **Given** a customer who makes exactly 5 payments within 60 seconds, **When** all five are submitted, **Then** all five are accepted for processing, because the rule triggers only on more than 5.
- **AC-19.3** **Given** an account blocked by the velocity rule at time T, **When** any payment is submitted before T plus 10 minutes, **Then** it is Rejected with ACCOUNT_BLOCKED; **When** a payment is submitted at or after T plus 10 minutes, **Then** it is accepted for processing without manual action.
- **AC-19.4** **Given** a repeated request with an idempotency key that was already used, **When** it arrives, **Then** it is not counted by the velocity rule.
- **AC-19.5** **Given** 5 payments spread over more than 60 seconds so that no 60-second window contains more than 5, **When** the next payment is submitted, **Then** the account is not blocked.

### FR-20 - Support searches a payment by reference (Must)

**User story:** As a support employee, I want to search a payment by its reference and see its full history, so that I can answer customer complaints.

**Acceptance criteria:**

- **AC-20.1** **Given** a support employee and an existing payment reference, **When** the employee searches by that reference, **Then** the response shows the payment (customer, agent if any, biller, amount, fee, total, state) and its full history in time order: state transitions, biller requests and answers, callbacks, ledger entries and notifications.
- **AC-20.2** **Given** a reference that does not exist, **When** the employee searches for it, **Then** the response is 404 with an empty result and no internal details.
- **AC-20.3** **Given** a user who is not a support employee or admin, **When** the user searches by reference, **Then** the request is refused with 403.

### FR-21 - Support blocks and unblocks an account (Should)

**User story:** As a support employee, I want to block and unblock an account, so that I can stop suspicious activity and restore service after review.

**Acceptance criteria:**

- **AC-21.1** **Given** an ACTIVE account, **When** a support employee blocks it with a reason, **Then** the status becomes BLOCKED, an audit record stores who, when and why, and every new payment attempt is Rejected with ACCOUNT_BLOCKED.
- **AC-21.2** **Given** a BLOCKED account, **When** a support employee unblocks it with a reason, **Then** the status becomes ACTIVE, any velocity block timer is cleared and an audit record is written.
- **AC-21.3** **Given** a block or unblock request without a reason, **When** it is submitted, **Then** it is rejected with a validation error and the status is unchanged.
- **AC-21.4** **Given** an account with a payment in FundsHeld, SentToBiller or Unknown, **When** support blocks the account, **Then** the payment in progress continues to its final state under the normal rules.
- **AC-21.5** **Given** a user who is not a support employee or admin, **When** the user blocks or unblocks an account, **Then** the request is refused with 403.

### FR-22 - SMS and push message on completion and failure (Must)

**User story:** As a customer, I want an SMS and a push message when my payment completes or fails, so that I know the result without opening the app.

**Acceptance criteria:**

- **AC-22.1** **Given** a payment that becomes Completed, **When** the completion is recorded, **Then** an SMS and a push message are created from the completion template with biller name, amount, fee, total and payment reference.
- **AC-22.2** **Given** a payment that becomes Failed (for example AquaCo refused with an error code), **When** the failure is recorded, **Then** an SMS and a push message are created that state a clear reason in plain language and say that the money was returned to the wallet.
- **AC-22.3** **Given** a payment that becomes Unknown, **When** the timeout is reached, **Then** the customer receives a push message that the payment is being processed and the money stays held.
- **AC-22.4** **Given** a notification that cannot be sent, **When** the sending fails, **Then** the payment state and the ledger are unchanged.
- **AC-22.5** **Given** any notification, **When** it is created, **Then** a notification record stores the channel, template, status and number of attempts.

### FR-23 - Notification retry and undelivered status (Should)

**User story:** As a customer, I want failed notifications retried and then marked undelivered, so that temporary provider outages do not lose my receipt silently.

**Acceptance criteria:**

- **AC-23.1** **Given** the SMS provider is down when a payment completes, **When** the SMS is sent, **Then** the payment stays Completed, the first attempt fails, the SMS is retried 3 times (4 attempts in total), and after the third retry fails it is marked UNDELIVERED (Hard case 5).
- **AC-23.2** **Given** an SMS whose second attempt succeeds, **When** the retry succeeds, **Then** the status becomes DELIVERED and no further retries are made.
- **AC-23.3** **Given** an email that fails, **When** it is sent, **Then** it follows the same rule: 3 retries, then UNDELIVERED.
- **AC-23.4** **Given** an SMS marked UNDELIVERED, **When** the push message for the same payment succeeds, **Then** the push is DELIVERED and its status is independent of the SMS status.

### FR-24 - Admin edits fee and limit values without redeploying (Should)

**User story:** As an admin, I want to edit fee and limit values without redeploying, so that I can react to business decisions quickly.

**Acceptance criteria:**

- **AC-24.1** **Given** an admin who changes `limits.daily.basic` from 5000.00 to 6000.00, **When** the next payment of a BASIC customer is evaluated, **Then** the limit 6,000.00 is used without restart or deployment.
- **AC-24.2** **Given** an admin who changes `fees.electricity.fixed` to 3.00, **When** the next ElectroCo inquiry is run, **Then** the fee is 3.00, while payments created earlier keep the fee they were created with.
- **AC-24.3** **Given** an invalid value (negative, not a decimal, or a daily limit below the minimum single payment), **When** the admin saves it, **Then** it is rejected with a validation error and the old value remains.
- **AC-24.4** **Given** any configuration change, **When** it is saved, **Then** an audit record stores the key, the old value, the new value, the admin and the time.
- **AC-24.5** **Given** a user who is not an admin, **When** the user changes a configuration value, **Then** the request is refused with 403.

### FR-25 - Audit log for every money movement and admin action (Must)

**User story:** As a PayLink manager, I want every money movement and admin action written to an audit log, so that who did what, when and why can always be shown.

**Acceptance criteria:**

- **AC-25.1** **Given** any ledger transaction (top-up, hold, release, charge, fee, commission or refund), **When** it is committed, **Then** an audit record with actor (user or SYSTEM), UTC timestamp, reason and the affected payment and wallet exists and is saved in the same database transaction.
- **AC-25.2** **Given** an admin or support action (biller change, configuration change, upgrade, block, unblock, refund approval), **When** it is performed, **Then** an audit record is written with the old and new values.
- **AC-25.3** **Given** an audit record that cannot be written, **When** the operation is committed, **Then** the whole operation is rolled back.
- **AC-25.4** **Given** an existing audit record, **When** an update or delete is attempted through the application, **Then** it is refused.
- **AC-25.5** **Given** any automated test run, **When** the consistency check runs, **Then** the number of ledger transactions without an audit record is 0.

### FR-26 - Daily reconciliation report from a settlement file (Should)

**User story:** As a support employee, I want a daily reconciliation report from the biller's settlement file, so that I can find and review differences between PayLink and the biller.

**Acceptance criteria:**

- **AC-26.1** **Given** a settlement file for biller B and date D listing each biller reference with amount and time, **When** reconciliation runs, **Then** every line is matched to PayLink payments by biller reference and the report lists the result per payment as MATCHED, MISSING_IN_PAYLINK, MISSING_AT_BILLER, AMOUNT_MISMATCH or STATUS_MISMATCH, with a count per type.
- **AC-26.2** **Given** a settlement file that lists a payment which PayLink has as Failed, **When** reconciliation runs, **Then** the difference is flagged as STATUS_MISMATCH in the report for support to review and no state or money is changed automatically (Hard case 7).
- **AC-26.3** **Given** a payment in Unknown that the settlement file lists, **When** reconciliation runs, **Then** the payment becomes Completed through the life cycle; **Given** an Unknown payment that the file does not list after the re-queries ended, **When** reconciliation runs, **Then** the payment becomes Failed, the hold is released and the payment becomes Refunded.
- **AC-26.4** **Given** a Completed payment that the settlement file does not list, **When** reconciliation runs, **Then** it is flagged MISSING_AT_BILLER for review and nothing is changed automatically.
- **AC-26.5** **Given** the same settlement file processed twice, **When** reconciliation runs the second time, **Then** the report is identical, no duplicate rows are created and no payment changes state twice.
- **AC-26.6** **Given** a malformed settlement file, **When** reconciliation runs, **Then** the whole file is rejected with an error and no partial result is stored.

### FR-27 - Refund of a completed payment approved by support (Could)

**User story:** As a customer, I want to request a refund of a completed payment that support approves, so that I can get my money back after a mistake.

**Acceptance criteria:**

- **AC-27.1** **Given** a Completed payment of the customer, **When** the customer requests a refund with a reason, **Then** the request is recorded as REFUND_REQUESTED, the payment stays Completed and no money moves.
- **AC-27.2** **Given** a pending refund request, **When** a support employee approves it, **Then** a compensating ledger transaction returns the full total paid (amount plus fee) to the customer's available balance, reverses the agent commission if one was paid, an audit record is written, and the payment stays Completed.
- **AC-27.3** **Given** a pending refund request, **When** a support employee rejects it with a reason, **Then** no money moves and the customer is notified.
- **AC-27.4** **Given** a payment that is not Completed (for example Unknown, Failed or Rejected), **When** the customer requests a refund, **Then** the request is refused with REFUND_NOT_ALLOWED.
- **AC-27.5** **Given** a payment with an approved refund, **When** another refund is requested for it, **Then** the request is refused.
- **AC-27.6** **Given** a customer who is not the owner of the payment, **When** the customer requests a refund for it, **Then** the request is refused with 403.

### FR-28 - Admin views daily totals (Could)

**User story:** As an admin, I want to view daily totals of payments, fees earned, commissions paid and failures per biller, so that I can monitor the business.

**Acceptance criteria:**

- **AC-28.1** **Given** a day D with payments, **When** the admin requests the totals for D, **Then** the response shows the number and total amount of Completed payments, the total fees earned, the total commissions paid, the number of Failed payments per biller (including those later Refunded) and the number of payments in Unknown.
- **AC-28.2** **Given** a biller that had no payments on day D, **When** the admin requests the totals, **Then** the biller appears with zero values.
- **AC-28.3** **Given** the totals for day D, **When** they are compared with the ledger, **Then** the fees earned equal the sum of the fee income ledger credits of day D and the difference is 0.00.
- **AC-28.4** **Given** a day with no payments, **When** the admin requests the totals, **Then** all values are 0.
- **AC-28.5** **Given** a user who is not an admin, **When** the user requests the totals, **Then** the request is refused with 403.

---

## 5. Non-Functional Requirements

Six measurable requirements taken from the quality table of section 3.4 of the brief. The tests or reports must show the number.

### NFR-1 - Correctness (Must)

- **Requirement:** Money is always handled exactly: no double charges and no negative balances.
- **Target:** 0 double charges and 0 negative balances in the concurrency and duplicate tests.
- **Measurement:** An automated test sends 100 simultaneous requests with the same idempotency key (expect exactly 1 payment and 1 charge) and 50 simultaneous payments against a balance that covers only one (expect exactly 1 success and 49 Rejected). After every test an invariant query checks that no wallet has an available or held balance below 0.00 and that the ledger sums to 0.00.
- **Related:** FR-5, FR-10, FR-12, Hard cases 1 and 4.

### NFR-2 - Performance (Should)

- **Requirement:** The system handles the busiest day of the month.
- **Target:** At least 100 payments per second sustained for 60 seconds (6,000 payments) against the mock billers, with 0 unexpected errors (HTTP 5xx).
- **Measurement:** A load script (for example k6 or Locust) runs the scenario in the pipeline or locally and the report shows throughput, error rate and p95 latency. The p95 latency is reported but has no fixed threshold in Phase 1.
- **Related:** FR-9, FR-10, FR-12.

### NFR-3 - Resilience (Must)

- **Requirement:** An outage of one biller does not stop payments to other billers.
- **Target:** While the AquaCo mock is down or answering after the 10-second timeout, 100% of ElectroCo and TopUpNet test payments still complete normally and the health endpoint stays OK.
- **Measurement:** An integration test starts the stack, switches the AquaCo mock to fail on purpose, runs 100 ElectroCo and TopUpNet payments in parallel with AquaCo payments, and checks the success count and that the AquaCo payments end as Unknown or Failed with their funds held or released correctly.
- **Related:** FR-13, FR-15, Hard case 2.

### NFR-4 - Modifiability (Must)

- **Requirement:** Adding a biller touches only the Biller Integration module and configuration.
- **Target:** 0 files changed outside the Biller Integration package and configuration when a new biller (and, in Phase 2, the ElectroCo v2 change) is added.
- **Measurement:** The pull request diff is checked with a script that lists changed paths and fails if any path outside the Biller Integration package, configuration and tests of the new biller is modified. The change-request report lists the exact files changed.
- **Related:** FR-15, FR-6, Hard case 8, ElectroCo API v2 change request.

### NFR-5 - Auditability (Must)

- **Requirement:** Every money movement and admin action can be traced to who, when and why.
- **Target:** 100% of ledger transactions and admin actions have an audit record with actor, UTC timestamp and reason.
- **Measurement:** After every integration test a consistency query counts ledger transactions and admin actions without a matching audit record; the result must be 0. The query also runs on the load-test database.
- **Related:** FR-5, FR-25.

### NFR-6 - Testability (Must)

- **Requirement:** Domain logic is covered by automated tests.
- **Target:** At least 70% line coverage on domain logic, measured in CI on every push.
- **Measurement:** The GitHub Actions workflow runs the tests with coverage (pytest-cov) and fails the build when domain-logic coverage is below 70%. The coverage report is attached to the workflow run.
- **Related:** All Must stories; one automated test for each of the 8 hard cases.

---

## 6. Ambiguities Found in the Brief and Our Decisions

### Ambiguity 1 - Refund of a completed payment versus the payment life cycle

- **Ambiguity:** Section 2.5 says Completed is final and that Refunded is reached only from Failed. FR-27 says a customer can request a refund of a completed payment.
- **Why it matters:** If a Completed payment moved to Refunded, the life cycle (and the State pattern built on it) would contain a transition that section 2.5 forbids.
- **Decision:** The payment stays Completed. A refund of a completed payment is a separate approved request recorded as payment events (REFUND_REQUESTED, REFUND_APPROVED, REFUND_REJECTED) with a compensating ledger transaction that returns the total paid (amount plus fee) and reverses any agent commission. The state Refunded is used only for the automatic return of a held amount after a Failed payment.
- **Affects:** FR-11, FR-12, FR-27, state diagram, `PaymentEvents` and `LedgerEntries` tables.

### Ambiguity 2 - Definition of the daily limit and the velocity rule

- **Ambiguity:** The brief does not say which payments count toward the daily limit, whether the fee counts, when a day starts, or what counts as a payment for the velocity rule.
- **Why it matters:** Different choices give different results for the same hard cases and tests.
- **Decision:** The daily total is the sum of bill amounts (not fees) of payments in FundsHeld, SentToBiller, Unknown and Completed; Rejected, Failed and Refunded payments are not counted. A day is the calendar day in Africa/Cairo. A payment is allowed when the daily total including the new payment is less than or equal to the limit. The velocity rule counts every new payment request (not idempotent replays) that reaches the fraud checks within a sliding 60-second window; the 6th request is rejected and the account is blocked for 10 minutes. Single-payment minimum and maximum apply to the bill amount.
- **Affects:** FR-2, FR-18, FR-19, `Users.blocked_until`, `SystemConfig` key `business.timezone`.

### Ambiguity 3 - Whose wallet pays in an agent-assisted payment

- **Ambiguity:** FR-17 says an agent pays on behalf of a customer and earns commission, and FR-4 mentions agent cash, but the brief does not say whose wallet is debited.
- **Why it matters:** It decides which limits apply, whose balance is held and where the ledger entries go.
- **Decision:** The customer's wallet is always debited and the customer's limits and fraud checks apply. The customer confirms with a simulated one-time code. A customer who gives cash to the agent first receives a cash top-up (FR-4) and then the agent submits the payment. The agent's id is stored on the payment and the commission (40 percent of the fee) is credited to the agent's wallet only when the payment completes.
- **Affects:** FR-4, FR-17, `Payments.agent_id`, `Wallets` (agent wallet).

### Ambiguity 4 - Time rules for Unknown payments and asynchronous billers

- **Ambiguity:** Section 2.6 says an Unknown payment is re-queried "after 1, 5 and 15 minutes" without saying from when. TopUpNet answers "accepted" and calls back "minutes later", but the life cycle has no pending state and no maximum waiting time. It is also not stated when reconciliation may conclude that "no payment" exists.
- **Why it matters:** Without clear clocks, retries, callbacks and reconciliation could refund a payment that the biller actually received.
- **Decision:** The three re-queries run at 1, 5 and 15 minutes after the payment entered Unknown (stored in `Payments.unknown_since`). A TopUpNet payment stays in SentToBiller after "accepted"; if no callback arrives within 30 minutes (configuration `biller.async_callback_timeout_minutes`) it becomes Unknown and the same re-query schedule applies. Reconciliation turns an Unknown payment into Failed only when the settlement file for its date has been processed and does not list the payment, and only then is the hold released. Until that point the money stays held.
- **Affects:** FR-13, FR-14, FR-26, `Payments.unknown_since`, `SystemConfig`.

### Ambiguity 5 - What an inquiry means for each biller, and partial payments

- **Ambiguity:** ElectroCo and AquaCo have bills, but TopUpNet (mobile top-up) has no bill and NetCo has no live inquiry. Also, AquaCo "rejects payments above the bill amount", but the brief does not say whether paying less than the bill is allowed.
- **Why it matters:** FR-8 requires an inquiry that shows amount, fee and total for every biller, and the amount decides the fee and the limits.
- **Decision:** For TopUpNet the "inquiry" is a quote: the customer chooses the amount (within 5.00 to 10,000.00 EGP) and the system returns fee and total without calling the biller. For ElectroCo the customer pays the full amount of the bill. For AquaCo the customer may pay an amount up to the bill amount; PayLink rejects a higher amount before calling AquaCo. NetCo is optional and not in the first release.
- **Affects:** FR-7, FR-8, FR-15, `BillerServices.required_fields`, Biller Integration adapters.
