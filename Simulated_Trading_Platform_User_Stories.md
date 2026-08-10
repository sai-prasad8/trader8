# Simulated Trading Platform — User Stories

Structured against the finalized 13-epic list. Security is folded into Epic 1.

---

# EPIC 1 — User & Authentication (incl. Security)

### US-1.1 — Register an account
As a new user, I want to register with a username/email and password, so that I can start trading with a personal portfolio.
- Registration creates one user and one portfolio, atomically.
- New portfolio starts at `cash_balance = $1,000,000`, `cash_held = $0`, no holdings.
- Duplicate username/email is rejected with a clear error.
- Password is hashed before storage — never stored plaintext.

### US-1.2 — Log in
As a registered trader, I want to log in, so that I can access my trading environment.
- Valid credentials return an authenticated session/token.
- Invalid credentials are rejected without revealing whether the username or password was wrong.

### US-1.3 — Maintain an authenticated session
As a trader, I want my session to stay valid while I'm active and expire appropriately, so that my account stays secure without constant re-login.
- Session/token has a defined expiry and refresh behavior.
- Expired/invalid tokens are rejected on every protected endpoint.

### US-1.4 — Access only my own trading environment
As a trader, I want every API call to verify I own the resource I'm requesting, so that another user can never see or modify my portfolio or orders.
- Every endpoint scopes data access to the authenticated user's ID.
- Requesting another user's portfolio/order returns 403/404, not the data.
- Covered by an automated cross-user authorization test suite.

### US-1.5 — Secure credential storage
As a trader, I want my password protected with a strong hashing algorithm, so that a data breach doesn't expose my credentials.
- Uses a modern, salted hashing algorithm (e.g. bcrypt/argon2), never reversible encryption or plaintext.

---

# EPIC 2 — Portfolio & Account Management

### US-2.1 — View cash position
As a trader, I want to see my total cash, held cash, and available cash, so that I know what I can actually spend.
- `Available Cash = Cash Balance − Cash Held`, always accurate against current holds.

### US-2.2 — View holdings and available shares
As a trader, I want to see owned shares, held shares, and available shares per ticker, so that I know what I can actually sell.
- `Available Shares = Owned Shares − Held Shares`, per holding.

### US-2.3 — View portfolio value
As a trader, I want to see my total portfolio value (cash + market value of holdings), so that I understand my overall position.
- Market value per holding uses live price from the Pricing API — never a locally cached price.

### US-2.4 — Track cost basis per holding
As a trader, I want the system to track my average cost per share, so that P&L can be calculated accurately.
- Holdings store `quantity` and `average_cost` (weighted average across BUYs).
- A SELL reduces quantity without altering the remaining shares' average cost.

### US-2.5 — View portfolio-level P&L
As a trader, I want to see both realized and unrealized profit/loss, so that I can evaluate how I'm doing.
- `Unrealized P&L = Market Value − (Quantity × Average Cost)`.
- `Realized P&L = Sale Proceeds − Cost Basis of Sold Shares`, computed at each SELL fulfillment.
- Total P&L = Realized + Unrealized, shown at the portfolio level.

---

# EPIC 3 — Order Creation & Validation

### US-3.1 — Select a stock and order side
As a trader, I want to choose a stock and BUY or SELL, so that I can begin placing a trade.
- Ticker must exist in the supported universe (validated against `GET /tickers`).

### US-3.2 — Enter quantity and limit price
As a trader, I want to specify quantity and limit price, so that I control the terms of my order.
- Quantity must be a positive integer (fractional shares rejected).
- Limit price must be positive.

### US-3.3 — Submit an order and receive validation feedback
As a trader, I want immediate, clear feedback if my order is invalid, so that I know why it was rejected.
- Validation runs before any hold or order record is created: ticker, side, quantity, price, cash/share sufficiency.
- Rejections return a specific reason (e.g. "insufficient cash," "invalid ticker") rather than a generic error.

