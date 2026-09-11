# Experiment 004 — Queueing Discipline Comparison

## Objective

Compare how two different Linux queueing disciplines handle the same overloaded UDP traffic when constrained by the same 1 Mbit/s bottleneck:

* FIFO (`pfifo`)
* FQ-CoDel (`fq_codel`)

The experiment focuses on the trade-off between **packet loss, queueing behavior, and traffic completion time**.

---

## Hypothesis

When offered traffic that exceeds the available link capacity, different queueing disciplines should handle the excess traffic differently.

The expectation was that FIFO would primarily buffer packets until its queue filled, while FQ-CoDel would actively manage queueing and prevent excessive queue buildup.

The experiment was designed to determine how this difference appears in measured traffic behavior.

---

## Controlled Variables

The following conditions were kept constant between both tests:

| Variable             | Value                     |
| -------------------- | ------------------------- |
| Topology             | Client → Router → Server  |
| Bottleneck interface | Router `eth2`             |
| HTB rate             | 1 Mbit/s                  |
| UDP offered rate     | 1.5 Mbit/s                |
| Test duration        | 10 seconds                |
| Traffic generator    | `iperf3`                  |
| Transport protocol   | UDP                       |
| Packet workload      | Same for both tests       |
| Queueing discipline  | **Only variable changed** |

The traffic therefore exceeded the configured bottleneck by approximately 50%, creating sustained congestion.

---

## Topology

```text
Client
10.0.1.2
   |
   |
Router
eth2
10.0.2.1
   |
   v
HTB 1:
  |
  └── class 1:10
        rate = 1 Mbit/s
        |
        ├── FIFO
        │
        └── FQ-CoDel
               |
               v
            Server
           10.0.2.2
```

Only one child queue was attached to the HTB class during each test.

### FIFO configuration

```text
HTB
 └── class 1:10
      rate = 1 Mbit/s
       └── pfifo
```

### FQ-CoDel configuration

```text
HTB
 └── class 1:10
      rate = 1 Mbit/s
       └── fq_codel
```

---

## Methodology

For each test, the router was configured with an HTB class limiting the traffic to 1 Mbit/s.

The same UDP workload was then generated from the client:

```bash
iperf3 -c 10.0.2.2 -u -b 1.5M -t 10
```

Router-side queueing statistics were collected using:

```bash
tc -s qdisc show dev eth2
```

and:

```bash
tc -s class show dev eth2
```

The queueing configuration was removed and recreated between tests so that statistics from one queueing discipline did not contaminate the other.

---

# Results

## FIFO

The FIFO test offered approximately 1.50 Mbit/s for 10 seconds.

### iperf3

| Metric                   |      Result |
| ------------------------ | ----------: |
| Sender bitrate           | 1.50 Mbit/s |
| Sender datagrams         |         199 |
| Receiver bitrate         |  996 Kbit/s |
| Receiver loss            |      **0%** |
| Receiver jitter          |   25.538 ms |
| Receiver completion time | **15.10 s** |

### Router statistics

```text
qdisc htb 1: root
 Sent 3781477 bytes 453 pkt
 dropped 0
 overlimits 431

qdisc pfifo 10: parent 1:10 limit 1000p
 Sent 3781477 bytes 453 pkt
 dropped 0
```

### Observation

FIFO recorded **zero packet drops** during this workload.

Although the sender stopped transmitting after 10 seconds, the receiver continued receiving traffic until approximately 15.10 seconds.

This indicates that excess traffic was being **buffered and subsequently drained** rather than immediately discarded.

The 1 Mbit/s bottleneck could not transmit the offered 1.5 Mbit/s in real time, so the excess traffic accumulated in the queue.

The `overlimits` value reported by HTB indicates that the configured rate was being enforced. It does **not** represent packet loss.

---

## FQ-CoDel

The same 1.5 Mbit/s UDP workload was run through the same 1 Mbit/s HTB class, with FQ-CoDel replacing FIFO.

### iperf3

| Metric                   |      Result |
| ------------------------ | ----------: |
| Sender bitrate           | 1.50 Mbit/s |
| Sender datagrams         |         199 |
| Receiver bitrate         |  996 Kbit/s |
| Receiver loss            |     **33%** |
| Receiver jitter          |   28.912 ms |
| Receiver completion time | **10.01 s** |

### Router statistics

```text
qdisc htb 1: root
 Sent 1283345 bytes 156 pkt
 dropped 64
 overlimits 146

qdisc fq_codel 10: parent 1:10
 Sent 1283345 bytes 156 pkt
 dropped 64
```

FQ-CoDel reported:

