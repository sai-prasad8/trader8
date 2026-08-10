# Simulated Trading Platform
## Project Requirements, Architecture & Functional Design Document

**Document purpose:**  
This document is the definitive functional and technical design for the simulated trading platform project. It is intended to be used as the primary development reference by all team members.

The system is a **mock trading platform**. It does not connect to a real brokerage, execute real trades, or use real money. Market prices and trade execution are provided by two externally supplied simulation services:

1. **Simulated Live Pricing API**
2. **Simulated Trade Broker API**

Our application is responsible for user management, portfolio management, order management, risk/hold management, event-driven order execution, trade advice, and the web interface.

---

# 1. Project Objective

The objective is to build a web-based trading platform that behaves similarly to a simplified real trading application.

A registered trader should be able to:

- create an account and log in;
- start with a virtual balance of **$1,000,000 USD**;
- view the current simulated price of supported stocks;
- view their cash balance and stock holdings;
- place BUY and SELL **limit orders**;
- see pending and historical orders;
- cancel pending orders;
- have the system prevent overspending of cash or shares;
- receive stock analysis and a BUY/HOLD/SELL recommendation;
- view technical analysis associated with the recommendation;
- view their portfolio's current market value and performance.

The application must maintain correct portfolio state even when several orders are pending simultaneously.

The system must use **Kafka as the event bus** between the trading components. Kafka is a core part of the architecture and is not optional.

---

# 2. Important External Services

Two services are provided externally and are **not part of the application that we are building**.

## 2.1 Simulated Live Pricing API

The pricing API provides a fixed universe of approximately 100 US-listed stocks. The supported ticker list is maintained by that service.

Relevant endpoints include:

```text
GET /tickers

GET /prices/{ticker}/live

GET /prices/{ticker}/history?date=YYYY-MM-DD

GET /prices/{ticker}/history?days=N
```

The `/live` endpoint returns the latest simulated price. The service uses cached historical prices and adds small simulated intraday jitter, so repeated calls can return slightly different values.

The API also provides historical prices, which will be used by the Trade Advisor.

### Important constraint

**Our application must not maintain its own market-price database.**

When the application needs a current or historical stock price, it obtains the information from the Live Pricing API.

The pricing API itself does not provide authentication and does not provide trading advice. Analytics/trading advice is explicitly outside its scope.

---

# 3. Simulated Trade Broker API

The Broker Simulator is responsible for simulating the outcome of submitted orders.

It does **not** manage:

- users;
- portfolios;
- cash;
- holdings;
- funds/shares validation.

Those responsibilities belong to our application.

The Broker Simulator supports only limit orders.

Its relevant endpoints are:

```text
POST /orders

GET /orders/{orderId}

DELETE /orders/{orderId}
```

or the equivalent cancel endpoint supplied by the service.

---

# 4. High-Level Architecture

The system consists of the following components:

```text
                         ┌──────────────────────────┐
                         │       React Web UI        │
                         └────────────┬─────────────┘
                                      │
                 ┌────────────────────┼────────────────────┐
                 │                    │                    │
                 ▼                    ▼                    ▼
        ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
        │  Authentication │  │   Trade REST    │  │ Portfolio REST  │
        │ / User Services │  │      API        │  │      API        │
        └─────────────────┘  └────────┬────────┘  └────────┬────────┘
                                      │                    │
                                      │                    │
                                      ▼                    ▼
                              ┌────────────────────────────────┐
                              │          PostgreSQL             │
                              │ Users / Orders / Portfolio /   │
                              │ Holdings / Holds               │
                              └────────────────────────────────┘

                                      │
                                      │ Kafka
                                      ▼
                              ┌─────────────────┐
                              │ Trade Executor  │
                              └────────┬────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ Broker Simulator│
                              └─────────────────┘


        ┌─────────────────┐
        │  Trade Advisor  │
        └────────┬────────┘
                 │
                 ▼
        ┌─────────────────┐
        │ Live Pricing API│
        └─────────────────┘
```

The React application is the primary interface for the trader.

The Trade REST API manages the order lifecycle.

The Portfolio REST API owns cash, holdings, and financial holds.