### US-3.4 — Server-side funds and share validation
As the system, I want to independently validate cash/share sufficiency server-side, so that the frontend can never bypass business rules.
- Required cash for BUY = quantity × limit price; rejected if greater than available cash.
- SELL rejected if quantity exceeds available shares.

### US-3.5 — Idempotent order submission
As a trader, I want a retried or duplicated submit request to not create a second order, so that a network hiccup doesn't cause me to trade twice.
- `POST /orders` accepts an idempotency key; same key returns the original order rather than creating a new one.

---

# EPIC 4 — Order Reservation & Holds

### US-4.1 — Create a cash hold on BUY
As the system, I want to place a cash hold at the moment a BUY order is accepted, so that the reserved cash can't be spent by another order.
- Hold amount = quantity × limit price.
- Hold and order are created atomically — never one without the other.

### US-4.2 — Create a share hold on SELL
As the system, I want to place a share hold at the moment a SELL order is accepted, so that the reserved shares can't be sold twice.
- Hold amount = order quantity for that ticker.

### US-4.3 — Prevent double-spending under concurrency
As the system, I want concurrent order requests against the same portfolio to be evaluated safely, so that two orders can never together reserve more than what's available.
- Resource check + hold creation is atomic (e.g. row-level locking or equivalent).
- Test: two simultaneous BUYs that together exceed available cash — exactly one succeeds, the other is cleanly rejected.

### US-4.4 — Release a hold exactly once
As the system, I want hold release to be idempotent, so that a duplicated result event can never release (or double-release) the same hold twice.
- `ACTIVE → RELEASED` is an atomic transition.
- A repeated release call on an already-released hold is a safe no-op, logged but not erroring.

### US-4.5 — Maintain hold invariants under all order outcomes
As the system, I want holds to behave correctly whether an order is FULFILLED, FAILED, or CANCELLED, so that the core invariants are never violated.
- No negative available cash or available shares, ever.
- A FAILED or CANCELLED order releases its hold without altering actual cash/holdings.

---

# EPIC 5 — Asynchronous Order Execution

### US-5.1 — Publish order-placed event reliably
As the system, I want the order-placed event to reach Kafka reliably even if Kafka is briefly unavailable, so that an accepted order is never silently lost.
- Order, hold, and outbox event are written in one DB transaction; a background publisher delivers the event to Kafka with retry.

### US-5.2 — Consume order-placed events
As the Trade Executor, I want to consume order-placed events from Kafka, so that I can begin submitting orders to the broker.
- Consumer is durable — a restart resumes from the last committed offset without dropping events.

### US-5.3 — Submit order to broker exactly once
As the Trade Executor, I want to submit each application order to the broker exactly once, even if its event is redelivered, so that duplicate broker orders are never created.
- Executor checks for an existing `broker_order_id` before submitting; a redelivered event with one already recorded is a no-op.
- `application_order_id → exactly one broker_order_id` is enforced.

### US-5.4 — Apply the fixed 180-second execution window
As the system, I want every order to use a 180-second time window, so that broker behavior is consistent and the trader doesn't need to configure it.
- `timeWindowSeconds = 180` is applied server-side on every order, not client-configurable.

### US-5.5 — Poll broker order status
As the Trade Executor, I want to poll the broker for status until resolution, so that I can detect fulfillment or failure.
- Polling interval, retry policy, and stop condition are explicitly defined and documented.

### US-5.6 — Retry on temporary broker failure
As the Trade Executor, I want to retry broker calls on transient failures without changing order status, so that infrastructure hiccups don't become business failures.
- Broker timeout/5xx triggers retry with backoff; order remains PENDING throughout.

### US-5.7 — Recover in-flight orders after executor restart
As the Trade Executor, I want to recover all in-flight orders after a restart or crash, so that no order is permanently orphaned.
- On startup, executor reloads all local orders with an active broker submission and resumes polling/tracking.

