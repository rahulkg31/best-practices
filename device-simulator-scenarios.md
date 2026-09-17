# Device Data Simulator Scenarios

Focused set of 10 scenarios with examples, for building a device-data simulator.

---

## 1. Fixed synchronized interval
Every device sends at the same timestamp on a fixed period.

**Example:**
```
Interval = 10s
Device A: 10s, 20s, 30s, 40s
Device B: 10s, 20s, 30s, 40s
Device C: 10s, 20s, 30s, 40s
```
All devices emit at identical timestamps, cycle after cycle.

---

## 2. Uniform distribution (fleet-aggregate)
Data arrives evenly spread across each time window — not because any single device is random, but because many periodic devices have uncorrelated boot offsets.

**Example (deterministic even spacing — reliable even with few devices):**
```
N devices, window = 10s
offset(i) = i * (10 / N)

5 devices → offsets: 0.0s, 2.0s, 4.0s, 6.0s, 8.0s
```
```
Device A (offset 0.0s): 0.0s, 10.0s, 20.0s, 30.0s ...
Device B (offset 2.0s): 2.0s, 12.0s, 22.0s, 32.0s ...
Device C (offset 4.0s): 4.0s, 14.0s, 24.0s, 34.0s ...
Device D (offset 6.0s): 6.0s, 16.0s, 26.0s, 36.0s ...
Device E (offset 8.0s): 8.0s, 18.0s, 28.0s, 38.0s ...

Aggregate arrivals in window 0–10s: 0.0s, 2.0s, 4.0s, 6.0s, 8.0s
Aggregate arrivals in window 10–20s: 10.0s, 12.0s, 14.0s, 16.0s, 18.0s
```
Each 10s window is evenly covered regardless of device count.

---

## 3. Jitter range (± ms around target interval)
Devices target a fixed interval but arrive slightly offset from each other and drift over time.

**Example:**
```
Target interval = 10s, jitter = ±500ms

Device A: 10.0s, 20.1s, 29.9s, 40.3s
Device B: 10.4s, 20.7s, 30.2s, 39.8s
Device C:  9.8s, 19.9s, 30.5s, 40.6s
```
Each device's actual send time = target + random(-500ms, +500ms).

---

## 4. Partial backlog
Only some messages are delayed, not the whole stream.

**Example:**
```
10s  → message sent on time
20s  → message sent on time
30s  → message delayed, arrives at 34s
40s  → message delayed, arrives at 44s
50s  → message sent on time
60s  → message sent on time
```
Out of 6 scheduled sends, 2 are late by ~4s while the rest are unaffected — reflecting a device with intermittent (not total) connectivity trouble.

---

## 5. Full backlog / delayed burst
No data for a period, then all buffered data arrives in a single burst.

**Example:**
```
0–30s   normal sends every 10s (0s, 10s, 20s, 30s)
30–60s  device offline, no data sent, but events are buffered locally
60s     reconnects → buffered events for 30s, 40s, 50s, 60s all arrive at once
60s+    normal sends resume every 10s
```
At t=60s, the ingestion pipeline receives 4 events simultaneously that were generated 10–30s earlier.

---

## 6. Device dropout / offline
Some devices in a fleet suddenly stop sending; a subset never come back.

**Example:**
```
Fleet = 100 devices, all sending every 10s

At t=120s:
  85 devices → continue normally
  10 devices → stop sending, resume at t=240s (2 min later)
  5 devices  → stop sending and never resume
```

---

## 7. Duplicate rate
The same message is sent more than once due to failed/delayed acknowledgments.

**Example:**
```
10.0s → message_id=A123 (payload X)
10.2s → message_id=A123 (payload X)   ← retry, ack was lost
10.5s → message_id=A123 (payload X)   ← retry, ack still lost
15.0s → message_id=A124 (payload Y)   ← next message, normal
```
Duplicate rate can be modeled as a probability (e.g., 3% of messages get resent 1–2 extra times).

---

## 8. Out-of-order arrival
Messages arrive at the ingestion endpoint in a different order than they were generated.

**Example:**
```
Generated:  10s, 15s, 20s, 25s, 30s
Arrived at server:
  10s  → arrives at t=10.1s
  20s  → arrives at t=20.1s
  15s  → arrives at t=20.3s   ← generated at 15s, but arrives late, after the 20s message
  25s  → arrives at t=25.1s
  30s  → arrives at t=30.1s
```
The 15s-timestamped message is delayed in transit and lands after the 20s message.

---

## 9. Payload size distribution
Not every event carries the same amount of data.

**Example:**
```
Heartbeat event   = 1 KB    (95% of events)
Status/log event  = 100 KB  (4% of events)
Media/snapshot event = 1 MB (1% of events, e.g., triggered by motion detection)
```
```
10s → 1 KB   heartbeat
20s → 1 KB   heartbeat
30s → 100 KB status log
40s → 1 KB   heartbeat
50s → 1 MB   motion-triggered snapshot
60s → 1 KB   heartbeat
```

---

## 10. Periodic spike
Mostly normal traffic, with recurring heavier bursts on a schedule or clock-aligned trigger.

**Example (scheduled batch spike):**
```
Every 10th heartbeat includes a full state sync (larger payload, more devices reporting)

Heartbeats 1–9:  normal, 1 KB each
Heartbeat 10:    full sync, 50 KB, sent by all devices simultaneously
Heartbeats 11–19: normal, 1 KB each
Heartbeat 20:    full sync, 50 KB, sent by all devices simultaneously
```

**Example (clock-boundary-aligned spike):**
```
Devices normally send every 10s, but many round their internal schedule to the top of the minute.

:00:00 → 400 of 1,000 devices report (clock-aligned cluster)
:00:10 → 60 devices report
:00:20 → 55 devices report
:00:30 → 58 devices report
:00:40 → 62 devices report
:00:50 → 65 devices report
:01:00 → 410 of 1,000 devices report (next clock-aligned cluster)
```

