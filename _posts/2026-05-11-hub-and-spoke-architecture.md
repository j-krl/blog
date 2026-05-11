---
title: "Wireguard Pt. 2: Hub and Spoke with iptables"
layout: post
tags:
    - networking
    - homelab
    - wireguard
date: 2026-05-11
---

In my last post I demonstrated how to set up a simple 1:1 Wireguard VPN. In my homelab I have expanded that architecture. I have a Kubernetes cluster with three nodes, and I wanted to be able to ssh into the nodes outside my home network, so I set up a Wireguard interface on each node. I then port forwarded a router port for each node so I could access each node individually. The architecture looked like this:

<pre class="mermaid" style="text-align: center;">
---
title: Partial mesh
---
flowchart LR
    ED@{shape: tag-rect, label: "External Host"}
    ED --> Gateway
    Gateway -->|Port forward| n1@{ shape: tag-rect, label: "LAN Host 1"}
    Gateway -->|Port forward| n2@{ shape: tag-rect, label: "LAN Host 2"}
    Gateway -->|Port forward| n3@{ shape: tag-rect, label: "LAN Host 3"}
    subgraph LAN
        Gateway
        n1
        n2
        n3
    end
</pre>

> **NOTE:** For all diagrams in this post, a rectangle with a diagonal notch represents a device with a Wireguard interface.

I have four Wireguard interfaces I'm maintaining in this design -- one for each device. Sometime after I set up the network I inherited a new laptop. To add the new laptop to my existing VPN I needed to update a total of four Wireguard config files, which was mildly annoying. When I later went to add an old Raspberry Pi as a fourth node to my cluster, I decided the architecture wasn't scalable and started looking for a new design for my network.

## Hub and spoke networking

Enter hub-and-spoke networking. In this architecture all traffic flows through a central host where access controls, routing rules, etc. can be configured. The previous design didn't follow the client-server pattern as every LAN host was also part of the VPN, each with its own Wireguard interface. With the new architecture, ingress traffic from a new source can be permitted for the entire LAN by updating a single interface. Here's how the network looks after the redesign:

<pre class="mermaid" style="text-align: center;">
---
title: Hub-and-spoke
---
flowchart LR
    ED@{ shape: tag-rect, label: "External
    Host 1"} --> Gateway
    ED2@{ shape: tag-rect, label: "External
    Host 2"} --> Gateway
    Gateway -->|Port forward| n1@{ shape: tag-rect, label: "LAN Host 1
    (Wireguard Server)"}
    n1 -. Forward .-> n2[LAN Host 2]
    n1 -. Forward .-> n3[LAN Host 3]
    subgraph LAN
        Gateway
        n1
        n2
        n3
    end
</pre>

In this new architecture, Host 1 acts as our server which terminates all our Wireguard traffic and forwards it within the LAN. We can talk to hosts on the LAN that aren't connected to the VPN. This means extra hosts added to the LAN can be routable without additional configuration. Not only is it easier to make LAN hosts reachable from the VPN this way; it's also more secure as VPN traffic flowing through a single point is easier to administer.

What makes this possible is the `AllowedIPs` attribute in the Wireguard config. I mentioned in my last post that for egress traffic, the AllowedIPs indicates the CIDR ranges that get routed to the peer from the enclosing block. The important part is that CIDR range *does not* have to be part of the VPN's subnet range. So we can set the AllowedIP range to be the *LAN's* range, and when the packet arrives at the Wireguard server it will forward that packet through its LAN interface. We only need to allow forwarding packets on the Wireguard server.

## Kernel forwarding

To start forwarding packets between interfaces, the kernel needs to be told to allow it. By default, most systems disable this to prevent inadvertently acting as a router. To enable it persistently, add the following line to `/etc/sysctl.conf`:
```
net.ipv4.ip_forward = 1
```

Without this setting, the kernel will drop any packet destined for a different interface. `/etc/sysctl.conf` is the configuration file for the `sysctl` program, which reads and writes kernel parameters at runtime. Run `sudo sysctl -p` to reload the configuration without having to reboot.

## iptables

Now we need to enable packet forwarding in the operating system's `iptables`. I won't go into great depth on iptables as I am no expert on them, but the general iptable hierarchy looks like this:

1. Tables
2. Chains
3. Rules

...where Tables are made up of Chains, and Chains are made up of Rules. The list of available Tables and Chains are preset by the system. Tables are separated by area of concern and support different types of Rules. The available tables are:

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

It's good to understand `iptables` broadly, but this is probably too much information. We only need two Rules to make our Wireguard server forward packets properly. Before we do this we should define the IPs of our key interfaces:

- Wireguard interfaces (192.168.100.0/24):
    - Wireguard Server: 192.168.100.1
    - External Host 1: 192.168.100.10
- LAN interfaces (192.168.1.0/24):
    - Wireguard Server: 192.168.1.1
    - LAN Host 2: 192.168.1.2

With those established, here are the iptable Rules we need on our Wireguard server:

- In the `filter` table we need an `ACCEPT` rule on the `FORWARD` chain for incoming packets from the Wireguard network through the Wireguard interface


```
iptables -t filter -A FORWARD -i wg0 -s 192.168.100.0/24 -o eth0 -d 192.168.1.0/24 -j ACCEPT
```

- In the `nat` table we need a `MASQUERADE` rule on the `POSTROUTING` chain to translate the source IP on the forwarded packets from the Laptop's Wireguard IP to the Wireguard server's LAN IP

```
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -d 192.168.1.0/24 -o eth0 -j MASQUERADE
```

The forward rule is fairly self-explanatory so we won't spend any time on it. We need the masquerade rule because if we forward packets from the Wireguard server to a LAN host without it, the hosts won't know where to respond as they aren't on the VPN. For the routing path to work, any responses from the non-VPN LAN hosts will have to be routed back through the Wireguard server.

The masquerade rule does exactly what we need. It translates the source IP from the external host to the Wireguard server's LAN IP on the outgoing requests. The kernel tracks the translation and will reverse it once it receives a response through the same connection. Below are two diagrams to demonstrate the request-response path with and without the rule, and why requests will fail without it:

<pre class="mermaid" style="text-align: center;">
---
title: Route path without MASQUERADE
---
sequenceDiagram
    box External Host
    participant EC as WG Interface (100.10)
    end
    box Wireguard Server
    participant WS as WG Interface (100.1)
    participant WL as LAN Interface (1.1)
    end
    box LAN Host
    participant N1 as LAN Interface (1.2)
    end
    EC->>WS: Src 100.10
    WS->>WL: Src 100.10 (forward)
    WL->>N1: Src 100.10 (forward)
    N1--xN1: Dest 100.10 (dropped)
</pre>

<pre class="mermaid" style="text-align: center;">
---
title: Route path with MASQUERADE
---
sequenceDiagram
    box External Host
    participant EC as WG Interface (100.10)
    end
    box Wireguard Server
    participant WS as WG Interface (100.1)
    participant WL as LAN Interface (1.1)
    end
    box LAN Host
    participant N1 as LAN Interface (1.2)
    end
    EC->>WS: Src 100.10
    WS->>WL: Src 100.10 (forward)
    WL->>N1: Src 1.1 (masq)
    activate WL
    N1-->>WL: Dest 1.1
    deactivate WL
    WL-->>WS: Dest 100.10 (unmasq)
    WS-->>EC: Dest 100.10
</pre>

And with that, we have a functioning hub-and-spoke Wireguard network.

## Wireguard on the router

Eventually I caved and bought a router that supported Wiregaurd natively. No more sshing into my hosts to generate and transfer keys around through the CLI. Instead, the router has a nice UI for generating the key and network IPs. Really, this way makes the most sense as all traffic flows through the gateway anyway, so may as well terminate the Wireguard tunnel at the logical hub. With the new router my architecture looks like this:

<pre class="mermaid" style="text-align: center;">
---
title: Wireguard server on the gateway
---
flowchart LR
    GW@{ shape: tag-rect, label: "Gateway
    (Wireguard Server)"}
    ED@{ shape: tag-rect, label: "External
    Host 1"} --> GW
    ED2@{ shape: tag-rect, label: "External
    Host 2"} --> GW
    GW -. Forward .-> n1[LAN Host 1]
    GW -. Forward .-> n2[LAN Host 2]
    GW -. Forward .-> n3[LAN Host 3]
    subgraph LAN
        GW
        n1
        n2
        n3
    end
</pre>

Our most bike wheel looking architecture yet. Mission accomplished.

<br />
<div align="center"><img src="/assets/img/bike-wheel.jpg" alt="Wireguard" width=350 /></div>

