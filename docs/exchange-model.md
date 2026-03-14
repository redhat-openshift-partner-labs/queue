## Exchange Model

### Design: One Topic Exchange Per Domain

Every message flows through a **topic exchange** named `opl.<domain>`. Publishers send to the exchange with a **routing key equal to the queue name**. Consumers bind their queues using exact or wildcard routing key patterns.

| Exchange Name   | Type  | Durable | Domain Covered                                | Example Routing Keys |
|-----------------|-------|---------|-----------------------------------------------|----------------------|
| `opl.intake`    | topic | yes     | Intake pipeline + dispatch                    | `intake.raw`, `intake.normalized`, `intake.dispatch.provision` |
| `opl.lab`       | topic | yes     | Provisioning, deprovisioning, day-one, day-two | `lab.provision.cluster-ready`, `lab.day1.ssl.create`, `lab.deprovision.requested` |
| `opl.handoff`   | topic | yes     | Handoff pipeline (credentials, welcome email) | `handoff.credentials.create`, `handoff.welcome-email.sent` |
| `opl.notify`    | topic | yes     | Notification commands + delivery confirmations | `notify.user.lab-ready`, `notify.admin.failure`, `notification.sent` |
| `opl.dlq`       | topic | yes     | Dead-letter queues                            | `dlq.intake`, `dlq.provision`, `dlq.generic` |

---

### Publishing Convention

```
// Every publisher follows this pattern — no exceptions.
channel.publish(
  exchange   = "opl.<domain>",    // e.g. "opl.lab"
  routingKey = "<queue.name>",    // e.g. "lab.day1.ssl.create"
  message    = <envelope>
)
```

**Rules:**

1. **Routing key = queue name** exactly as listed in the Queue / Topic Names tables.
2. **`event_type` in the envelope must equal the routing key.** This is validated at consumption — mismatches are rejected.
3. **Never publish to the default exchange (`""`)** or to a queue directly. All messages enter through a named `opl.*` exchange.
4. All exchanges are declared `durable: true`. Messages carrying state transitions (`*.complete`, `*.failed`, state commands) should be published `persistent: true` (delivery mode 2).

---

### Binding Convention

Consumers bind their queues to exchanges using exact matches or RabbitMQ topic wildcards:

| Pattern Type       | Syntax                        | Matches |
|--------------------|-------------------------------|---------|
| Exact              | `lab.day1.ssl.create`         | That one routing key only |
| Single-word wildcard | `lab.day1.*.complete`       | `lab.day1.ssl.complete`, `lab.day1.rbac.complete`, etc. (`*` = one dot-delimited word) |
| Subtree wildcard   | `lab.day1.#`                  | Everything under `lab.day1` at any depth (`#` = zero or more words) |
| Domain-wide        | `#`                           | Every message on the exchange (audit, logging) |

---

### Binding Table by Component

This table is the authoritative binding contract. If a component is not listed for an exchange, it has no business publishing to or consuming from it.

