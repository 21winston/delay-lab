# Experiment 003 — TCP vs UDP Under a Bandwidth Bottleneck

## Objective

Compare how TCP and UDP behave when transmitting through the same 1 Mbit/s bandwidth bottleneck.

The experiment investigates how the two transport protocols respond when the offered traffic exceeds the available link capacity.

## Hypothesis

UDP will continue transmitting at its configured sending rate, resulting in packet loss when the offered rate exceeds the 1 Mbit/s bottleneck.

TCP will detect network conditions through acknowledgements and retransmissions and adapt its sending behavior through congestion control. Rather than exposing packet loss directly to the application, TCP will retransmit lost data to provide reliable delivery.

## Experimental Setup

The same network topology and traffic-control configuration were used for both protocols.

```text
Client (10.0.1.2)                         Server (10.0.2.2)
       │                                         │
       └────── Router (10.0.1.1/10.0.2.1) ─────┘
                    │
                 eth2
                    │
              TBF bottleneck
              1 Mbit/s
```

### Controlled variables

| Parameter             | Value                    |
| --------------------- | ------------------------ |
| Topology              | Client → Router → Server |
| Bottleneck interface  | Router `eth2`            |
| TBF rate              | 1 Mbit/s                 |
| TBF burst             | 32 KB                    |
| TBF latency parameter | 400 ms                   |
| Test duration         | 10 seconds               |

The transport protocol was the primary variable.

## Methodology

### UDP

UDP was configured to generate traffic at 1.5 Mbit/s:

```bash
iperf3 -c 10.0.2.2 -u -b 1.5M -t 10
```

### TCP

TCP was tested for the same duration without specifying a sending rate:

```bash
iperf3 -c 10.0.2.2 -t 10
```

TCP's sending rate was controlled by its congestion-control mechanisms rather than by an explicitly configured application rate.

## Results

### UDP

The UDP experiment produced:

| Metric           |      Result |
| ---------------- | ----------: |
| Offered bitrate  | 1.50 Mbit/s |
| Receiver bitrate | 1.02 Mbit/s |
| Datagram loss    |         28% |
| Jitter           |   25.048 ms |

The UDP sender continued attempting to transmit at 1.5 Mbit/s despite the 1 Mbit/s bottleneck. The resulting excess traffic led to packet drops at the bottleneck.

### TCP

The TCP experiment produced:

| Metric           |      Result |
| ---------------- | ----------: |
| Sender bitrate   | 1.36 Mbit/s |
| Receiver bitrate | 1.00 Mbit/s |
| Retransmissions  |          15 |

TCP's per-second bitrate varied considerably during the experiment, including intervals with zero reported throughput. The congestion window (`Cwnd`) also changed throughout the test.

The final receiver throughput was approximately 1.00 Mbit/s, matching the configured bottleneck rate.

### Router-side TBF statistics

The TBF statistics captured immediately after the TCP experiment were:

```text
Sent 1416638 bytes 181 pkt (dropped 15, overlimits 505 requeues 0)
backlog 0b 0p requeues 0
```

The TBF recorded 15 dropped packets and 505 overlimit events during the experiment.

## Interpretation

The results demonstrate different transport-layer responses to the same bandwidth constraint.

UDP continued transmitting at its configured 1.5 Mbit/s offered rate. Because the bottleneck could only forward approximately 1 Mbit/s, excess traffic resulted in packet loss. The application therefore observed 28% datagram loss.

TCP behaved differently. It does not maintain a fixed application sending rate in the same way as the UDP test. Instead, TCP uses congestion-control mechanisms and retransmissions to respond to network conditions.

The TCP test recorded 15 retransmissions while the TBF recorded 15 dropped packets during the same experiment. These values should not be interpreted as proving a one-to-one relationship between individual dropped packets and retransmission events.

Despite the packet drops, the TCP receiver obtained approximately 1.00 Mbit/s. TCP therefore provided reliable byte-stream delivery while adapting its transmission behavior to the constrained network.

## Key Observation

The same bottleneck produced different observable behavior depending on the transport protocol:

```text
UDP
1.5 Mbit/s offered
        ↓
1 Mbit/s bottleneck
        ↓
Excess traffic
        ↓
Packet loss
        ↓
28% datagram loss

TCP
Application sends data
        ↓
Congestion control
        ↓
Network loss detected
        ↓
Retransmission + rate adaptation
        ↓
~1 Mbit/s received
```

This demonstrates that **bandwidth limitation is a network condition, while the transport protocol determines how the resulting congestion and packet loss are handled at the transport layer.**

## Evidence

Raw experimental outputs are stored in:

```text
results/003-tcp-vs-udp/tcp/iperf3.txt
results/003-tcp-vs-udp/tcp/tc-stats.txt
```

The UDP results from Experiment 002 are stored in:

```text
results/002-bandwidth/control/iperf3.txt
results/002-bandwidth/limited/iperf3.txt
```

## Limitations

This experiment uses a software-emulated bottleneck inside a Containerlab environment rather than a physical network link.

The router-side `tc` statistics provide evidence of qdisc behavior, while `iperf3` provides transport-level measurements. Packet-level analysis with Wireshark or `tcpdump` could provide additional evidence of individual packet drops, retransmissions, and TCP acknowledgements.

## Conclusion

The experiment confirmed that TCP and UDP respond differently when subjected to the same bandwidth bottleneck.

UDP maintained its configured offered rate and experienced significant datagram loss when the offered traffic exceeded the bottleneck capacity.

TCP responded through congestion-control behavior and retransmissions, resulting in approximately 1 Mbit/s of received throughput despite packet loss within the network.

The experiment therefore demonstrates the interaction between **network-layer capacity constraints and transport-layer congestion-control mechanisms**.
