## Exchange Model

### Design: One Topic Exchange Per Domain

Every message flows through a **topic exchange** named `opl.<domain>`. Publishers send to the exchange with a **routing key equal to the queue name**. Consumers bind their queues using exact or wildcard routing key patterns.

| Exchange Name     | Type  | Durable | Domain Covered                                | Example Routing Keys |
|-------------------|-------|---------|-----------------------------------------------|----------------------|
| `opl.intake`      | topic | yes     | Intake pipeline + dispatch                    | `intake.raw`, `intake.normalized`, `intake.dispatch.provision` |
| `opl.provision`   | topic | yes     | Cluster provisioning lifecycle                | `lab.provision.generate-manifests`, `lab.provision.cluster-ready` |
| `opl.deprovision` | topic | yes     | Cluster deprovisioning lifecycle              | `lab.deprovision.requested`, `lab.deprovision.cluster-removed` |
| `opl.day1`        | topic | yes     | Day-one configuration tasks                   | `lab.day1.orchestrate`, `lab.day1.ssl.create`, `lab.day1.complete` |
| `opl.day2`        | topic | yes     | Day-two operations                            | `lab.day2.orchestrate`, `lab.day2.task.execute` |
| `opl.handoff`     | topic | yes     | Handoff pipeline (credentials, welcome email) | `handoff.credentials.create`, `handoff.welcome-email.sent` |
| `opl.notify`      | topic | yes     | Notification commands + delivery confirmations | `notify.user.lab-ready`, `notify.admin.failure`, `notification.sent` |
| `opl.dlx`         | topic | yes     | Dead-letter exchange                          | `dlq.intake`, `dlq.provision`, `dlq.generic` |

---

### Publishing Convention

