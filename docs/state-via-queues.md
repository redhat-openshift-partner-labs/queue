# State Capture via Exchanges/Queues

This document explains how the OpenShift Partner Labs system captures workflow state using message-broker exchanges and queues rather than relying solely on database storage.

## Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant GF as web-form
    participant GS as spreadsheet
    participant n8n as workflow-engine (Ingress)
    participant EX_I as opl.intake<br/>(Exchange)
    participant Q_RAW as intake.raw<br/>(Queue)
    participant ETL as worker-etl
    participant Q_NORM as intake.normalized<br/>(Queue)
    participant Scribe as Scribe<br/>(Orchestrator)
    participant EX_P as opl.provision<br/>(Exchange)
    participant Q_GEN as lab.provision.*<br/>(Queues)
    participant Worker as worker-provisioning
    participant Watcher as provision-watcher
    participant EX_D1 as opl.day1<br/>(Exchange)
    participant Q_D1 as lab.day1.*<br/>(Queues)
    participant D1Worker as worker-day-one
    participant EX_H as opl.handoff<br/>(Exchange)
    participant Q_H as handoff.*<br/>(Queues)
    participant Messenger as Messenger<br/>(team-chat)
    participant DLX as opl.dlx<br/>(Dead Letter)

    Note over GF,DLX: STATE IS THE MESSAGE - Each queue represents a state transition point

    rect rgb(240, 248, 255)
        Note over GF,Q_NORM: INTAKE PIPELINE - State: intake_received -> intake_transforming -> intake_stored
        GF->>GS: Submit form
        GS->>n8n: automation POST (on approval)
        n8n->>EX_I: publish(intake.raw)
        Note right of n8n: State embedded in message:<br/>correlation_id, event_type,<br/>timestamp, payload
        EX_I->>Q_RAW: route by key
        Q_RAW->>ETL: consume
        alt Transform Success
            ETL->>EX_I: publish(intake.normalized)
            Note right of ETL: State transition captured<br/>in new message event_type
        else Transform Failed
            ETL->>EX_I: publish(intake.raw.failed)
            Note right of ETL: Failure state = new<br/>message on failure queue
        end
    end

    rect rgb(255, 248, 240)
        Note over Scribe,Q_GEN: DISPATCH - State: dispatch_review_requested | manifests_generating
        Q_NORM->>Scribe: consume intake.normalized
        Note right of Scribe: Scribe persists to DB<br/>BUT state is also in queue position
        alt Standard Config
            Scribe->>EX_P: publish(lab.provision.generate-manifests)
            Scribe->>EX_I: publish(intake.dispatch.notify-fyi)
            Note right of Scribe: Parallel: provision + notify<br/>State = message in flight
        else Non-Standard
            Scribe->>EX_I: publish(intake.dispatch.review-requested)
            Note right of Scribe: State = waiting in queue<br/>for Messenger to consume
        end
    end

    rect rgb(248, 255, 240)
        Note over EX_P,Watcher: PROVISIONING - State: pr_opened -> cluster_installing -> cluster_ready
        EX_P->>Q_GEN: route to generate-manifests
        Q_GEN->>Worker: consume
        Worker->>EX_P: publish(manifests-complete)
        Note right of Worker: Each pub = state snapshot<br/>Envelope carries full context
        Worker->>EX_P: publish(pr.checks-running)
        Worker->>EX_P: publish(pr.checks-passed)
        Worker->>EX_P: publish(pr.merged)
        Watcher->>EX_P: publish(cluster-installing)
        Note right of Watcher: CronJob polls K8s,<br/>publishes state changes
        Watcher->>EX_P: publish(cluster-ready)
    end

    rect rgb(255, 240, 255)
        Note over EX_D1,D1Worker: DAY-ONE - State: day1 sub-tasks via fan-out/fan-in pattern
        Scribe->>EX_D1: publish(lab.day1.orchestrate)
        EX_D1->>Q_D1: route to orchestrate queue
        Q_D1->>D1Worker: consume orchestrate
        par Parallel Sub-Tasks
            D1Worker->>EX_D1: publish(insights.disable)
            D1Worker->>EX_D1: publish(ssl.create)
            D1Worker->>EX_D1: publish(oauth-hub.create)
            D1Worker->>EX_D1: publish(kubeadmin.set)
        end
        Note right of D1Worker: State = messages in<br/>respective sub-task queues
        D1Worker->>EX_D1: publish(*.complete)
        Note right of D1Worker: Each complete/failed<br/>= state transition event
        D1Worker->>EX_D1: publish(lab.day1.complete)
    end

    rect rgb(240, 255, 255)
        Note over EX_H,Messenger: HANDOFF - State: pending_handoff -> credentials_created -> handoff_complete
        Scribe->>EX_H: publish(day1-complete-notify)
        EX_H->>Q_H: route
        Q_H->>Messenger: consume -> team-chat notification
        Messenger->>EX_H: publish(cluster-ready)
        Note right of Messenger: Human action (slash cmd)<br/>becomes queue message
        Scribe->>EX_H: publish(credentials.create)
        Note right of Scribe: Scribe reacts to messages,<br/>orchestrates via pub/sub
        EX_H-->>Scribe: credentials.complete
        Scribe->>EX_H: publish(welcome-email.send)
        EX_H-->>Scribe: welcome-email.sent
        Scribe->>EX_H: publish(handoff.complete)
    end

    rect rgb(255, 240, 240)
        Note over DLX,DLX: DEAD LETTER - Failed state after max retries
        Q_RAW--xDLX: x-delivery-limit exceeded
        Note right of DLX: DLQ = permanent failure state<br/>Ops monitors for stuck messages
    end

    Note over GF,DLX: KEY INSIGHT: State lives in message position + envelope metadata.<br/>correlation_id links messages across pipeline.<br/>event_type = routing key = current state.<br/>Queue depth = work backlog. DLQ depth = failure count.
