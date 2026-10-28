![Version](https://img.shields.io/badge/version-3.0.0-blue)
![Java](https://img.shields.io/badge/Java-26-orange)
![Depends](https://img.shields.io/badge/depends%20on-axiom-informational)
![License](https://img.shields.io/badge/license-Limited%20Commercial-red)

# Axiom Telemetry (`axiom-telemetry`)

> The sovereign, zero-dependency in-memory telemetry and black-box flight recorder for Axiom. When an operation fails,
> you get the failure **and** what happened right before it.

`axiom-telemetry` keeps a time-bounded rolling window of successful results in memory. When an instrumented operation
returns a failed `Result`, it automatically attaches a snapshot of that pre-failure window to the failure message.
No agents, no exporters, no external collector.

---

## Architecture overview

```
           Supplier<Result<T>> action
                    │
                    ▼
    SovereignExecution.wieldWithTelemetry(buffer, lookback, action)
                    │
        ┌───────────┴────────────┐
   success                    failure
        │                         │
 buffer.logEvent(value)   buffer.streamHistory(lookback)
        │                         │
        ▼                         ▼
 TemporalStreamBuffer<T>   failure message enriched with:
 AtomicReference<            💥 AHE BREACH | PRE-FAILURE SNAPSHOT: [...]
   PersistentList<            ---------------------------
     (Instant, T)>>           RAW BREACH EVIDENCE: <original message>
 time-window eviction
```

| Class | Responsibility |
|---|---|
| `TemporalStreamBuffer<T>` | Lock-free rolling buffer (`AtomicReference` + immutable `PersistentList`). Evicts events older than the window on every write. |
| `SovereignExecution` | Static helper that runs an action, records successes, and decorates failures with the snapshot. |

Dependency graph:

```
axiom  ◄──  axiom-telemetry
```

---

## Installation

**Maven**

```xml
<dependency>
    <groupId>com.ensemblu</groupId>
    <artifactId>axiom-telemetry</artifactId>
    <version>3.0.0</version>
</dependency>
```

**Gradle (Kotlin DSL)**

```kotlin
implementation("com.ensemblu:axiom-telemetry:3.0.0")
```

JPMS automatic module name: `com.ensemblu.axiom.telemetry` · Runtime: JDK 26 with `--enable-preview`.

---

## Usage examples

### Wrap an operation

```java
import com.ensemblu.axiom.telemetry.SovereignExecution;
import com.ensemblu.axiom.telemetry.TemporalStreamBuffer;
import java.time.Duration;

// One buffer per stream you want to observe; keep it as a long-lived field.
TemporalStreamBuffer<PersistentList<PersistentMap<String, Object>>> recorder =
        TemporalStreamBuffer.ofWindowDuration(Duration.ofMinutes(5));

var result = SovereignExecution.wieldWithTelemetry(
        recorder,
        Duration.ofSeconds(30),            // how far back to include in a failure snapshot
        () -> executioner.wield(conn ->
                executioner.shot("SELECT * FROM accounts").withConnection(conn)));

result.peekFailure(e -> Axiom.Io.log(e.getMessage()).run());
```

On failure the message looks like:

```
💥 AHE BREACH | PRE-FAILURE SNAPSHOT: [ ...last 30s of successful results... ]
---------------------------
RAW BREACH EVIDENCE: Axiom Native Shot Error: PostgreSQL execution failed ...
```

### Instrument manually

```java
Result<Order> placed = recorder.instrument(orderService.place(cmd)); // logs only on success

recorder.logEvent(heartbeat);                                       // log arbitrary events

Result<PersistentList<Order>> lastMinute = recorder.streamHistory(Duration.ofMinutes(1));
```

### Isolate test waves

```java
@AfterEach
void reset() {
    recorder.clear();
}
```

---

## Configuration guide

| Setting | Where | Guidance |
|---|---|---|
| Window duration | `TemporalStreamBuffer.ofWindowDuration(Duration)` | Upper bound on retained history. Every `logEvent` evicts entries older than `now - window`. |
| Lookback | `wieldWithTelemetry(.., lookback, ..)` / `streamHistory(lookback)` | How much of the window goes into a snapshot. Should be ≤ window duration. |
| Buffer `null` | `wieldWithTelemetry(null, ..)` | Allowed. Successes are not recorded and failures report `Buffer Unavailable`. |

Sizing notes:

- Memory use is proportional to *events per window* × *event size*. Prefer storing small summaries (ids, counts,
  status) over full payloads for high-throughput streams.
- The buffer is thread-safe (`AtomicReference.updateAndGet`). Under very high contention, shard by using one buffer per
  worker or per stream.
- Snapshots are rendered with `Dop.toString`, so events should be Axiom data types or have a meaningful `toString()`.

---

## License

**Limited Commercial License** — free for evaluation, testing, and non-commercial development.
Commercial or production use requires a paid annual contract.
See [LICENSE](https://raw.githubusercontent.com/ensemblu-corp/axiom/main/LICENSE) · Contact: contact@ensemblu.com

© Ensemblu Corp
