---
title: "How Docker Networking Works"
description: "How the Docker automates Linux kernel features"
publishDate: "2026-06-25"
tags: ["docker", "networking", "linux"]
---
Docker networking automates the setup of Linux networking features. When a container starts Docker sets up all the required networking for a container to reach the internet: a network namespace, a bridge, virtual cables, interfaces, routes, IP forwarding, and NAT or masquerade.

Docker networking begins by giving the container an isolated network stack through a network namespace.

## Network namespaces

A network namespace gives a container an isolated network stack. Think of this like a private networking room; it has nothing in it.

To create a network namespace:

```bash
sudo ip netns add red
sudo ip netns add blue
```

To inspect a network namespace:

```bash
sudo ip netns exec red ip link show
```

This command will give the ID of the interface and its state. Namespaces give containers their own network stack and configuration, allowing a separation of concerns and reduced blast radius.

## Docker bridge networking

The namespace acts as an isolated room but there is nothing in it, so Docker adds a Linux bridge which works in a similar way to a switch.

When spinning up a container with Docker, Docker creates a Linux bridge called `docker0` with the IP of `172.17.0.1`. `docker0` is what creates a gateway for the namespace and allows communication between containers.

If you were to create a bridge manually in Linux you would run the following commands:

```bash
sudo ip link add br-study type bridge
sudo ip link set br-study up
sudo ip addr add 10.0.0.254/24 dev br-study
```

If you were on the Docker host, you could run `ip link show docker0` to see the bridge and its MAC address. `docker network inspect bridge` shows the subnet, gateway, and container IP addresses for Docker's default bridge network in JSON.

However the namespace and the bridge are two separate entities and still need to be connected. This is where we use veth pairs.

## Veth pairs

Docker creates virtual cables called `veth-pairs` which connect the bridge and the containers. Veth pairs need to be connected on both sides of the wire.

To connect the two, one end of the veth pair is moved into the namespace and the other end is attached to the bridge:

```bash
sudo ip link add veth-r type veth peer name veth-r-br
sudo ip link set veth-r netns red
sudo ip link set veth-r-br master br-study
sudo ip link set veth-r-br up
```

Docker does this automatically for us, and allows traffic to move between container to container through the `docker0` bridge.

There are two sides to a `veth-pair`, one on the `docker0` side and one on the container side. Inside the container, the Docker network interface is usually called `eth0`. On the bridge side, the interface is called something like `vethxxxx`.

This means the same connection can be inspected from two angles: inside the container as `eth0`, and on the host bridge as a veth interface.

To find what interface connects to which container on the bridge side look at the interface IDs on `docker0` with `ip link show master docker0`. We will then be able to match with the container interface names using `ip addr` to find out which interface points to which container.

## Routes

Once the container is connected to the bridge, routes decide whether container traffic stays local or leaves the network. If there is no default route or a route out, the bridge will have no idea where to send the traffic to.

Routes decide whether container traffic stays local or needs a way out of the network. Without a route out of the namespace, there is nowhere to go.

```bash
# assign an IP address to red's veth interface
sudo ip netns exec red ip addr add 10.0.0.1/24 dev veth-r

# veth interface up
sudo ip netns exec red ip link set veth-r up

# loopback interface up
sudo ip netns exec red ip link set lo up
```

By assigning an IP a connected route is created for the local subnet. To reach outside that subnet to the internet, a namespace needs a default route via the bridge. To create a default route in Linux:

```bash
sudo ip netns exec red ip route add default via 10.0.0.254
```

`ip route` shows the default gateway and the routes available to the container or namespace.

## NAT and masquerade

For containers to reach the internet, IP forwarding and NAT or masquerade are needed. NAT lets traffic on the internet know where to return information to.

For containers to reach the internet, IP forwarding must be enabled between the Docker bridge and the internet.

NAT or masquerade must also be enabled so that when traffic leaves the host, it gets rewritten to the host IP address. That way, traffic on the internet knows where to return information to.

To configure this on Linux with forward and masquerade run:

```bash
sudo iptables -A FORWARD -i br-study -j ACCEPT
sudo iptables -A FORWARD -o br-study -j ACCEPT
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o enp12s0 -j MASQUERADE
```

NAT comes after routing because external traffic needs a return path through the host IP.