The Trade Executor handles asynchronous interaction with the Broker Simulator.

Kafka connects the asynchronous trading components.

PostgreSQL is the persistent system of record.

The Trade Advisor is responsible for analysing stocks independently of the trader's portfolio.

---

# 5. Component Responsibilities

## 5.1 React Web UI

The React frontend provides the trader-facing application.

It should provide a UI comparable to a simplified real trading platform.

At minimum it should contain:

- registration;
- login;
- dashboard;
- portfolio view;
- stock/market view;
- stock detail view;
- technical analysis;
- order placement;
- pending orders;
- order history;
- account information.

The UI should clearly distinguish:

- available cash;
- held cash;
- total cash;
- owned shares;
- held shares;
- available shares;
- portfolio market value;
- profit/loss;
- pending orders;
- completed orders.

The UI should update order status as orders progress from PENDING to FULFILLED, FAILED, or CANCELLED.

---

# 6. User Management

## 6.1 Registration

A new user must be able to register.

The registration process creates:

1. a user account;
2. a portfolio associated with that user.

Every newly created portfolio must start with:

```text
Initial Cash = $1,000,000 USD
Initial Holdings = none
Initial Cash Holds = $0
```

The user does not receive any shares automatically.

## 6.2 Login

A registered user must be able to log in and access only their own:

- portfolio;
- holdings;
- cash;
- orders;
- order history.

Authentication technology is intentionally left open.

A simple username/password authentication mechanism is sufficient. JWT may be used if convenient, but JWT is **not a mandatory requirement**.

Passwords must not be stored as plaintext.

---

# 7. Portfolio Management

Each user has exactly one logical trading portfolio.

The portfolio maintains:

```text
Cash Balance
Cash Held
Holdings
Share Holds
```

For cash:

```text
Available Cash = Cash Balance - Cash Held
```

For a stock:

```text
Available Shares = Owned Shares - Held Shares
```

The Portfolio API is responsible for enforcing these calculations.

---

# 8. Portfolio Valuation

The portfolio page should resemble a real trading platform.

For each holding, the system should obtain the current simulated price from the Live Pricing API.

For example:

```text
AAPL
Shares:             100
Current Price:      $200
Market Value:       $20,000
```

The portfolio should also display:

- cash;
- total stock market value;
- total portfolio value;
- available cash;
- held cash;
- holdings;
- available/held shares;
- unrealized profit/loss where sufficient information is available.

The application must not maintain a separate market-price store.

---

# 9. Stock/Market View

The trader should be able to browse the supported stock universe.

The application may obtain the list from:

```text
GET /tickers
```

The stock list is approximately 100 US-listed stocks.

For each stock, the UI should show at minimum:

```text
Ticker
Company Name
Current Price
```

Additional metadata such as sector/industry may be displayed if provided by the pricing API.

---

# 10. Stock Detail Page

Clicking a stock should open a detailed stock view.

The page should contain at least:

```text
Stock name
Ticker
Current price
Price/history chart
Technical analysis
BUY/HOLD/SELL recommendation
Explanation
Strategy currently being used
```

The user should be able to initiate a BUY or SELL order from this page.

The technical analysis should be a visible part of the application rather than being hidden behind a chatbot.

---

# 11. Order Types

Only **LIMIT orders** are part of the core project.

Market orders and stop orders are explicitly out of scope.

A limit BUY means:

> The trader is willing to buy the specified number of shares at no more than the specified limit price.

A limit SELL means:

> The trader is willing to sell the specified number of shares at no less than the specified limit price.

---

# 12. Order Placement

The trader should provide:

```text
Ticker
Side: BUY / SELL
Quantity
Limit Price
```

The application will use a **fixed order time window of 180 seconds**.

The trader does not choose the time window.

Therefore, conceptually:

```text
timeWindowSeconds = 180
```

for every submitted order.

---

# 13. Order Validation

The Trade API must validate the order before accepting it.

At minimum:

### Ticker

The ticker must be supported.

### Side

Only:

```text
BUY
SELL
```

are valid.

### Quantity

Quantity must be a positive value.

Zero or negative quantities must be rejected.

### Limit price

