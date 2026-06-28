---
title: "The OSI Model From Layer 1 to 4"
description: "The most useful path for understanding normal network traffic"
publishDate: "2026-06-28"
tags: ["linux", "networking"]
---
The OSI model is a framework created by ISO that standardised how computer systems communicate. It is built in layers from the bottom up, layer 1 to 7.

In this post I will only be explaining layers 1 to 4. This will explain the most useful path for understanding normal network traffic.

The first four layers answer these questions:

- Layer 1: Can bits move?
- Layer 2: Can local devices talk?
- Layer 3: Can traffic reach the right network and host?
- Layer 4: Can traffic reach the right application?

## Layer 1: The Physical Layer

Before anything can be done, there needs to be a way to transmit and receive bits from one place to another. This is where layer 1 begins.

Layer 1 is responsible for sending and receiving bits over some kind of link or medium.

Common examples of the physical layer are:

- Physical Ethernet cables
- WiFi
- NIC hardware

Although a `veth-pair` is not physical, it can be treated as a virtual cable. It represents a link between two network namespaces. This makes it useful when learning Docker or Kubernetes networking because containers still need a link, even when that link is virtual instead of a physical cable.

Layer 1 gives us a link, but a link on its own is not enough. Devices still need a way to communicate on that same local link.

## Layer 2: The Data Link Layer

Once there is a link, connected devices need a way to communicate as they are directly attached, Layer 2 ensures that the data is transferred between these devices that are directly reachable on the same link. It works by sending data frames between MAC addresses.

In Ethernet, a frame uses a source MAC address and destination MAC address so that switches or bridges know where to forward it.

An Ethernet frame contains:

- Source MAC address
- Destination MAC address
- Type of payload
- Payload/data
- Error-checking trailer

The important thing is that layer 2 is local. It is not about reaching every device on every network. It is about moving data across the same local link.

For example, if two devices are on the same local link, they can communicate at layer 2. If traffic needs to go somewhere outside of that local network, layer 2 is no longer enough.

A frame can move data locally, but it does not tell us how to reach a different network.

For that, we need layer 3.

## Layer 3: The Network Layer

Now there is a new problem, how can a device reach another device beyond the local link? Devices in the same subnet can communicate locally, but traffic to another subnet needs a route to another network. Layer 3 routes traffic between networks by using IP addresses to identify devices across networks.

At layer 3 you would ask questions like:

- Which subnet or IP network are we on?
- Which hosts are on this network?
- If the destination is outside of my network, where should I send the packet next?

### An IPv4 Example

```text
10.1.1.2
```

This is an IP address at Layer 3, but the IP address alone does not tell us which subnet it belongs to. To know the subnet, an IP address must be paired with a CIDR prefix.

For example:

```text
10.1.1.2/24
```

IPv4 addresses have 32 bits total.

The `/24` means:

```text
24 bits = network
8 bits  = host
```

This split is important because it lets you work out two things:

- The network address
- The host addresses

A network address identifies the subnet, not a usable host.

For example:

```text
10.1.1.2/24
```

belongs to:

```text
10.1.1.0/24
```

The `10.1.1.0` address represents the whole subnet. It is not normally assigned to a device because it identifies the network itself.

In that subnet, the host addresses are usually:

```text
10.1.1.1 - 10.1.1.254
```

The last address is the broadcast address:

```text
10.1.1.255
```

Once you can identify the network address and host range, you can decide whether another destination is local or remote.

For example:

```text
10.1.1.2/24
10.1.1.253/24
```

Both are inside the subnet:

```text
10.1.1.0/24
```

So they are local to each other.

But:

```text
10.1.1.2/30
10.1.1.4/30
```

are not in the same subnet.

`10.1.1.2/30` belongs to:

```text
10.1.1.0/30
```

`10.1.1.4/30` is the next subnet, and `10.1.1.4` is the network address for that subnet.

```text
10.1.1.4/30
```

So they are remote from each other and need routing to communicate.

### Routes

Think of a routing table like a map. When a router needs to forward a packet, it checks the destination IP against the routes it knows and decides where to send the packet next.

There are three types of routes I will cover here:

**Connected routes**, the reason they are called connected routes is because they are directly connected through an interface in that subnet and thus they can reach it without another router. They are automatically populated for every network that is directly connected to the router. If you looked at the routing table, the router type would show as `local` and the next hop `direct`.

A **static route** is a manually defined route. The router does not own that subnet, but it knows which next hop to send traffic to so it can reach that subnet. 

Here is an example of the commands used in SRLinux to create a static route:
```
--{ running }--[  ]--
A:srl1# enter candidate

--{ candidate shared default }--[  ]--
A:srl1# network-instance default

--{ * candidate shared default }--[ network-instance default ]--
A:srl1# set next-hop-groups group ngh-10-1-4-0-24 admin-state enable

--{ * candidate shared default }--[ network-instance default ]--
A:srl1# set next-hop-groups group  ngh-10-1-4-0-24 nexthop 1 ip-address 10.1.2.2

--{ * candidate shared default }--[ network-instance default ]--
A:srl1# exit

--{ * candidate shared default }--[  ]--
A:srl1# network-instance default

--{ * candidate shared default }--[ network-instance default ]--
A:srl1# set static-routes route 10.1.4.0/24 admin-state enable

--{ * candidate shared default }--[ network-instance default ]--
A:srl1# set static-routes route 10.1.4.0/24 next-hop-group ngh-10-1-4-0-24

--{ * candidate shared default }--[ network-instance default ]--
A:srl1# commit now
All changes have been committed. Leaving candidate mode.

--{ + running }--[ network-instance default ]--
A:srl1# info static-routes
    route 10.1.4.0/24 {
        admin-state enable
        next-hop-group ngh-10-1-4-0-24
    }
```

A **default route** is the catch-all route. It is written as `0.0.0.0/0` the `/0` means 0 bits are masked/matched, which means it can match every IPv4 address. This is used when there is no specific route in the routing table.

When multiple routes match, the longest prefix match wins. This means the most specific route is used.

For example:

```text
10.1.1.0/30 -> eth0
10.1.1.0/24 -> eth1
```

If the router needs to reach `10.1.1.1`, both routes match, but `10.1.1.0/30` is more specific than `10.1.1.0/24`, so the router uses `eth0`.

Static routes are useful for small networks, stub networks, and default routes. Once the network gets bigger, manually managing routes becomes harder, which is when dynamic routing protocols are usually used.

## Layer 4: The Transport Layer

Layer 4 is the Transport Layer. It answers which application traffic should go to on a host.

At layer 4, the question is:

- Which application do things go to?
- Which application am I looking at?

TCP and UDP are layer 4 protocols.

TCP must be acknowledged. UDP has no acknowledgement and is used for real time streaming.

Ports identify the application.

For example:

```text
80   HTTP
67   DHCP
443  HTTPS
```

This means that layer 3 can get traffic to the host, but layer 4 gets that traffic to the right application on that host.

Without layer 4, the host may receive the traffic, but it would not know which application should handle it.

## Putting it together

1. Layer 1 gives us a way to send and receive bits.
2. Layer 2 moves frames across the same local link using MAC addresses.
3. Layer 3 uses IP addresses, subnets, and routes to move traffic between networks.
4. Layer 4 uses TCP, UDP, and ports to send traffic to the right application.

If something is broken, you can troubleshoot through the layers and ask yourself:
- Is there a link?
- Can local devices communicate?
- Is the IP address and subnet correct?
- Is there a route?
- Is the traffic reaching the right port?
