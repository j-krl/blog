---
title: "Wireguard Pt. 2: Hub and Spoke with iptables"
layout: post
tags:
    - networking
    - homelab
    - wireguard
date: 2026-05-08
---

In my last post I demonstrated how to set up a simple 1:1 Wireguard VPN. In my homelab I have expanded that architecture. I have a Kubernetes cluster with three nodes in their own subnet. I wanted to be able to ssh into the nodes outside my home network, so I set up a Wireguard interface on each node. I then port forwarded a router port for each node so I could access each individually. The architecture looked like this:

<pre class="mermaid" style="text-align: center;">
---
title: Partial mesh
---
flowchart LR
    ED@{shape: tag-rect, label: "External Device"}
    ED --> Gateway
    Gateway -->|Port forward| n1@{ shape: tag-rect, label: "Node 1"}
    Gateway -->|Port forward| n2@{ shape: tag-rect, label: "Node 2"}
    Gateway -->|Port forward| n3@{ shape: tag-rect, label: "Node 3"}
    subgraph LAN
        Gateway
        n1
        n2
        n3
    end
</pre>

> **NOTE:** For all diagrams in this post, a rectangle with a diagonal notch represents a device with a Wireguard interface.

I have four Wireguard interfaces I'm maintaining in this design -- one for each device. Sometime after I set up the network I inherited a new laptop. To add the new laptop to my existing VPN I needed to update a total of *four* Wireduard config files. As the network grew the architecture felt less and less scalable, so I started looking into other designs for the network.

## Hub and spoke networking

Enter hub-and-spoke networking. I learned about this design and implemented it for my VPN. The idea is there's a a central Wireguard "server" that acts as a hub for the network. All traffic flows through it and is forwarded to the relevant endpoint. The previous design didn't follow the client-server pattern as every node could be connected to individually. Adding devices to the new network is much easier, and the configuration effort stays constant the more devices that are added. Here's how the network looks now:

<pre class="mermaid" style="text-align: center;">
---
title: Hub-and-spoke
---
flowchart LR
    ED["External
    Device"] --> Gateway
    ED2["External
    Device 2"] --> Gateway
    Gateway -->|Port forward| n1["Node 1"]
    n1 -. Forward .-> n2[Node 2]
    n1 -. Forward .-> n3[Node 3]
    subgraph LAN
        Gateway
        n1
        n2
        n3
    end
</pre>

In this new architecture, Node 1 acts as our *server* which terminates all our Wireguard traffic and forwards it within the LAN. We can talk to devices on the LAN without them being in the VPN. In this architecture, the server is our "hub" that all traffic routes through. Not only is it easier to add devices to the network this way, It's also more secure as traffic is only flowing through a single point at which allowed IPs can be configured for the network. Any extra devices added to the LAN are routable without any extra configuration. The server just needs to be able to forward packets on the local network.

## iptables

The forwarding behaviour will require a little additional configuration. We'll leverage Linux's `iptables` for this. I won't go into great depth on `iptables` as I am no expert on them, but the general iptable hierarchy looks like:

1. Tables
2. Chains
3. Rules

Where Tables are made up of Chains, and Chains are made up of Rules. The list of available Tables and Chains are preset by the system. Tables are separated by area of concern and support different types of Rules. The available tables are:

- `filter` (default): Table used for filtering packets
- `nat`: Set network address translation rules on packets
- `mangle`: Modify packet headers
- `raw`: Specialized alteration of packets
- `security`: Access control rules on SELinux only

Chains are just the step in the traffic flow that some rule is applied to. Here's a description of the available Chains and what Tables they can be applied to:

| Chain         | Step                                                             | Relevant iptables             |
| ------------- | ---------------------------------------------------------------- | ----------------------------- |
| `INPUT`       | Just before the packet is given to the local process             | `filter` `mangle` `raw`       |
| `OUTPUT`      | Just after being produced by the local process                   | `filter` `mangle` `nat` `raw` |
| `FORWARD`     | Any packets being routed through the host, just after prerouting | `filter` `mangle`             |
| `PREROUTING`  | Just as packets arrive on the network interface                  | `nat` `mangle` `raw`          |
| `POSTROUTING` | Just before packets leave through the network interface          | `nat` `mangle`                |

<br />
See [this article](https://www.booleanworld.com/depth-guide-iptables-linux-firewall/) for a great in-depth explanation of `iptables`.


## Applying iptables rules to our Wireguard server

It's good to understand `iptables` broadly, but this is probably too much information. We only need two Rules to make our Wireguard server forward packets properly:

1. In the `filter` table we need an `ACCEPT` rule on the `FORWARD` chain for incoming packets from the Wireguard network through the Wireguard interface
2. In the `nat` table we need a `MASQUERADE` rule on the `POSTROUTING` chain to translate the source IP on the forwarded packets from the Laptop's Wireguard IP to the Wireguard server's LAN IP

(1) is fairly self-explanatory. We need (2) because if we forward packets from the Wireguard server to Node 2 or Node 3 without it, the nodes will have no idea where to respond. Remember, with the Hub and Spoke architecture the only device with a Wireguard interface on our LAN is Node 1, our Wireguard server. This means for Nodes 2 & 3 to be able to respond back to the Wireguard interface on the laptop, The responses will have to be routed back through Node 1 which is on the Wireguard network. The `MASQUERADE` rule translates the source IP from the laptop's Wireguard IP to Node 1's LAN IP. It tracks that translation in the kernel's connection tracking table and will translate back to the original address once it receives a response through that connection.

<pre class="mermaid" style="text-align: center;">
---
title: Request with MASQUERADE
---
sequenceDiagram
    participant EC as External Device
    participant WS as Wireguard Server
    participant N1 as LAN Device
    EC->>WS: Src 192.168.100.10
    WS->>N1: Src 192.168.1.1 (masq)
    activate WS
    N1-->>WS: Dest 192.168.1.1
    deactivate WS
    WS-->>EC: Dest 192.168.100.10
</pre>

<pre class="mermaid" style="text-align: center;">
---
title: Request without MASQUERADE
---
sequenceDiagram
    participant EC as External Device
    participant WS as Wireguard Server
    participant N1 as LAN Device
    EC->>WS: Src 192.168.100.10
    WS->>N1: Src 192.168.100.10
    N1--xN1: Dest 192.168.100.10 (dropped)
</pre>

## Wireguard on the router

Eventually I caved and bought a router that supported Wiregaurd natively. No more sshing into my node to generate and transfer keys around through the CLI. The router has a nice UI for generating the key and network IPs. Really this way makes the most sense as all traffic flows through the gateway anyway, so may as well terminate the Wireguard tunnel at the logical hub. Now my architecture looks like this:

<pre class="mermaid" style="text-align: center;">
---
title: Wireguard server on the gateway
---
flowchart LR
    GW["Gateway"]
    ED["External
    Device"] --> GW
    ED2["External
    Device 2"] --> GW
    GW -. Forward .-> n1[Node 1]
    GW -. Forward .-> n2[Node 2]
    GW -. Forward .-> n3[Node 3]
    subgraph LAN
        GW
        n1
        n2
        n3
    end
</pre>

