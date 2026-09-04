# Experiment 002 - Bandwidth Limitation

## Objective

The objective of this experiment is to investigate how limiting the transmission
rate of a network link affects throughput, queueing, and packet loss.

## Background

A packet's transmission delay can be expressed as:

d_trans = L / R

Where:

- L is the packet length in bits.
- R is the transmission rate in bits per second.
- d_trans is the transmission delay.

Traffic intensity is expressed as:

I = La / R

Where:

- L is the packet length in bits.
- a is the average packet arrival rate.
- R is the transmission rate.

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
2. Run a control throughput test.
3. Apply a bandwidth limitation to router `eth2`.
4. Repeat the same throughput test.
5. Inspect traffic-control statistics.
6. Compare the control and limited conditions.

## Configuration

### Control configuration

Check the current queueing discipline:

```bash
sudo docker exec clab-delay-lab-router tc qdisc show dev eth2
```

Expected baseline:

```text
qdisc noqueue ...
```

### Bandwidth limitation

The router's `eth2` interface will be configured with a Token Bucket Filter
(TBF) to limit the outgoing rate.

```bash
sudo docker exec clab-delay-lab-router tc qdisc add dev eth2 root tbf rate 1mbit burst 32kb latency 400ms
```

## Tests

### Control test

Run `iperf3` without the artificial bandwidth limitation:

```bash
...
```

### Limited test

Run the same test after applying the 1 Mbit/s limitation:

```bash
...
```

### Traffic-control statistics

Inspect the qdisc statistics:

```bash
sudo docker exec clab-delay-lab-router tc -s qdisc show dev eth2
```

## Results

### Control

The raw results from the control test are available in:
`../../results/002-bandwidth/control/`

### Bandwidth-limited

The raw results from the bandwidth-limited test are available in:
`../../results/002-bandwidth/limited/`

| Condition | Throughput | Packet Loss | Average RTT |
|---|---:|---:|---:|
| Control | ... | ... | ... |
| Bandwidth-limited | ... | ... | ... |

## Analysis

The control and bandwidth-limited measurements will be compared to determine
how reducing the transmission rate affected throughput, queueing, and packet
loss.

## What I Learned

To be completed after the experiment.
