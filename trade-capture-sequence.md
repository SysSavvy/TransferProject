# Trade capture sequence (corrected)

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant UI as Angular Web App
    participant Auth as Auth Service
    participant API as Trade REST API (Spring Boot)
    participant DB as Postgres
    participant K as Kafka Broker
    participant TE as Trade Executor
    participant MD as Fauxnance Market Data API

    Client->>UI: Click Buy (AAPL, qty 10)
    UI->>UI: Validate form and generate idempotencyKey (UUID)

    opt No valid access token
        UI->>Auth: POST /auth/login (username, password)
        alt Authentication failed
            Auth-->>UI: 401 AUTH-401
            UI-->>Client: Show "Please log in again"
        else Credentials valid
            Auth-->>UI: 200 accessToken, refreshToken
        end
    end

    UI->>API: POST /api/v1/orders {accountId, symbol=AAPL, side=BUY, quantity=10, price, idempotencyKey} + Bearer JWT
    API->>API: Verify JWT signature, expiry, accountId claim

    alt Token missing, expired or invalid
        API-->>UI: 401 AUTH-401
        UI->>Auth: POST /auth/refresh {refreshToken}
        Auth-->>UI: 200 new accessToken, refreshToken
        UI->>API: Retry POST /api/v1/orders (same idempotencyKey)
    else accountId claim does not match body accountId
        API-->>UI: 403 ACC-403
        UI-->>Client: Show "Account not accessible"
    else Token valid
        API->>DB: Load account, instrument, cash balance, holdings
        DB-->>API: Account, Instrument, balance, position

        Note over API: Rules 1-7 in order, first failure wins

        alt Rejection at validation
            API->>DB: INSERT order status=REJECTED (audit trail)
            DB-->>API: committed
            API-->>UI: 404 ACC-404 or INS-404, 403 ACC-403, 422 VAL-422, 400 ORD-400, 409 ORD-409
            UI-->>Client: Show rejection reason from errorCode
        else Rules 1-7 pass
            API->>DB: BEGIN, INSERT order status=NEW with idempotencyKey, COMMIT
            alt idempotencyKey already used (unique constraint)
                DB-->>API: Unique violation
                API-->>UI: 409 ORD-409 Duplicate order
                UI-->>Client: Show "Duplicate order"
            else Database unavailable
                API-->>UI: 5xx error
                UI-->>Client: Show "Try again later"
            else Order recorded
                DB-->>API: committed, orderId

                Note over API,K: Publish only after the database commit
                API->>K: Publish to orders, key=accountId, eventType=ORDER_PLACED, source=trade-api
                opt Publish fails
                    API->>API: Log, order stays NEW and is republished from the order table
                end
                API-->>UI: 200 {orderId, status=NEW, message=Order accepted}
                UI-->>Client: Show "Order accepted"

                K->>TE: Consume orders (group trade-executor), ORDER_PLACED
                TE->>MD: Request quote for AAPL
                MD-->>TE: Quote price, stale flag

                alt Rejection at execution (quote unavailable or stale, PRICE_NOT_MET, INSUFFICIENT_FUNDS, INSTRUMENT_NOT_TRADABLE)
                    TE->>DB: BEGIN, UPDATE order status=REJECTED, COMMIT
                    DB-->>TE: committed
                    TE->>K: Publish to trade-events, key=accountId, eventType=ORDER_REJECTED, status=REJECTED, reason
                else Fill rules pass
                    TE->>DB: BEGIN
                    TE->>DB: UPDATE order status=FILLED, executedPrice (trade record)
                    TE->>DB: UPDATE account cashBalance minus quantity x executedPrice
                    TE->>DB: UPSERT position (account, AAPL) quantity plus 10, averageCost
                    TE->>DB: COMMIT (order, cash and position in one transaction)
                    DB-->>TE: committed
                    TE->>K: Publish to trade-events, key=accountId, eventType=ORDER_FILLED, status=FILLED, executedPrice, cashDelta, positionQuantityAfter, averageCostAfter
                end

                loop Until order status is not NEW
                    UI->>API: GET /api/v1/accounts/{id}/orders
                    API->>DB: SELECT orders for account
                    DB-->>API: Orders with status
                    API-->>UI: 200 order history
                end

                alt status=FILLED
                    UI->>API: GET /api/v1/accounts/{id}/positions
                    API-->>UI: 200 AAPL quantity, averageCost
                    UI->>API: GET /api/v1/accounts/{id}/balance
                    API-->>UI: 200 cashBalance
                    UI-->>Client: Show filled order and updated position
                else status=REJECTED
                    UI-->>Client: Show "Order rejected"
                end
            end
        end
    end
```

## Notes

- Endpoint, request fields and error codes follow `shared/contracts/trade-api.yaml` (`placeOrder`). The idempotency key is a body field, not a header.
- Topics, key, event types and `source` follow `shared/contracts/kafka-topics.md`: `orders` carries `ORDER_PLACED`, and `trade-events` carries `ORDER_FILLED` and `ORDER_REJECTED`.
- The order is committed as `NEW` before `ORDER_PLACED` is published and before the Trade Executor runs (BR-06). A rejected order is also recorded.
- The order status and `executedPrice` (the trade record), the cash balance and the position update in one transaction (BR-09).
- The Angular UI never talks to Kafka. It polls the account endpoints for the outcome.
- The contract has no `Trade`, `OrderEvent` or `Settlement` table. The order row is the audit trail and the trade record.
- The contract uses status `NEW`, so this diagram uses `NEW` where the class model uses `PLACED` and `ACCEPTED`. Align `OrderStatus` in the class model.
- A request for an account that does not exist (`ACC-404`) has no account to record the rejected order against.