Limit price must be positive.

Zero or negative prices must be rejected.

### BUY funds validation

For:

```text
BUY quantity @ limitPrice
```

the required cash is:

```text
quantity × limitPrice
```

The order must be rejected if:

```text
required cash > available cash
```

### SELL holdings validation

For:

```text
SELL quantity
```

the order must be rejected if:

```text
quantity > available shares
```

The application must perform these validations before submitting an order to the Broker Simulator.

---

# 14. Cash Hold Logic

Consider:

```text
Cash Balance = $1,000,000
Cash Held = $0

BUY 100 AAPL @ $200
```

Required hold:

```text
100 × $200 = $20,000
```

After placing the hold:

```text
Cash Balance = $1,000,000
Cash Held = $20,000
Available Cash = $980,000
```

A second order cannot use that $20,000.

For example, if the trader then tries:

```text
BUY 5,000 shares @ $200
```

the second order requires:

```text
$1,000,000
```

but only $980,000 is available.

Therefore the second order must be rejected.

This rule is critical.

---

# 15. Share Hold Logic

Suppose:

```text
AAPL owned = 500
AAPL held = 0
```

The trader submits:

```text
SELL 100 AAPL
```

After the hold:

```text
Owned = 500
Held = 100
Available = 400
```

A second SELL cannot use those 100 held shares.

If the trader attempts:

```text
SELL 450 AAPL
```

the order must be rejected because only 400 shares are available.

---

# 16. Order Lifecycle

The application's order lifecycle is:

```text
                         ┌──────────────┐
                         │    CREATE    │
                         └──────┬───────┘
                                │
                         validation
                                │
                         place hold
                                │
                                ▼
                         ┌──────────────┐
                         │    PENDING   │
                         └──────┬───────┘
                                │
                     ┌──────────┼──────────┐
                     │          │          │
                     ▼          ▼          ▼
                 FULFILLED    FAILED   CANCELLED
```

An order begins as `PENDING` only after validation and the required portfolio hold have succeeded.

---

# 17. Order Placement Flow

The complete order placement process is:

```text
1. User submits order through React UI.

2. React calls Trade REST API.

3. Trade API validates:
   - authenticated user
   - ticker
   - side
   - quantity
   - limit price

4. Trade API asks Portfolio API to place:
   - cash hold for BUY
   - share hold for SELL

5. Portfolio API checks availability.

6. If insufficient resources:
   - reject the order
   - no order is created
   - no hold remains

7. If the hold succeeds:
   - Trade API creates order record
   - status = PENDING

8. Trade API publishes an order-placed Kafka event.

9. Trade Executor consumes the event.

10. Trade Executor submits the order to Broker Simulator.

11. Broker Simulator returns an order ID.

12. Trade Executor tracks the broker order.

13. Eventually the broker resolves the order.

14. Trade Executor publishes an order-result event.

15. Trade API updates the application's order status.

16. Portfolio API releases the hold.

17. If fulfilled:
    - update cash/holdings

18. If failed:
    - do not change actual cash/holdings

19. React displays the final result.
```

---

# 18. Broker Fulfilment Behavior

The Broker Simulator decides whether an order will eventually fulfil.

At order placement, it obtains the current live price from the Live Pricing API.

The documented default fulfilment logic checks:

1. whether the limit price is within 5% of the current live price;
2. whether the quantity is below the configured flat volume threshold.

If both conditions pass:

```text
Order → eventually FULFILLED
```

If either condition fails:

```text
Order → eventually FAILED
```

A failed order is not immediately returned as FAILED by the Broker Simulator. It remains pending until its time window expires.

The Broker Simulator returns a unique broker order ID when an order is submitted.

---

# 19. Important: Our System Must Not Duplicate Broker Logic

The Trade Executor should **not independently decide whether an order should fill**.

The Broker Simulator is authoritative.

Our Trade Executor's responsibility is:

```text
Receive event
      ↓
Submit order to broker
      ↓
Track broker order
      ↓
Retrieve status
      ↓
Report result
```

It must not implement a second competing matching algorithm.

---

# 20. Trade Executor

The Trade Executor is an asynchronous service.

