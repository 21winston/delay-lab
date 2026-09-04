# Experiment 002 - Bandwidth Limitation and Congestion

## Objective

The objective of this experiment is to investigate how limiting the transmission
rate of a network link affects throughput, queueing, and packet loss.

## Background

A packet's transmission delay can be expressed as:

```text
d_trans = L / R
```

Where:

* `L` is the packet length in bits.
* `R` is the transmission rate in bits per second.
* `d_trans` is the transmission delay.

Traffic intensity is expressed as:

```text
I = La / R
```

Where:

* `L` is the packet length in bits.
* `a` is the average packet arrival rate.
* `R` is the transmission rate.

When the offered traffic exceeds the capacity of the link, packets may have to
wait in a queue. If the available buffering becomes insufficient, packets may
eventually be dropped.

## Hypothesis

If the transmission rate of the router-to-server link is limited to a lower
rate than the traffic being offered, the receiver should observe reduced
throughput. Queueing should increase as the offered traffic approaches or
exceeds the available transmission capacity.

## Topology

```text
Client                  Router                  Server
10.0.1.2                10.0.1.1               10.0.2.2
   |                        |                       |
   +-------- eth1 ----------+---------- eth2 -------+
                                      ^
                                      |
                              bandwidth limit
```

## Methodology

The experiment consists of a control measurement followed by a bandwidth-
limited measurement.

1. Verify that no artificial traffic control is applied.
2. Run a control UDP throughput test.
3. Apply a 1 Mbit/s bandwidth limitation to router `eth2`.
4. Repeat the same UDP throughput test.
5. Inspect traffic-control statistics.
6. Compare the control and limited conditions.

The UDP traffic was intentionally offered at `1.5 Mbit/s`, which is greater
than the configured `1 Mbit/s` transmission rate of the bandwidth-limited
condition.

## Configuration

### Control configuration

The control condition was verified before applying the bandwidth limitation.

```bash
sudo docker exec clab-delay-lab-router tc qdisc show dev eth2
```

Expected baseline:

```text
qdisc noqueue ...
```

### Bandwidth limitation

The router's `eth2` interface was configured with a Token Bucket Filter (TBF)
to limit the outgoing rate.

```bash
sudo docker exec clab-delay-lab-router tc qdisc add dev eth2 root tbf rate 1mbit burst 32kb latency 400ms
```

The resulting configuration was verified with:

```bash
sudo docker exec clab-delay-lab-router tc qdisc show dev eth2
```

The configured transmission rate was `1 Mbit/s`.

## Tests

### Control test

The control test used UDP traffic with an offered rate of `1.5 Mbit/s` for
10 seconds.

```bash
iperf3 -c 10.0.2.2 -u -b 1.5M -t 10
```

### Limited test

The same UDP test was repeated after applying the `1 Mbit/s` TBF limitation.

```bash
iperf3 -c 10.0.2.2 -u -b 1.5M -t 10
```

### Traffic-control statistics

The qdisc statistics were inspected after the limited test:

```bash
sudo docker exec clab-delay-lab-router tc -s qdisc show dev eth2
```

## Results

### Control

The raw results from the control test are available in:

`../../results/002-bandwidth/control/`

The control test achieved approximately `1.50 Mbit/s` at the receiver with
`0%` packet loss and `0.041 ms` jitter.

### Bandwidth-limited

The raw results from the bandwidth-limited test are available in:

`../../results/002-bandwidth/limited/`

The bandwidth-limited test achieved approximately `1.02 Mbit/s` at the receiver.
Packet loss was `28%`, and reported UDP jitter increased to `25.048 ms`.

### Comparison

| Condition         | Offered Rate | Receiver Throughput | Packet Loss | UDP Jitter |
| ----------------- | -----------: | ------------------: | ----------: | ---------: |
| Control           |  1.50 Mbit/s |         1.50 Mbit/s |          0% |   0.041 ms |
| Bandwidth-limited |  1.50 Mbit/s |         1.02 Mbit/s |         28% |  25.048 ms |

The TBF statistics after the limited test showed:

```text
Sent 1349481 bytes 163 pkt (dropped 57, overlimits 472 requeues 0)
backlog 0b 0p requeues 0
```

The `tc` statistics represent counters accumulated by the TBF qdisc since it
was created and therefore provide router-side evidence of the traffic-control
behavior rather than an independent experiment-wide packet-loss measurement.

## Analysis

The control condition successfully transmitted the offered `1.5 Mbit/s`
without packet loss. This established that the underlying virtual topology
could carry the offered traffic without the artificial bandwidth limitation.

After limiting router `eth2` to `1 Mbit/s`, the receiver throughput decreased
to approximately `1.02 Mbit/s`, which is close to the configured transmission
limit. The offered traffic remained at `1.5 Mbit/s`, meaning that more traffic
was being offered than the link could transmit.

The excess traffic resulted in queueing and packet loss. The receiver reported
`28%` UDP datagram loss and `25.048 ms` of jitter, compared with `0%` loss and
`0.041 ms` jitter during the control test.

The TBF statistics also recorded dropped packets and overlimit events,
providing additional evidence that the traffic exceeded the configured
transmission capacity.

The limited test also continued receiving traffic until approximately `10.52 seconds`, despite the sender stopping at 10 seconds. This indicates that some packets continued to arrive after the sender had finished transmitting, consistent with the effects of queueing introduced by the bandwidth limitatio

## What I Learned

This experiment demonstrated how a transmission-rate bottleneck affects UDP
traffic. Offering `1.5 Mbit/s` to a link limited to `1 Mbit/s` caused the
receiver throughput to become constrained by the bottleneck, while also
introducing packet loss and increased jitter.

The experiment also demonstrated the difference between application-level
measurements from `iperf3` and router-side statistics from `tc`. `iperf3`
measures what happened to the UDP traffic from the sender and receiver's
perspective, while `tc` reports statistics accumulated by the configured
queueing discipline.