---

# EPIC 6 — Order Resolution & Portfolio Updates

### US-6.1 — Handle a FULFILLED result
As the system, when the broker reports FULFILLED, I want to update cash and holdings correctly, so that the portfolio reflects the executed trade.
- BUY: cash decreases by quantity × fill price; shares increase by quantity.
- SELL: shares decrease by quantity; cash increases by quantity × fill price.
- Uses the broker-provided fill price as authoritative (not assumed equal to limit price).
- Any unused cash reservation (limit price vs. fill price difference) is released back to available cash.

### US-6.2 — Handle a FAILED result
As the system, when the broker reports FAILED, I want to release the hold without altering actual cash or holdings, so that a failed trade has zero side effects.
- Order marked FAILED; failure reason surfaced to the trader if the broker provides one.

### US-6.3 — Handle a CANCELLED result
As the system, when a cancellation is confirmed, I want to release the hold and mark the order CANCELLED, so that reserved resources become available again.

### US-6.4 — Publish and consume order-result events reliably
As the system, I want the order-result event to reach both the Trade API and Portfolio API reliably, so that order status and portfolio state update together.
- Same outbox/idempotency guarantees as order-placed events (Epic 5) apply here.

### US-6.5 — Idempotent result processing
As the system, I want a duplicated result event to update state at most once, so that the portfolio can't be double-credited or double-debited.
- Result processing checks current order/hold status before applying changes; a result for an already-resolved order is a no-op.

---

# EPIC 7 — Order Cancellation

### US-7.1 — Request cancellation of a pending order
As a trader, I want to cancel an order while it's still pending, so that I can back out before it executes.
- Cancellation is only accepted while local status is PENDING.
- On request, local status moves to `CANCEL_REQUESTED` immediately, before the broker confirms anything.

### US-7.2 — See cancellation as pending
As a trader, I want to see that my cancellation request is in progress, so that I understand my order hasn't definitively resolved yet.
- UI shows a distinct `CANCEL_REQUESTED` state, separate from PENDING and from a final CANCELLED.

### US-7.3 — Resolve the cancel-vs-fulfillment race deterministically
As the system, I want a clear, consistent outcome when cancellation and broker fulfillment happen at nearly the same time, so that the final state is never ambiguous.
- If the broker fulfills before processing the cancellation, the order resolves FULFILLED — cancellation does not override an already-executed trade.
- If the broker confirms cancellation first, the order resolves CANCELLED.
- Exactly one of these outcomes occurs; the hold is released exactly once regardless of which one wins.

### US-7.4 — Reject cancellation of an already-resolved order
As a trader, I want a clear error if I try to cancel an order that already resolved, so that I'm not confused by a cancel action that can't do anything.
- Attempting to cancel a FULFILLED/FAILED/CANCELLED order returns a specific "already resolved" error.

### US-7.5 — See the final cancellation result
As a trader, I want to see the definitive outcome (CANCELLED or FULFILLED) once resolved, so that I know what actually happened to my order.

---

# EPIC 8 — Market Data

### US-8.1 — Browse the supported stock universe
As a trader, I want to see the list of supported tickers with current prices, so that I can find something to trade.
- Sourced live from `GET /tickers`; never persisted as a local price database.

### US-8.2 — View current simulated price for a stock
As a trader, I want to see a stock's current price, so that I can decide on a limit price.
- Price is fetched live from `GET /prices/{ticker}/live` on each request (jitter expected and accepted).

### US-8.3 — View historical price data
As a trader, I want to see a stock's price history, so that I can understand its recent trend.
- Sourced from `GET /prices/{ticker}/history`; used both for charting and as input to the Technical Advisor.

### US-8.4 — See when price data was last updated
As a trader, I want to know how fresh the displayed price is, so that I don't mistake it for real-time exchange data.
- UI shows a timestamp or "simulated price" indicator alongside quoted prices.

