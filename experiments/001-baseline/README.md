# Experiment 001 - Baseline

## Objective

My goal is to establish the baseline network behaviour .I want to understand how the network normally looks like before introducing any artificial network impairments.

## Topology

```text
Client                              Router                           Server
10.0.1.2                       10.0.1.1 /10.0.2.1                    10.0.2.2
   |                                   |                                |
   +-----------------------------------+--------------------------------+
            eth1                                     eth2                      

```

## Configuration

The client has the ip address `10.0.1.2` and connects to the router through the virtual eth1 link.The router's eth1 interface has the address `10.0.1.1`,making it the client's gateway.
The router's eth2 interface has the address `10.0.2.1`. The server has the address `10.0.2.2` and uses `10.0.2.1` as its gateway
No artificial delay, packet loss, jitter, or bandwidth limitation was applied during this experiment.

##Tests 

I used `ping` to measure the baseline round-trip time(RTT) between the client and the server.
I also used `traceroute` to verify the path taken by the packets and confirm that traffic passes through the router.

## Results 

The baseline ping test produced an average RTT of 0.161ms. There was 0% packet loss.
The traceroute showed that traffic travelled through the router before reaching the server. The first hop was the router and the second hop was the server at `10.0.2.2`.

## What I learned 

I learned that the baseline network has very low latency .This makes sense because the network consists of virtual links between containers inside a virtual machine rather thana physical network.

I also learned that 0% packet loss means that every ICMP packet sent during this particular test recieved a reply

Traceroute showed me that the traffic passes through the router before reaching the server, confirming the path we intended to create.