Its primary responsibilities are:

- consume Kafka order-placed events;
- submit orders to Broker Simulator;
- track broker order IDs;
- poll/check broker order status;
- handle broker failures through retry;
- publish final order-result events.

---

# 21. Broker Failure Handling

If the Broker Simulator is temporarily unavailable:

```text
Trade Executor
      ↓
Broker unavailable
      ↓
Retry
      ↓
Order remains PENDING
```

The application must **not automatically mark the order FAILED simply because the broker is temporarily unreachable**.

A `FAILED` status represents the Broker Simulator's actual business outcome.

Infrastructure failures and business failures must be treated differently.

---

# 22. Kafka Architecture

Kafka is a mandatory part of the architecture.

At minimum, the event architecture needs to support:

### Order placed event

```text
Trade API
    ↓
Kafka
    ↓
Trade Executor
```

### Order result event

```text
Trade Executor
    ↓
Kafka
    ├── Trade API
    └── Portfolio API
```

The exact topic names and message formats may be chosen during implementation, but they must be documented and consistent.

Suggested topics:

```text
orders.placed
orders.results
```

Additional topics may be introduced if necessary.

---

# 23. Event Content

An order-placed event should contain enough information for the Trade Executor to submit the order without querying unnecessary application state.

Conceptually:

```json
{
  "orderId": "...",
  "userId": "...",
  "ticker": "AAPL",
  "side": "BUY",
  "quantity": 100,
  "limitPrice": 200.00,
  "timeWindowSeconds": 180
}
```

The application should also include an event ID or another mechanism allowing consumers to safely handle duplicate delivery.

The exact implementation can use a suitable event schema, but the meaning of each field must remain consistent.

---

# 24. Database Ownership

PostgreSQL is the persistent system of record for:

- users;
- portfolios;
- orders;
- holdings;
- cash/share holds.

The exact table decomposition may be decided during implementation, but the data model must preserve the following relationships:

```text
User
  │
  └── Portfolio
        │
        ├── Cash
        ├── Holdings
        └── Holds

User
  │
  └── Orders
```

Every order must belong to a specific user.

Every portfolio must belong to a specific user.

A user must never be able to access another user's portfolio or orders.

---

# 25. Suggested Data Model

A reasonable initial schema is:

## users

```text
id
username/email
password_hash
created_at
```

## portfolios

```text
id
user_id
cash_balance
cash_held
created_at
updated_at
```

## holdings

```text
id
portfolio_id
ticker
quantity
```

## orders

```text
id
user_id
ticker
side
quantity
limit_price
status
broker_order_id
created_at
updated_at
resolved_at
fill_price
failure_reason
```

## holds

A separate hold table may be used to make individual reservations explicit:

```text
id
order_id
portfolio_id
ticker
hold_type
quantity_or_amount
status
created_at
released_at
```

Alternatively, aggregate hold values may be maintained directly in portfolio/holding records.

The final implementation should prioritize **transactional correctness** over minimizing table count.

---

# 26. Transactional Requirement

Hold placement and order acceptance must be designed carefully.

The system must never reach a state where:

```text
Order = PENDING
Hold = missing
```

or:

```text
Hold = active
Order = rejected/nonexistent
```

unless the hold is deliberately being rolled back as part of a failed transaction.

The portfolio and order operations must therefore be implemented with appropriate transactional guarantees.

This is one of the most important correctness requirements in the entire system.

---

# 27. Order Completion

When the Broker Simulator reports `FULFILLED`:

For BUY:

```text
cash decreases by:
quantity × fill price

shares increase by:
quantity
```

Because the Broker Simulator's documented default is to fill at the limit price, the expected fill price will normally equal the submitted limit price.

The corresponding cash hold is released.

For SELL:

```text
shares decrease by:
quantity

cash increases by:
quantity × fill price
```

The corresponding share hold is released.

---

# 28. Failed Order

When the Broker reports:

```text
FAILED
```

the application must:

```text
release hold
do not change actual cash
do not change actual holdings
mark order FAILED
```

The UI should show the failure reason if one is provided.

---

# 29. Cancelled Order

Cancellation is permitted only while the order is still pending.