### US-8.5 — Handle pricing API unavailability
As a trader, I want a clear message if live pricing is temporarily unavailable, so that I'm never shown stale or fabricated prices as if they were current.
- A pricing API failure is surfaced as an explicit infrastructure error, not silently swapped for cached or invented data.

---

# EPIC 9 — Technical Trading Advisor (Deterministic)

### US-9.1 — Retrieve market data for analysis
As the Advisor, I want to pull current and historical prices from the Pricing API, so that my analysis is based on real, current data.

### US-9.2 — Calculate Bollinger Band indicators
As the Advisor, I want to calculate the 20-day moving average, standard deviation, and upper/lower bands, so that recommendations rest on transparent, reproducible math.
- Period and multiplier are fixed and documented (20 days, ×2 std dev).

### US-9.3 — Produce a BUY/HOLD/SELL recommendation
As a trader, I want a recommendation for any stock regardless of whether I own it, so that I can evaluate stocks I don't currently hold.
- `Price ≤ Lower Band → BUY`; `Price ≥ Upper Band → SELL`; otherwise `HOLD`.
- Same ticker → same recommendation for any user, independent of their portfolio (base signal only).

### US-9.4 — Explain the technical signal without a fabricated confidence number
As a trader, I want to understand why a recommendation was made, without seeing an unjustified percentage, so that I can trust what I'm looking at.
- No raw arbitrary "Confidence: X%".
- A defined, documented signal-strength measure is shown instead (e.g. distance from band expressed as Strong/Moderate/Weak).

### US-9.5 — Display the relevant indicators
As a trader, I want to see the actual moving average, bands, and current price alongside the recommendation, so that I can verify the logic myself.

---

# EPIC 10 — Portfolio-Aware AI Recommendation Engine

### US-10.1 — Advisor reads portfolio data (read-only)
As the system, I want the Advisor to read a trader's holdings, concentration, and cash from the Portfolio API, so that it can personalize its output.
- Access is strictly read-only — no write path exists from the Advisor to the Portfolio API. Enforced at the API/permissions level.

### US-10.2 — Personalized recommendation adjustment
As a trader, I want my recommendation to account for my existing position in that stock, so that the advice reflects my actual situation, not just the raw market signal.
- Personalization inputs are limited to: quantity held, % portfolio concentration in that ticker, available cash.
- Adjustment logic is deterministic and documented (e.g. downgrading BUY to HOLD when a position is already heavily concentrated) — not opaque model judgment.
- The base (non-personalized) signal from Epic 9 remains visible for comparison.

### US-10.3 — Portfolio commentary alongside the recommendation
As a trader, I want commentary on how a potential trade would affect my overall portfolio (e.g. concentration risk), so that I see the bigger picture, not just a single-stock signal.
- Commentary is generated only from pre-computed real facts (concentration %, cash impact) — the AI does not calculate these facts itself.

### US-10.4 — AI cannot invent portfolio or market data
As a trader, I want every number I see from the AI to be traceable to a real value, so that I'm never misled by a hallucinated figure.
- The AI pipeline only ever receives pre-computed structured facts; it has no free-form path to generate numbers of its own.
- Automated test verifies all numeric claims in AI output trace back to a value supplied in the prompt.

### US-10.5 — AI never places, modifies, or cancels orders
As a trader, I want the AI to only advise, never act, so that I keep full control over my trades.
- No code path exists from the AI/Advisor layer to the Trade API's write endpoints.
- Verified by an architecture-level check, not just documentation.

---

# EPIC 11 — AI Explanation Layer

### US-11.1 — Natural-language explanation of the technical signal
As a trader, I want a plain-English explanation of why a stock got its recommendation, so that I don't have to interpret raw indicator numbers myself.
- AI receives only structured facts (price, moving average, bands, recommendation) and generates explanatory text; numeric values in the text must match those facts exactly.

