# LOGGER Levels — Best Practices
## Quick Mental Model

```
TRACE  → "every single step, line by line"        (deep-dive debugging only)
DEBUG  → "here's exactly what's happening"         (dev-only, verbose)
INFO   → "this happened, as expected"              (normal operation)
WARN   → "this happened, unexpectedly, but we're okay" (degraded, recovered)
ERROR  → "this failed, and we couldn't recover"    (broken, needs attention)
FATAL/CRITICAL → "the app can't continue"          (unrecoverable, shutting down)
```

**One-line test per level:**
- **TRACE:** Would this flood the console during a single request/operation?
- **DEBUG:** Would I only want this while actively debugging?
- **INFO:** Does this tell the story of normal system behavior?
- **WARN:** Did something unexpected happen, but the system handled it?
- **ERROR:** Did an operation fail and require human/system intervention?
- **FATAL/CRITICAL:** Is the process itself now unable to continue running?

---

## TRACE

| Scenario | Sample LOGGER Statement |
|---|---|
| Method entry/exit (finer than DEBUG) | `LOGGER.trace("Entering method: {} with args={}", methodName, args);` |
| Every loop iteration (high-frequency) | `LOGGER.trace("Iteration {}: current value={}", i, currentValue);` |
| Raw low-level I/O | `LOGGER.trace("Raw bytes received: {}", bytesToHex(data));` |
| Lock acquisition/release | `LOGGER.trace("Thread {} acquired lock on {}", threadName, lockName);` |
| Detailed object state dump | `LOGGER.trace("Full object state: {}", objectMapper.writeValueAsString(obj));` |
| Internal state machine transitions | `LOGGER.trace("State transition: {} -> {} on event {}", oldState, newState, event);` |
| Every HTTP header (not just payload) | `LOGGER.trace("Request headers: {}", headers);` |

---

## DEBUG

| Scenario | Sample LOGGER Statement |
|---|---|
| Function entry with params | `LOGGER.debug("Entering calculateTax: amount={}, region={}", amount, region);` |
| Loop/batch progress | `LOGGER.debug("Processing item {}/{} in batch", index, total);` |
| Cache miss | `LOGGER.debug("Cache miss for key={}, fetching from DB", cacheKey);` |
| SQL query executed | `LOGGER.debug("Executing query: {} with params={}", sql, params);` |
| Intermediate computed value | `LOGGER.debug("Computed discount={}%, beforeTax={}", discountPct, beforeTax);` |
| Outbound/inbound payload | `LOGGER.debug("Request to Stripe: {}", requestPayload);` |
| Health check result | `LOGGER.debug("Health check: DB={}, Cache={}", dbStatus, cacheStatus);` |

---

## INFO

| Scenario | Sample LOGGER Statement |
|---|---|
| Service startup | `LOGGER.info("Server started on port {}, env={}", port, environment);` |
| Service shutdown | `LOGGER.info("Server shutting down gracefully, pending requests={}", pendingCount);` |
| Config loaded | `LOGGER.info("Loaded config from {}, {} keys", configPath, keyCount);` |
| Request completed | `LOGGER.info("Request completed: {} {} -> {} in {}ms", method, path, status, duration);` |
| Scheduled job success | `LOGGER.info("Scheduled job '{}' completed: {} records processed", jobName, recordCount);` |
| User business event | `LOGGER.info("User {} registered successfully", userId);` |
| Order/domain event | `LOGGER.info("Order {} shipped, carrier={}", orderId, carrier);` |
| DB connection established | `LOGGER.info("Connected to database: host={}, db={}", host, dbName);` |
| Cache warmed | `LOGGER.info("Product cache warmed with {} entries", entryCount);` |
| Feature flag state | `LOGGER.info("Feature flag '{}' disabled, using legacy flow", flagName);` |

---

## WARN