```text
target 5ms
interval 100ms
flows 1024
```

It also reported:

```text
new_flow_count 9
old_flows_len 1
```

### Observation

Unlike FIFO, FQ-CoDel dropped packets during the overloaded workload.

The receiver therefore reported **33% UDP datagram loss**, while the receiver completed at approximately the same time that the sender stopped transmitting.

This is consistent with FQ-CoDel actively managing the queue rather than allowing the excess traffic to remain buffered until it can eventually be transmitted.

---

# Comparison

| Metric              |        FIFO |    FQ-CoDel |
| ------------------- | ----------: | ----------: |
| Offered rate        | 1.50 Mbit/s | 1.50 Mbit/s |
| Bottleneck          |    1 Mbit/s |    1 Mbit/s |
| Receiver bitrate    |  996 Kbit/s |  996 Kbit/s |
| Receiver loss       |      **0%** |     **33%** |
| Receiver jitter     |   25.538 ms |   28.912 ms |
| Receiver completion | **15.10 s** | **10.01 s** |
| qdisc drops         |       **0** |      **64** |
| HTB overlimits      |         431 |         146 |

The most important result is that the two queueing disciplines delivered approximately the **same bottleneck throughput**, but handled the excess traffic differently.

### FIFO

```text
Excess traffic
      ↓
Queue it
      ↓
Keep packets waiting
      ↓
Transmit them as capacity becomes available
      ↓
Longer completion time
```

### FQ-CoDel

```text
Excess traffic
      ↓
Manage queue
      ↓
Discard packets during congestion
      ↓
Prevent the queue from simply absorbing the excess
      ↓
Shorter completion time
```

---

# Interpretation

The experiment demonstrates that a queueing discipline does not increase the physical or configured capacity of a link.

Both tests were constrained to approximately 1 Mbit/s, while the sender attempted to transmit at 1.5 Mbit/s.

The difference was **what happened to the excess traffic**.

FIFO was able to absorb the workload without dropping packets because its queue had sufficient capacity for this particular test. The cost was that packets remained in the system after the sender had stopped transmitting.

FQ-CoDel instead dropped a significant amount of traffic during congestion. This resulted in higher observed packet loss but avoided the same prolonged post-test draining period seen with FIFO.

Therefore, "better" depends on what the network is trying to optimize. Preserving packets at the cost of increased waiting time is different from actively controlling queue buildup at the cost of discarding some packets.

---

## Important Measurement Note

The `iperf3` **jitter** value should not be interpreted as a direct measurement of queueing delay.

Likewise, the `new_flows_len` and `old_flows_len` values reported by FQ-CoDel describe its internal flow queues and are not direct measurements of end-to-end packet delay.

A dedicated latency measurement, such as timestamped packet measurements or RTT measurements under load, would be required to quantify queueing delay directly.

---

## Further Observation — Queue Size

The FIFO test also illustrates a broader relationship between buffering and packet loss.

A sufficiently large queue can absorb excess traffic for a period of time, reducing immediate packet loss but increasing the amount of traffic waiting in the queue.

Once the available queue space is exhausted, newly arriving packets can no longer be stored and must be dropped.

This experiment therefore provides an initial demonstration of the **buffering-versus-loss trade-off** without requiring a separate experiment specifically for FIFO queue size.

---

## Limitations

This experiment has several limitations:

1. Only one UDP flow was tested.
2. The offered load was fixed at 1.5 Mbit/s.
3. The bottleneck was configured at 1 Mbit/s.
4. The experiment does not directly measure queue length over time.
5. `iperf3` jitter is not equivalent to queueing delay.
6. The results describe this specific containerized Linux environment and configuration.
7. The FIFO and FQ-CoDel results should not be generalized to every possible workload or network topology.

In particular, FQ-CoDel is designed to provide benefits that become more apparent with multiple competing flows and persistent queueing. A single-flow UDP test does not exercise all of its capabilities.

---

## Conclusion

Experiment 004 showed that queueing disciplines can produce substantially different behavior even when the underlying bandwidth constraint is identical.

FIFO preserved the packets in this test, resulting in **0% observed UDP loss**, but the receiver continued processing traffic for approximately **5 seconds after the sender stopped**.

FQ-CoDel delivered approximately the same bottleneck throughput but dropped **33% of the UDP datagrams**, resulting in a receiver completion time close to the original 10-second sending period.

The experiment therefore demonstrates a fundamental networking trade-off:

> **A network can handle congestion by allowing packets to wait, or by actively controlling the queue and discarding traffic when necessary.**

The queueing discipline determines how that trade-off is managed.