### US-11.2 — Explanation degrades gracefully if AI is unavailable
As a trader, I want to still see the recommendation and raw indicators if the AI explanation service is down, so that a third-party outage never blocks my trading decisions.
- The structured analysis (Epics 9/10) renders fully independent of AI layer availability, with a clear fallback state for the narrative section.

### US-11.3 — Distinguish calculated facts from AI-generated text
As a trader, I want to visually tell which numbers are calculated and which text is AI-generated explanation, so that I know what to verify versus what to take as narrative.

---

# EPIC 12 — Trading Platform UI

### US-12.1 — Dashboard overview
As a trader, I want a dashboard showing portfolio value, available cash, holdings, and pending orders at a glance, so that I get an immediate sense of my account.

### US-12.2 — Portfolio and holdings view
As a trader, I want a dedicated portfolio page with per-holding detail (quantity, price, market value, P&L), so that I can review my positions in depth.

### US-12.3 — Stock detail page with chart
As a trader, I want a stock detail page combining price, history chart, technical analysis, and AI commentary, so that I can move naturally from research to a decision.

### US-12.4 — Order ticket
As a trader, I want a simple order entry form (side, quantity, limit price) with real-time validation feedback, so that placing a trade is fast and error-free.

### US-12.5 — Pending orders and order history
As a trader, I want to see pending orders separately from historical orders, with a cancel action only where applicable, so that I can manage active trades easily.

### US-12.6 — Order status updates without manual refresh
As a trader, I want an order's status to update from PENDING to its final state without me refreshing the page, so that I always see current information.

### US-12.7 — Advisor panel
As a trader, I want the technical analysis, recommendation, and AI explanation clearly laid out on the stock detail page, so that analysis is a visible part of the app rather than hidden in a chatbot.

### US-12.8 — Clear cash/share breakdowns everywhere
As a trader, I want available vs. held vs. total cash and shares clearly distinguished on every relevant screen, so that I never misjudge what I can actually trade.

---

# EPIC 13 — Reliability, Observability & Recovery

### US-13.1 — Idempotency across all critical operations
As the system, I want order submission, broker submission, and result processing to all be idempotent, so that retries and duplicate deliveries never corrupt state.
- Consolidates the idempotency acceptance criteria already defined in Epics 3, 4, 5, and 6 into a cross-cutting test suite.

### US-13.2 — Correlation IDs across the full order journey
As an engineer, I want a single correlation ID to trace an order from application ID through Kafka event ID to broker order ID, so that I can answer "what happened to order X?" without inspecting multiple services manually.

### US-13.3 — Structured logging of key lifecycle events
As an engineer, I want order placement, validation failures, hold creation/release, Kafka events, broker calls, and portfolio updates all logged in a structured, queryable format.

### US-13.4 — Graceful handling of Kafka failures
As the system, I want Kafka publish failures to be retried via the outbox pattern rather than silently dropping events, so that no order is lost due to a messaging outage.

### US-13.5 — Graceful handling of database failures
As the system, I want database unavailability to fail requests safely (no partial writes) rather than leaving orders or holds in an inconsistent state.

### US-13.6 — Graceful handling of broker failures
As the system, I want broker unavailability to trigger retries without changing order status, so that infrastructure failure is never mistaken for business failure.

### US-13.7 — Reconciliation job
As the system, I want a periodic job comparing local in-flight orders against actual broker state, so that a crashed component doesn't leave orders permanently stuck.
- Repairs discrepancies (status, holds) and logs every correction made.

### US-13.8 — Executor restart recovery
As the system, I want the Trade Executor to resume tracking all in-flight orders after a restart, so that no order is orphaned by a deployment or crash.

### US-13.9 — Concurrency and duplicate-event test coverage
As an engineer, I want automated tests specifically covering concurrent orders, duplicate Kafka events, and cancellation races, so that these failure modes are verified continuously, not just handled by design intent.