| Scenario | Sample LOGGER Statement |
|---|---|
| Retry succeeded | `LOGGER.warn("Retry #{} succeeded for payment API call, orderId={}", attempt, orderId);` |
| Deprecated config/API usage | `LOGGER.warn("Config key '{}' is deprecated, use '{}' instead", oldKey, newKey);` |
| Fallback/default used | `LOGGER.warn("No locale preference set for user {}, defaulting to en-US", userId);` |
| Resource nearing limit | `LOGGER.warn("DB connection pool at {}% capacity ({}/{})", pctUsed, active, max);` |
| Slow but completed operation | `LOGGER.warn("Query took {}ms, exceeds threshold of {}ms: {}", duration, threshold, sql);` |
| Auto-corrected input | `LOGGER.warn("Invalid whitespace trimmed from email field, userId={}", userId);` |
| Unusual but handled input | `LOGGER.warn("Empty cart at checkout, redirecting user {}", userId);` |
| External service degraded | `LOGGER.warn("Payment gateway latency elevated: {}ms avg (threshold {}ms)", avgLatency, threshold);` |
| Circuit breaker opened | `LOGGER.warn("Circuit breaker OPEN for service {}", serviceName);` |
| Batch completed with partial failures | `LOGGER.warn("Batch completed with {} failures out of {}", failCount, total);` |

---

## ERROR

| Scenario | Sample LOGGER Statement |
|---|---|
| Unhandled exception | `LOGGER.error("Unexpected error processing order {}", orderId, exception);` |
| Operation failed after retries | `LOGGER.error("Failed to send email to {} after {} retries", email, maxRetries, exception);` |
| Dependency unreachable | `LOGGER.error("Could not connect to Redis after {}ms timeout", timeoutMs, exception);` |
| Data validation failure (blocking) | `LOGGER.error("Order {} has invalid total: {}, rejecting", orderId, total);` |
| Failed transaction/payment | `LOGGER.error("Card charge failed for order {}: {}", orderId, failureReason, exception);` |
| Scheduled job failure | `LOGGER.error("Scheduled job '{}' failed: {}", jobName, errorMessage, exception);` |
| Auth/security failure | `LOGGER.error("JWT signature verification failed for request from IP {}", clientIp);` |
| Uncaught 5xx error | `LOGGER.error("Internal server error on {} {}: {}", method, path, exception.getMessage(), exception);` |
| Startup validation failure (fail-fast) | `LOGGER.error("Missing required env var: {}", varName);` — typically precedes `System.exit(1)` |

---

## FATAL / CRITICAL

> **Naming note:** Java's `java.util.logging` doesn't have a native `FATAL` level — `ERROR` is their highest. `FATAL` exists in Log4j/Log4j2. `CRITICAL` is the equivalent in Python's `logging` module (`logging.critical`), syslog (`crit`), and many other ecosystems.

| Scenario | Sample LOGGER Statement |
|---|---|
| Unrecoverable startup failure | `LOGGER.fatal("Database schema version mismatch (expected {}, found {}), cannot start", expected, found);` — followed by `System.exit(1)` |
| Out of memory / resource exhaustion the process can't recover from | `LOGGER.fatal("OutOfMemoryError encountered, process terminating", error);` |
| Corrupted critical state that makes further processing unsafe | `LOGGER.fatal("In-memory ledger checksum failed, halting to prevent data corruption");` |
| Loss of a hard dependency with no fallback path | `LOGGER.fatal("Primary and failover DB both unreachable, shutting down service", exception);` |
| Security breach requiring immediate shutdown | `LOGGER.fatal("Detected tampering with signing keys, shutting down for safety");` |
| Uncaught error in main/bootstrap thread | `LOGGER.fatal("Uncaught exception in main thread, application exiting", throwable);` |

**Key distinction from ERROR:** ERROR means *an operation* failed and the system stays up to handle the next request. FATAL/CRITICAL means *the process itself* can no longer run correctly and is about to exit or crash — it's rare, should almost always precede a shutdown (`System.exit`, container restart, crash), and typically pages someone immediately. If the app keeps serving traffic afterward, it probably wasn't FATAL — use ERROR instead.

---