| Component | Exchange | Binding(s) | Pub/Sub | Notes |
|-----------|----------|------------|---------|-------|
| **n8n** | `opl.intake` | `intake.raw` | pub | Source-validated ingress only |
| **worker-etl** | `opl.intake` | `intake.raw` | sub | Consumes raw payloads |
| **worker-etl** | `opl.intake` | `intake.normalized`, `intake.raw.failed` | pub | Publishes transform results |
| **scribe** | `opl.intake` | `intake.normalized`, `intake.raw.failed`, `intake.dispatch.*` | sub | Orchestrator — all intake status events |
| **scribe** | `opl.intake` | `intake.dispatch.notify-fyi`, `intake.dispatch.review-requested` | pub | Dispatch commands to messenger |
| **scribe** | `opl.lab` | `lab.provision.manifests-complete`, `lab.provision.pr.*`, `lab.provision.cluster-*`, `lab.day1.*.complete`, `lab.day1.*.failed`, `lab.day1.complete`, `lab.day1.failed`, `lab.deprovision.archive-complete`, `lab.deprovision.pr.merged`, `lab.deprovision.cluster-*` | sub | Provisioning + day-one + deprovision status |
| **scribe** | `opl.lab` | `lab.provision.generate-manifests`, `lab.day1.orchestrate`, `lab.day2.orchestrate`, `lab.deprovision.requested`, `lab.day1.complete`, `lab.day1.failed` | pub | Commands to workers |
| **scribe** | `opl.handoff` | `handoff.cluster-ready`, `handoff.credentials.complete`, `handoff.credentials.failed`, `handoff.welcome-email.sent`, `handoff.welcome-email.failed` | sub | Handoff status events |
| **scribe** | `opl.handoff` | `handoff.day1-complete-notify`, `handoff.credentials.create`, `handoff.welcome-email.send`, `handoff.complete` | pub | Handoff commands |
| **scribe** | `opl.notify` | `notification.sent`, `notification.failed` | sub | Delivery confirmations |
| **scribe** | `opl.notify` | `notify.user.*`, `notify.admin.*` | pub | Notification commands |
| **messenger** | `opl.intake` | `intake.dispatch.notify-fyi`, `intake.dispatch.review-requested` | sub | Slack message triggers |
| **messenger** | `opl.intake` | `intake.dispatch.provision`, `intake.dispatch.review`, `intake.dispatch.provision-after-review`, `intake.dispatch.rejected` | pub | Button-click / slash-command responses |
| **messenger** | `opl.handoff` | `handoff.day1-complete-notify`, `handoff.complete` | sub | Slack handoff notifications |
| **messenger** | `opl.handoff` | `handoff.cluster-ready` | pub | `/cluster {id} ready` slash command |
| **worker-provisioning** | `opl.lab` | `lab.provision.generate-manifests` | sub | Single command queue |
| **worker-provisioning** | `opl.lab` | `lab.provision.manifests-complete`, `lab.provision.pr.*` | pub | PR lifecycle events |
| **worker-day-one** | `opl.lab` | `lab.day1.orchestrate`, `lab.day1.*.disable`, `lab.day1.*.create`, `lab.day1.*.patch`, `lab.day1.*.set`, `lab.day1.*.configure` | sub | All day-one command verbs (see noLocal note below) |
| **worker-day-one** | `opl.lab` | `lab.day1.*.disable`, `lab.day1.*.create`, `lab.day1.*.patch`, `lab.day1.*.set`, `lab.day1.*.configure`, `lab.day1.*.complete`, `lab.day1.*.failed` | pub | Fan-out sub-tasks + completion results |
| **worker-day-two** | `opl.lab` | `lab.day2.orchestrate`, `lab.day2.task.execute` | sub | Day-two commands |
| **worker-day-two** | `opl.lab` | `lab.day2.task.execute`, `lab.day2.task.complete`, `lab.day2.task.failed` | pub | Day-two sub-tasks + results |
| **worker-deprovision** | `opl.lab` | `lab.deprovision.requested` | sub | Deprovision command |
| **worker-deprovision** | `opl.lab` | `lab.deprovision.archive-complete` | pub | Archive result |
| **provision-watcher** | `opl.lab` | `lab.provision.cluster-installing`, `lab.provision.cluster-ready`, `lab.provision.cluster-failed` | pub | Cluster status from CronJob polling |
| **deprovision-watcher** | `opl.lab` | `lab.deprovision.cluster-removing`, `lab.deprovision.cluster-removed`, `lab.deprovision.failed` | pub | Deprovision status from ArgoCD Job |
| **worker-credentials** | `opl.handoff` | `handoff.credentials.create` | sub | Single command queue |
| **worker-credentials** | `opl.handoff` | `handoff.credentials.complete`, `handoff.credentials.failed` | pub | Credential creation results |
| **worker-notification** | `opl.notify` | `notify.#` | sub | All outbound notification commands |
| **worker-notification** | `opl.notify` | `notification.sent`, `notification.failed` | pub | Delivery confirmations |
| **worker-notification** | `opl.handoff` | `handoff.welcome-email.send` | sub | Welcome email command |
| **worker-notification** | `opl.handoff` | `handoff.welcome-email.sent`, `handoff.welcome-email.failed` | pub | Welcome email results |

---

### The `notify.*` / `notification.*` Split

The `opl.notify` exchange carries two distinct naming prefixes:

| Prefix | Direction | Publisher | Consumer | Examples |
|--------|-----------|-----------|----------|----------|
| `notify.*` | **Command** (outbound) | scribe | worker-notification | `notify.user.lab-ready`, `notify.admin.failure` |
| `notification.*` | **Confirmation** (inbound) | worker-notification | scribe | `notification.sent`, `notification.failed` |

This is intentional. The prefix distinguishes command flow (`notify.*` = "send this notification") from reply flow (`notification.*` = "here's what happened"). Both live on `opl.notify` because they belong to the same domain and the same two components. Separating them into two exchanges would add infrastructure for no access-control benefit.