```
// Every publisher follows this pattern — no exceptions.
channel.publish(
  exchange   = "opl.<domain>",    // e.g. "opl.day1"
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
| **workflow-engine** | `opl.intake` | `intake.raw` | pub | Source-validated ingress only |
| **worker-etl** | `opl.intake` | `intake.raw` | sub | Consumes raw payloads |
| **worker-etl** | `opl.intake` | `intake.normalized`, `intake.raw.failed` | pub | Publishes transform results |
| **scribe** | `opl.intake` | `intake.normalized`, `intake.raw.failed`, `intake.dispatch.*` | sub | Orchestrator — all intake status events |
| **scribe** | `opl.intake` | `intake.dispatch.notify-fyi`, `intake.dispatch.review-requested` | pub | Dispatch commands to messenger |
| **scribe** | `opl.provision` | `lab.provision.manifests-complete`, `lab.provision.pr.*`, `lab.provision.cluster-*` | sub | Provisioning status events |
| **scribe** | `opl.provision` | `lab.provision.generate-manifests` | pub | Commands to worker-provisioning |
| **scribe** | `opl.deprovision` | `lab.deprovision.archive-complete`, `lab.deprovision.pr.merged`, `lab.deprovision.cluster-*` | sub | Deprovision status events |
| **scribe** | `opl.deprovision` | `lab.deprovision.requested` | pub | Commands to worker-deprovision |
| **scribe** | `opl.day1` | `lab.day1.*.complete`, `lab.day1.*.failed`, `lab.day1.complete`, `lab.day1.failed` | sub | Day-one status events |
| **scribe** | `opl.day1` | `lab.day1.orchestrate`, `lab.day1.complete`, `lab.day1.failed` | pub | Commands to worker-day-one |
| **scribe** | `opl.day2` | `lab.day2.task.complete`, `lab.day2.task.failed`, `lab.day2.complete`, `lab.day2.failed` | sub | Day-two status events |
| **scribe** | `opl.day2` | `lab.day2.orchestrate` | pub | Commands to worker-day-two |
| **scribe** | `opl.handoff` | `handoff.cluster-ready`, `handoff.credentials.complete`, `handoff.credentials.failed`, `handoff.welcome-email.sent`, `handoff.welcome-email.failed` | sub | Handoff status events |
| **scribe** | `opl.handoff` | `handoff.day1-complete-notify`, `handoff.credentials.create`, `handoff.welcome-email.send`, `handoff.complete` | pub | Handoff commands |
| **scribe** | `opl.notify` | `notification.sent`, `notification.failed` | sub | Delivery confirmations |
| **scribe** | `opl.notify` | `notify.user.*`, `notify.admin.*` | pub | Notification commands |
| **messenger** | `opl.intake` | `intake.dispatch.notify-fyi`, `intake.dispatch.review-requested` | sub | Team-chat message triggers |
| **messenger** | `opl.intake` | `intake.dispatch.provision`, `intake.dispatch.review`, `intake.dispatch.provision-after-review`, `intake.dispatch.rejected` | pub | Button-click / slash-command responses |
| **messenger** | `opl.handoff` | `handoff.day1-complete-notify`, `handoff.complete` | sub | Team-chat handoff notifications |
| **messenger** | `opl.handoff` | `handoff.cluster-ready` | pub | `/cluster {id} ready` slash command |
| **worker-provisioning** | `opl.provision` | `lab.provision.generate-manifests` | sub | Single command queue |
| **worker-provisioning** | `opl.provision` | `lab.provision.manifests-complete`, `lab.provision.pr.*` | pub | PR lifecycle events |
| **worker-day-one** | `opl.day1` | `lab.day1.orchestrate`, `lab.day1.*.disable`, `lab.day1.*.create`, `lab.day1.*.patch`, `lab.day1.*.set`, `lab.day1.*.configure` | sub | All day-one command verbs (see noLocal note below) |
| **worker-day-one** | `opl.day1` | `lab.day1.*.disable`, `lab.day1.*.create`, `lab.day1.*.patch`, `lab.day1.*.set`, `lab.day1.*.configure`, `lab.day1.*.complete`, `lab.day1.*.failed` | pub | Fan-out sub-tasks + completion results |
| **worker-day-two** | `opl.day2` | `lab.day2.orchestrate`, `lab.day2.task.execute` | sub | Day-two commands |
| **worker-day-two** | `opl.day2` | `lab.day2.task.execute`, `lab.day2.task.complete`, `lab.day2.task.failed` | pub | Day-two sub-tasks + results |
| **worker-deprovision** | `opl.deprovision` | `lab.deprovision.requested` | sub | Deprovision command |
| **worker-deprovision** | `opl.deprovision` | `lab.deprovision.archive-complete` | pub | Archive result |
| **provision-watcher** | `opl.provision` | `lab.provision.cluster-installing`, `lab.provision.cluster-ready`, `lab.provision.cluster-failed` | pub | Cluster status from CronJob polling |
| **deprovision-watcher** | `opl.deprovision` | `lab.deprovision.cluster-removing`, `lab.deprovision.cluster-removed`, `lab.deprovision.failed` | pub | Deprovision status from gitops-controller Job |
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

**Problem:** worker-day-one and worker-day-two both publish and consume on their respective exchanges (`opl.day1` and `opl.day2`). When worker-day-one receives `lab.day1.orchestrate`, it fans out sub-tasks like `lab.day1.ssl.create` back to the same exchange. If worker-day-one binds `lab.day1.#`, it will also receive the `*.complete` and `*.failed` messages it publishes — messages intended for Scribe, not for itself.

**The message-broker does not support AMQP's `noLocal` flag on `basic.consume`.** You cannot tell the broker to suppress messages originating from the same connection.

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

Every queue declares `opl.dlx` as its dead-letter exchange. The dead-letter routing key maps to the appropriate DLQ bucket:

```yaml
# Example queue declaration (worker-provisioning)
queue: "lab.provision.generate-manifests"
arguments:
  x-dead-letter-exchange: "opl.dlx"
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

The `opl.dlx` exchange declares an **alternate exchange** (`opl.dlx.unroutable`) bound to the `dlq.generic` queue. Any message that arrives at `opl.dlx` without a matching binding lands in `dlq.generic` rather than being silently dropped.

---

### Why This Model

**Why topic exchanges?** The queue names are already hierarchical dot-notation. Topic exchanges let consumers use wildcards (`lab.day1.*.complete`, `lab.provision.pr.*`) instead of maintaining a brittle enumerated binding list. Scribe — which orchestrates across 30+ event types — benefits most, but every component that handles a category of messages benefits.

**Why per-domain, not one global exchange?** Blast radius and least-privilege. Messenger needs `opl.intake` and `opl.handoff` — it has no business seeing provisioning traffic. Worker-provisioning only touches `opl.provision`. Per-domain exchanges make message-broker user permissions straightforward: grant `configure`/`write`/`read` per exchange, per component.

**Why split into `opl.provision`, `opl.day1`, etc.?** Finer-grained RBAC. Worker-provisioning only needs access to `opl.provision`, not `opl.day1`. This separation enforces least-privilege at the broker level and makes permission audits cleaner. The tradeoff is more exchanges to manage, but the Topology Operator handles declaration.

---