## DEBUG vs TRACE — the distinction people often blur

| | DEBUG | TRACE |
|---|---|---|
| Frequency | Moderate — key checkpoints | Extreme — every step/iteration |
| Typical use | "Show me what happened in this function" | "Show me literally everything, line by line" |
| Left enabled how often | Sometimes in staging/troubleshooting | Almost never, even in dev, unless deep-diving a specific bug |
| Performance cost | Low-moderate | Can be significant (loops, hot paths) |

---

## ERROR vs FATAL/CRITICAL — the distinction people often blur

| | ERROR | FATAL / CRITICAL |
|---|---|---|
| Scope of impact | One operation/request fails | The whole process is compromised |
| Process survives? | Yes — keeps serving other requests | Usually no — exits, crashes, or is about to be restarted |
| Frequency | Can happen repeatedly, in isolation | Should be rare — near-zero in a healthy system |
| Typical follow-up | Retry, alert, move on | `System.exit()`, orchestrator restart, page on-call immediately |
| Example | Payment API call failed for one order | DB unreachable at startup, service can't come up at all |

## Full Level Ordering

Most frameworks treat levels as a strict severity hierarchy (lowest → highest); setting a threshold enables that level and everything above it:

```
TRACE < DEBUG < INFO < WARN < ERROR < FATAL/CRITICAL
```

Setting the logger to `WARN`, for example, silently drops TRACE/DEBUG/INFO and only emits WARN, ERROR, and FATAL.

---

## Key Formatting Habits

1. **Use parameterized logging** (`{}` placeholders), not string concatenation — avoids the cost of building strings when the log level is disabled.

   ```java
   // Bad — always builds the string
   LOGGER.debug("User: " + userId + ", action: " + action);

   // Good — lazy evaluation
   LOGGER.debug("User: {}, action: {}", userId, action);
   ```

2. **Pass the exception as the last argument** — SLF4J auto-detects a trailing `Throwable` and logs the full stack trace.

   ```java
   LOGGER.error("Failed to process order {}", orderId, exception); // stack trace included
   ```

3. **Use MDC for correlation IDs** (request tracing across a call chain):

   ```java
   MDC.put("requestId", requestId);
   LOGGER.info("Processing request");
   // ... later
   MDC.clear();
   ```

4. **Never log secrets** — passwords, tokens, full credit card numbers, PII — mask or omit regardless of level.

5. **Set level by environment** — typically DEBUG/TRACE in dev, INFO in staging/prod (WARN/ERROR/FATAL always on), with DEBUG/TRACE enabled temporarily in prod only for targeted troubleshooting.

6. **Don't log and swallow** — if you catch an exception and log it as ERROR, either re-raise, handle it meaningfully, or explicitly note why it's safe to ignore.

7. **Security/audit events** (e.g., password changes, permission changes) are often routed to a **separate audit log**, not just INFO — check your compliance requirements.

8. **Prefer structured (JSON/key-value) logging in production** — plain text is fine locally, but structured fields make logs queryable in aggregators (ELK, Datadog, CloudWatch) without brittle regex parsing.

9. **Sample or rate-limit high-frequency log lines** — TRACE/DEBUG in hot loops or high-QPS paths can overwhelm storage/cost even when the level is enabled; consider sampling (e.g., log 1 in N) or aggregating counts instead of logging every occurrence.

10. **FATAL/CRITICAL should almost always be paired with a process exit or crash** — logging at this level without actually terminating (or triggering a restart) undermines the signal; downstream alerting often treats a FATAL log line as "the service is down."

---

## Quick Decision Test

> "If this log line appeared 10,000 times in a day, would that be normal?"

- Yes, and it's fine-grained detail → **TRACE**
- Yes, and it's a checkpoint I'd want while debugging → **DEBUG**
- Yes, and it's just normal operation → **INFO**
- No, and it means something's degraded → **WARN**
- No, and something broke, but the app is still running → **ERROR**
- No, and the app can no longer run correctly → **FATAL/CRITICAL**