**Binding consequence:** worker-notification binds `notify.#` (commands only). Scribe binds `notification.sent` and `notification.failed` (confirmations only). The prefix split guarantees no overlap — neither component accidentally consumes its own messages.

---

### Self-Published Message Handling (noLocal Pattern)

**Problem:** worker-day-one and worker-day-two both publish and consume on `opl.lab`. When worker-day-one receives `lab.day1.orchestrate`, it fans out sub-tasks like `lab.day1.ssl.create` back to the same exchange. If worker-day-one binds `lab.day1.#`, it will also receive the `*.complete` and `*.failed` messages it publishes — messages intended for Scribe, not for itself.

**RabbitMQ does not support AMQP's `noLocal` flag on `basic.consume`.** You cannot tell the broker to suppress messages originating from the same connection.

**Solution — separate queues with verb-specific bindings:**

Instead of one broad `lab.day1.#` binding, worker-day-one binds only the command verbs:

```
// worker-day-one queue bindings (commands it acts on):
lab.day1.orchestrate
lab.day1.*.disable
lab.day1.*.create
lab.day1.*.patch
lab.day1.*.set
lab.day1.*.configure
```

It does **not** bind:
```
// These go to Scribe's queue — worker-day-one never sees them:
lab.day1.*.complete
lab.day1.*.failed
lab.day1.complete
lab.day1.failed
```

This is why the Queue / Topic Names tables use distinct verb suffixes for commands vs results. The naming convention is the routing mechanism — commands use action verbs (`create`, `patch`, `set`, `disable`, `configure`), results use outcome nouns (`complete`, `failed`). Wildcard bindings on the verb suffix cleanly separate the two flows without application-level filtering.

The same pattern applies to **worker-day-two** (`lab.day2.task.execute` vs `lab.day2.task.complete`/`lab.day2.task.failed`).

> **If a future task verb collides with a result suffix** (unlikely given the convention), fall back to application-level filtering: check `source` in the envelope and discard messages where `source` equals the consuming component's own identifier. But prefer fixing the naming first.

---

### Dead-Letter Exchange Configuration

Every queue declares `opl.dlq` as its dead-letter exchange. The dead-letter routing key maps to the appropriate DLQ bucket:

```yaml
# Example queue declaration (worker-provisioning)
queue: "lab.provision.generate-manifests"
arguments:
  x-dead-letter-exchange: "opl.dlq"
  x-dead-letter-routing-key: "dlq.provision"
```

**DLQ routing key mapping:**

| Source Queue Prefix        | Dead-Letter Routing Key |
|---------------------------|------------------------|
| `intake.*`                | `dlq.intake`           |
| `lab.provision.*`         | `dlq.provision`        |
| `lab.deprovision.*`       | `dlq.deprovision`      |
| `lab.day1.*`              | `dlq.day1`             |
| `lab.day2.*`              | `dlq.day2`             |
| `handoff.*`               | `dlq.handoff`          |
| `notify.*` / `notification.*` | `dlq.notification` |
| *(unroutable)*            | `dlq.generic`          |

The `opl.dlq` exchange declares an **alternate exchange** (`opl.dlq.unroutable`) bound to the `dlq.generic` queue. Any message that arrives at `opl.dlq` without a matching binding lands in `dlq.generic` rather than being silently dropped.

---

### Why This Model

**Why topic exchanges?** The queue names are already hierarchical dot-notation. Topic exchanges let consumers use wildcards (`lab.day1.*.complete`, `lab.provision.pr.*`) instead of maintaining a brittle enumerated binding list. Scribe — which orchestrates across 30+ event types — benefits most, but every component that handles a category of messages benefits.

**Why per-domain, not one global exchange?** Blast radius and least-privilege. Messenger needs `opl.intake` and `opl.handoff` — it has no business seeing `opl.lab` traffic. Worker-provisioning only touches `opl.lab`. Per-domain exchanges make RabbitMQ user permissions straightforward: grant `configure`/`write`/`read` per exchange, per component.

**Why not split `opl.lab` further** (e.g., `opl.lab.provision`, `opl.lab.day1`)? The topic wildcard bindings already provide that isolation (`lab.provision.#` vs `lab.day1.#` on the same exchange). Splitting multiplies exchanges without meaningful access-control benefit — RabbitMQ permissions are per-exchange, but the components that touch `opl.lab` (Scribe, workers, watchers) generally need cross-subdomain visibility anyway. Revisit only if a security audit demands exchange-level RBAC boundaries within the `lab` domain.

---
