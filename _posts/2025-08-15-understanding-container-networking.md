---
title: "Understanding Container Networking: From Bridges to Overlay Networks"
date: 2025-08-15 10:00:00 -0700
categories: [Infrastructure, Containers]
tags: [docker, networking, linux, containers]
---

Container networking can feel like magic until you peel back the layers. In this post, I'll walk through how container networking actually works under the hood, from Linux bridges to overlay networks.

## The Basics: Network Namespaces

At the heart of container networking lies the Linux network namespace. Each container gets its own isolated network stack -- its own interfaces, routing tables, and iptables rules.

```bash
# Create a network namespace
ip netns add container1

# List namespaces
ip netns list

# Execute a command inside the namespace
ip netns exec container1 ip addr
```

When you create a namespace, it starts with only a loopback interface. To communicate with the outside world, we need to create virtual ethernet (veth) pairs.

## Virtual Ethernet Pairs

A veth pair acts like a virtual cable connecting two network namespaces. One end goes inside the container's namespace, the other stays in the host namespace (or connects to a bridge).

```bash
# Create a veth pair
ip link add veth0 type veth peer name veth1

# Move one end into the container namespace
ip link set veth1 netns container1

# Configure IP addresses
ip addr add 172.18.0.1/24 dev veth0
ip netns exec container1 ip addr add 172.18.0.2/24 dev veth1

# Bring interfaces up
ip link set veth0 up
ip netns exec container1 ip link set veth1 up
```

## Linux Bridges

When you have multiple containers that need to talk to each other, you need a bridge. Docker's default `bridge` network creates a Linux bridge called `docker0`.

```bash
# Create a bridge
ip link add br0 type bridge
ip link set br0 up

# Attach veth interfaces to the bridge
ip link set veth0 master br0
```

The bridge works like a virtual switch -- it learns MAC addresses and forwards frames between connected interfaces.

## Overlay Networks

Things get interesting in multi-host setups. Overlay networks use VXLAN (Virtual Extensible LAN) to encapsulate Layer 2 frames inside UDP packets, allowing containers on different hosts to communicate as if they were on the same network.

```
Host A                          Host B
┌─────────────┐                ┌─────────────┐
│ Container 1 │                │ Container 2 │
│ 10.0.0.2    │                │ 10.0.0.3    │
└──────┬──────┘                └──────┬──────┘
       │                              │
┌──────┴──────┐                ┌──────┴──────┐
│   vxlan0    │                │   vxlan0    │
│  VTEP       │───── UDP ─────│  VTEP       │
└──────┬──────┘                └──────┴──────┘
       │                              │
   eth0:192.168.1.10           eth0:192.168.1.11
```

Docker Swarm and Kubernetes both use variations of this approach to enable cross-host container communication.

## Key Takeaways

1. **Network namespaces** provide isolation at the kernel level
2. **Veth pairs** connect namespaces together
3. **Bridges** enable multi-container communication on a single host
4. **Overlay networks** (VXLAN) extend this to multi-host setups

Understanding these primitives makes debugging container networking issues much more tractable. Instead of guessing, you can inspect each layer systematically.
