# Mermaid Diagram Patterns

Copy-ready templates. Use Mermaid v11 syntax. For advanced syntax rules, activate the `mermaidjs-v11` skill.

## 1. Architecture / C4 System Context (flowchart style)

Reliable across all Mermaid renderers — prefer this over the native `C4Context` block.

```mermaid
flowchart TB
    subgraph users[Users]
        trader[Trader]
        admin[Admin / Ops]
    end
    subgraph frontend[Frontend Layer]
        fe[Terminal FE]
        adminfe[Admin FE]
    end
    subgraph backend[Backend Services]
        mw[Middleware / API Gateway]
        oms[OMS - Order Management]
        gw[FIX / Integration Gateway]
        stp[STP - Straight-Through Processing]
    end
    subgraph data[Data Layer]
        db[(Database)]
        mq[[Message Queue: NATS/Kafka]]
    end
    ext[External System / Exchange]

    trader --> fe --> mw
    admin --> adminfe --> stp
    mw --> oms
    oms --> gw --> ext
    oms <--> mq
    stp <--> mq
    oms --> db
    stp --> db
```

## 2. Native C4 Context (when renderer supports it)

```mermaid
C4Context
    title System Context — <System Name>
    Person(trader, "Trader", "Places orders")
    System_Boundary(sys, "Trading Platform") {
        System(fe, "Terminal FE", "Web frontend")
        System(mw, "Middleware", "API gateway / orchestration")
        System(oms, "OMS", "Order management")
        System(gw, "FIX Gateway", "Exchange connectivity")
    }
    System_Ext(exch, "Exchange", "External matching engine")
    Rel(trader, fe, "Uses")
    Rel(fe, mw, "REST/HTTPS")
    Rel(mw, oms, "REST/gRPC")
    Rel(oms, gw, "Internal")
    Rel(gw, exch, "FIX 4.4")
```

## 3. Sequence Diagram — business flow across services

```mermaid
sequenceDiagram
    autonumber
    actor U as Trader
    participant FE as Terminal FE
    participant MW as Middleware
    participant OMS as OMS
    participant GW as FIX Gateway
    participant MQ as NATS
    participant DB as Database
    participant EX as Exchange

    U->>FE: Fill order form, submit
    FE->>MW: POST /orders {payload}
    MW->>MW: Validate + auth
    MW->>OMS: Create order request
    OMS->>DB: Persist order (status=PENDING)
    OMS->>MQ: Publish order.new
    GW-->>MQ: Consume order.new
    GW->>EX: FIX NewOrderSingle (35=D)
    EX-->>GW: ExecutionReport (35=8)
    GW->>MQ: Publish order.status
    OMS-->>MQ: Consume order.status
    OMS->>DB: Update order status
    OMS-->>MW: Status update
    MW-->>FE: Response / push
    FE-->>U: Order confirmed
```

## Diagram Rules

- One concern per diagram; split rather than overload.
- Use `autonumber` in sequence diagrams for step traceability.
- Label every relationship/arrow with the protocol or message type.
- Keep participant names consistent with Part 1 service names.
- Validate Mermaid renders before finalizing (preview skill or markdown viewer).