```

## How State is Captured

The system uses an **event-sourcing pattern** where messages flowing through queues represent state transitions:

### 1. Messages ARE State

Each message envelope contains the complete state context:

| Field | Purpose |
|-------|---------|
| `event_type` | Current state name (equals routing key) |
| `correlation_id` | Links all messages for one lab request |
| `timestamp` | When state transition occurred |
| `payload` | Full context for that state |

### 2. Queue Position = Workflow Progress

A message's location indicates the current workflow state:

- Message in `intake.raw` = `intake_received` state
- Message in `lab.provision.generate-manifests` = `manifests_generating` state
- Message in `handoff.credentials.create` = `handoff_initiated` state

### 3. State Transitions = Publish New Message

Moving between states means consuming from one queue and publishing to another:

```
intake.raw --consume--> worker-etl --publish--> intake.normalized
     ^                                               ^
     |                                               |
intake_received                              intake_stored
```

### 4. Fan-Out/Fan-In Pattern

Day-one tasks run in parallel using message fan-out:

1. **Fan-out**: Scribe publishes `lab.day1.orchestrate`
2. **Parallel**: worker-day-one publishes multiple sub-task messages (`insights.disable`, `ssl.create`, etc.)
3. **Fan-in**: Scribe counts `*.complete` messages to determine when all sub-tasks finish
4. **Aggregate**: Scribe publishes `lab.day1.complete` when all sub-tasks succeed

### 5. Failure State = Dead Letter Queue

After `x-delivery-limit` retries (typically 3-5), messages land in dead-letter queues:

| Source Queue Pattern | Dead Letter Routing Key |
|---------------------|------------------------|
| `intake.*` | `dlq.intake` |
| `lab.provision.*` | `dlq.provision` |
| `lab.day1.*` | `dlq.day1` |
| `handoff.*` | `dlq.handoff` |

DLQ depth = number of permanently failed operations requiring manual intervention.

## Why This Pattern

### Advantages Over Database-Only State

| Aspect | Database State | Queue-Based State |
|--------|---------------|-------------------|
| **Durability** | Single point of failure | Distributed across quorum queues |
| **Recovery** | Requires replay logic | Resume from queue position |
| **Visibility** | Query required | Queue depth = backlog |
| **Decoupling** | Tight coupling to schema | Components only know message format |
| **Scalability** | DB connection limits | Horizontal consumer scaling |

### Scribe Still Uses a Database

Scribe persists to a database for:
- **Querying**: Portal needs to list/filter labs
- **Reporting**: Historical analytics
- **Idempotency**: Deduplication via `correlation_id`

But the **authoritative workflow state** is the message flow. If Scribe crashes mid-operation:
1. Unacknowledged messages return to queues
2. On restart, Scribe consumes and resumes
3. No "stuck in limbo" states

## Observability

Queue metrics provide real-time state visibility:

| Metric | Meaning |
|--------|---------|
| Queue depth | Work backlog for that state |
| Consumer count | Processing capacity |
| Publish rate | State transition throughput |
| DLQ depth | Failure count requiring attention |
| Message age | Time spent in current state |

## Related Documentation

- [Exchange Model](exchange-model.md) - Routing topology and binding conventions
- [REFERENCE.md](../REFERENCE.md) - Queue names and message schemas