The flow is:

```text
User requests cancellation
        ↓
Trade API verifies local order is PENDING
        ↓
Request cancellation at Broker Simulator
        ↓
Broker confirms cancellation
        ↓
Release portfolio hold
        ↓
Mark local order CANCELLED
```

If the broker order has already resolved, cancellation must fail.

The Broker API explicitly specifies an error for cancellation of an already resolved or already cancelled order.

---

# 30. Order History

The trader should be able to see both active and historical orders.

The order history should show information such as:

```text
Order ID
Ticker
BUY / SELL
Quantity
Limit Price
Status
Created Time
Resolved Time
Fill Price
```

Pending orders should be visually distinguishable from completed orders.

The user should have a cancel action only for orders that are still cancellable.

---

# 31. Trade Advisor

The Trade Advisor is a separate application/service.

It communicates with the Live Pricing API.

It does **not** need the user's portfolio to generate its base stock recommendation.

The same stock analysis should therefore be possible whether the user owns:

```text
0 shares
```

or:

```text
1,000 shares
```

The initial advisor is based on an **explainable technical-analysis strategy**.

---

# 32. Initial Advisor Strategy

The first implemented strategy will be a **Bollinger Band-based strategy**.

The advisor obtains historical prices from the Live Pricing API.

It calculates:

```text
Moving Average
Standard Deviation
Upper Bollinger Band
Lower Bollinger Band
```

A suitable initial configuration is:

```text
Period = 20 trading days
Standard deviation multiplier = 2
```

The recommendation logic is:

```text
Current Price <= Lower Bollinger Band
        → BUY

Current Price >= Upper Bollinger Band
        → SELL

Otherwise
        → HOLD
```

The exact calculated values must be shown to the user so that the recommendation is explainable.

---

# 33. Advisor Output

The advisor should return structured information rather than only natural-language text.

Conceptually:

```text
Ticker
Current Price
Recommendation
Confidence
Moving Average
Upper Bollinger Band
Lower Bollinger Band
Relevant indicators
Explanation
Strategy name
```

Example:

```text
AAPL

Current Price: $201.43

Recommendation:
BUY

Confidence:
78%

Technical Analysis:

20-Day Moving Average: $208.20
Upper Band:            $221.50
Lower Band:            $195.40

Interpretation:
The current price is below the moving average and
approaching the lower Bollinger Band.

Recommendation rationale:
The selected strategy considers this a potential
buying opportunity.
```

The UI should make the numerical analysis visible.

---

# 34. AI-Assisted Advisor — Future Enhancement

The initial implementation does **not** need to be a chatbot.

The priority is to build the structured technical-analysis feature correctly.

A future enhancement may introduce AI assistance.

The AI could receive structured calculated indicators such as:

```text
current price
moving average
upper band
lower band
price movement
recommendation
```

and generate a more natural explanation.

The AI should **not invent market data**.

The underlying numerical values must come from the pricing API and calculations performed by our system.

The AI should therefore act primarily as an **analysis/explanation layer**, rather than becoming an uncontrolled source of financial data.

---

# 35. Strategy Selection — Future Enhancement

A future version may allow the trader to select different investment strategies.

For example:

```text
Bollinger Band Technical Strategy
Conservative Strategy
Value Strategy
Growth Strategy
Buffett-inspired Strategy
```

The system should not claim to reproduce a real individual's exact investment decisions.

For example, the UI should use:

> "Buffett-inspired strategy"

rather than:

> "This is exactly how Warren Buffett would invest."

The initial MVP should not depend on external fundamental-data sources that have not been provided.

If fundamental information such as:

```text
P/E
earnings
revenue
debt
free cash flow
```

is required for a future strategy, an appropriate data source must first be identified.

---

# 36. Portfolio Independence

The advisor's base analysis is independent of the user's portfolio.

For example:

```text
User A owns 0 AAPL
User B owns 500 AAPL
```

Both users can receive the same technical recommendation for AAPL.

Portfolio-aware advice may be introduced later as an enhancement.

---

# 37. Dashboard

The dashboard should feel like a real but simplified trading platform.

A possible structure:

```text
┌─────────────────────────────────────────────────────┐
│ Trading Platform                 Account / Logout   │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Portfolio Value       Cash Available               │
│ $1,025,400            $850,000                     │
│                                                     │
├─────────────────────────────────────────────────────┤
│ Holdings                                            │
│                                                     │
│ AAPL     100 shares    $201.43    $20,143          │
│ MSFT     50 shares     $512.20    $25,610          │
│                                                     │
├─────────────────────────────────────────────────────┤
│ Watchlist / Market                                  │
│                                                     │
│ AAPL     $201.43    BUY                             │
│ MSFT     $512.20    HOLD                            │
│ TSLA     $321.50    SELL                            │
│                                                     │
├─────────────────────────────────────────────────────┤
│ Pending Orders                                      │
│                                                     │
│ BUY AAPL 100 @ $200      PENDING      CANCEL       │
└─────────────────────────────────────────────────────┘
```

The exact visual design is open, but the application should prioritize usability and clarity.

---

# 38. Stock Analysis Page

The stock detail page should combine market information, analysis, and trading.

A recommended structure is:

```text
AAPL
Apple Inc.

Current Price: $201.43
────────────────────────────

[ Price Chart ]

────────────────────────────

Technical Analysis

20-Day Moving Average: $208.20
Upper Bollinger Band:  $221.50
Lower Bollinger Band:  $195.40

Recommendation

        BUY

Confidence: 78%

Why?

• Current price is below the moving average
• Price is close to the lower Bollinger Band
• The selected strategy identifies this as a potential
  buying condition

────────────────────────────

[ BUY ]    [ SELL ]

Quantity: ______
Limit Price: ______
```

This allows the trader to move naturally from:

```text
Market information
      ↓
Analysis
      ↓
Decision
      ↓
Order
```

---

# 39. No Real-Market Integration

The application must never:

- submit orders to a real broker;
- use real money;
- imply that a trade was executed in a real market;
- depend on real-time exchange connectivity.

The pricing and brokerage behavior are completely simulated.

---

# 40. Error Handling

The application must distinguish between user errors and infrastructure errors.

Examples of user errors:

```text
Invalid ticker
Invalid quantity
Invalid limit price
Insufficient cash
Insufficient shares
Invalid cancellation
Unauthenticated request
```

These should return clear errors to the frontend.

Examples of infrastructure errors:

```text
Pricing API unavailable
Broker API unavailable
Kafka temporarily unavailable
Database unavailable
```

These should be handled without corrupting portfolio state.

In particular:

> A temporary Broker API outage must not automatically turn a pending order into FAILED.

---

# 41. Security

The platform contains user accounts and financial state, even though all money is simulated.

Therefore:

- passwords must be securely hashed;
- authenticated requests must identify the current user;
- users must only access their own portfolios;
- users must only access their own orders;
- APIs must validate ownership before returning or modifying resources.

The exact authentication mechanism remains an implementation decision.

JWT is acceptable but not mandatory.

---

# 42. Non-Functional Requirements

The system should be designed as a set of maintainable services rather than a single large application.

The externally supplied pricing and broker services are designed for at least approximately 100 simultaneous connections on modest resources.

Our services should likewise be lightweight and suitable for the training/development environment.

Logging should be implemented for important events, especially:

- order placement;
- validation failures;
- hold creation;
- hold release;
- Kafka events;
- broker submission;
- broker polling;
- broker failures;
- order resolution;
- portfolio updates;
- advisor requests/errors.

---

# 43. Testing Requirements

The application should include automated tests.

At minimum:

## Authentication

- successful registration;
- duplicate registration rejection;
- successful login;
- invalid credentials;
- unauthorized resource access.

## Portfolio

- new portfolio starts at $1,000,000;
- holdings initially empty;
- cash hold calculation;
- share hold calculation;
- hold release;
- successful BUY update;
- successful SELL update.

## Orders

- valid BUY;
- valid SELL;
- invalid quantity;
- invalid price;
- invalid ticker;
- insufficient cash;
- insufficient shares;
- pending order;
- successful fulfilment;
- failed order;
- cancellation;
- cancellation of already resolved order.

## Risk/hold behavior

Explicitly test:

```text
Order A holds cash
       ↓
Order B cannot spend that cash
```

and:

```text
Order A holds shares
       ↓
Order B cannot sell those shares
```

This test is particularly important because it represents one of the core business requirements.

## Kafka

Test that:

```text
Order placed
    ↓
Kafka
    ↓
Trade Executor
```

and:

```text
Broker result
    ↓
Kafka
    ↓
Trade API + Portfolio API
```

work correctly.

## Trade Advisor

Test:

- historical price retrieval;
- Bollinger calculation;
- BUY condition;
- SELL condition;
- HOLD condition;
- explanation generation;
- invalid ticker/API failure.

---

# 44. Observability

The application should make it possible to trace an order through the system.

A useful correlation structure is:

```text
User
 ↓
Application Order ID
 ↓
Kafka Event ID
 ↓
Broker Order ID
```

Logs should make it possible to answer:

> What happened to order X?

without manually inspecting multiple services.

---

# 45. MVP Scope

The following features are **mandatory for the first complete version**.

### User Management

- Registration
- Login
- Per-user authentication
- Per-user portfolio

### Portfolio

- $1,000,000 starting cash
- Holdings
- Available cash
- Held cash
- Available shares
- Held shares
- Current portfolio valuation
- P&L

### Market

- Supported stock list
- Current simulated prices
- Stock detail page
- Historical price chart/data

### Trading

- BUY limit orders
- SELL limit orders
- 180-second order duration
- Order validation
- Cash holds
- Share holds
- Pending orders
- Order cancellation
- Order history

### Event-driven architecture

- Kafka
- Trade REST API
- Portfolio REST API
- Trade Executor
- PostgreSQL

### External integration

- Live Pricing API
- Broker Simulator API

### Advisor

- Bollinger Band strategy
- BUY/HOLD/SELL recommendation
- Technical indicators
- Explainable recommendation
- Structured analysis UI

### Frontend

- Trading dashboard
- Portfolio
- Market/stock list
- Stock detail
- Analysis
- Order placement
- Pending orders
- Order history

---

# 46. Explicitly Out of MVP Scope

The following should **not** be implemented as part of the initial core project unless the team later explicitly decides to extend the scope:

- real brokerage integration;
- real money;
- market orders;
- stop orders;
- partial fills;
- real-time WebSocket market feeds;
- user-selectable order duration;
- fundamental-data-based investment analysis;
- exact replication of Warren Buffett or another investor;
- advanced autonomous AI trading;
- AI making trades automatically;
- portfolio-aware AI recommendations;
- complex exchange matching;
- trading fees/commissions;
- slippage modelling.

The Broker Simulator itself explicitly excludes partial fills, persistent storage, fees, slippage, continuous market re-evaluation, and non-limit order types.

---

# 47. Future Enhancements

Once the MVP is stable, the following can be considered.

## Phase 2 — AI-assisted analysis

Add an AI layer that explains the numerical technical analysis in natural language.

The AI receives calculated facts rather than inventing market information.

## Phase 3 — Strategy selection

Allow the trader to choose among multiple strategies.

Example:

```text
Technical
Conservative
Value
Growth
Buffett-inspired
```

## Phase 4 — Fundamental analysis

Add a reliable financial-data source and allow strategies to consider:

- earnings;
- valuation;
- revenue;
- debt;
- profitability;
- cash flow;
- dividends.

## Phase 5 — Conversational advisor

Add an optional chatbot interface that allows questions such as:

> Why did you recommend BUY for AAPL?

or:

> What would change this recommendation to HOLD?

The chatbot is **not the primary advisor UI**. The structured analysis remains the primary feature.

## Phase 6 — Advanced trading features

Potentially:

- market orders;
- stop orders;
- more sophisticated order types;
- richer portfolio analytics;
- backtesting;
- alerts.

These are not required for the current project.

---

# 48. Core Business Invariants

These rules must never be violated.

### Invariant 1 — No negative available cash

```text
available cash >= 0
```

at all times.

### Invariant 2 — No overselling

```text
available shares >= 0
```

for every holding.

### Invariant 3 — Pending orders reserve resources

A pending BUY reserves its required cash.

A pending SELL reserves its required shares.

### Invariant 4 — Failed orders do not alter actual holdings

A failed BUY does not reduce cash.

A failed SELL does not reduce shares.

### Invariant 5 — Successful BUY updates both sides

A successful BUY:

```text
cash decreases
shares increase
```

### Invariant 6 — Successful SELL updates both sides

A successful SELL:

```text
shares decrease
cash increases
```

### Invariant 7 — Holds are released exactly once

When an order reaches:

```text
FULFILLED
FAILED
CANCELLED
```

its associated hold must no longer be active.

### Invariant 8 — Users cannot access other users' state

Portfolio and order access must always be scoped to the authenticated user.

### Invariant 9 — Broker is authoritative for execution

The Trade Executor must not invent its own fulfilment result.

### Invariant 10 — Temporary infrastructure failure is not business failure

A temporary Broker API outage must not automatically make an order FAILED.

---

# 49. Definition of Done

The project can be considered functionally complete when a new user can perform the following end-to-end scenario.

### Scenario

1. Register an account.

2. Log in.

3. See:

```text
Cash = $1,000,000
Holdings = none
Portfolio value = $1,000,000
```

4. Open the stock list.

5. Select a supported stock such as AAPL.

6. See its current simulated price.

7. See historical price information.

8. See Bollinger Band technical analysis.

9. Receive a BUY/HOLD/SELL recommendation.

10. Place a valid limit order.

11. The system validates the order.

12. The system places the appropriate cash/share hold.

13. The order appears as PENDING.

14. The Trade API publishes an event to Kafka.

15. The Trade Executor receives the event.

16. The Trade Executor submits it to the Broker Simulator.

17. The Broker Simulator eventually resolves it.

18. The Trade Executor publishes the result.

19. The Portfolio API releases the hold.

20. If fulfilled, the portfolio is updated.

21. The Trade API updates the order status.

22. The UI displays the final result.

23. The user can see the order in order history.

24. If the user submits another order while resources are already held, the system correctly prevents double-spending.

25. If the user cancels a pending order, the broker order is cancelled first and the associated hold is then released.

This complete flow must work without manual database manipulation.

---

# 50. Final Architecture Principle

The most important design principle for the team is to maintain clear ownership.

```text
React UI
    ↓
User interaction

Trade API
    ↓
Order lifecycle and order state

Portfolio API
    ↓
Money, shares, holds, portfolio valuation

Trade Executor
    ↓
Asynchronous broker communication

Broker Simulator
    ↓
Simulated trade execution decision

Live Pricing API
    ↓
Simulated market prices

Trade Advisor
    ↓
Stock analysis and recommendation

Kafka
    ↓
Asynchronous event communication

PostgreSQL
    ↓
Persistent application state
```

No service should silently take responsibility for another service's domain.

In particular:

**The Broker Simulator does not manage portfolios.**

**The Portfolio API does not decide whether trades execute.**

**The Trade Executor does not decide whether trades should succeed.**

**The Live Pricing API does not provide trading advice.**

**The Trade Advisor does not modify the portfolio.**

**The React frontend does not implement financial business rules.**

The backend services are responsible for enforcing the rules; the frontend is responsible for presenting them.

---

# 51. Final MVP Mental Model

The entire application can be understood as five major capabilities:

```text
                  SIMULATED TRADING PLATFORM

                         ┌───────────┐
                         │   USER    │
                         └─────┬─────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
         PORTFOLIO          TRADING          ANALYSIS
              │                │                │
              │                │                │
        Cash / Shares      Orders / Holds   BUY/HOLD/SELL
              │                │                │
              │                │                │
              ▼                ▼                ▼
         PostgreSQL          Kafka        Pricing API
                               │
                               ▼
                        Trade Executor
                               │
                               ▼
                       Broker Simulator
```

The trader sees a single coherent trading application, while internally the system is composed of separate services with clear responsibilities.

**The primary goal of the MVP is correctness of portfolio/order state and a convincing trading-platform experience. The AI features are enhancements on top of that foundation, not substitutes for the underlying deterministic trading logic.**