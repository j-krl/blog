---
title: "Wireguard Pt. 2: Hub and Spoke Architecture"
layout: post
tags:
    - networking
    - homelab
    - wireguard
date: 2026-05-08
---

In my last post I demonstrated how to set up a simple 1:1 Wireguard VPN. On my home server I have a Kubernetes cluster with three nodes that I wanted to be able to ssh into individually. I set up a Wireguard interface on each node as well as my laptop. The architecture looked like this:

<pre class="mermaid" style="text-align: center;">
---
title: Partial mesh
---
flowchart LR
    Laptop --> Gateway
    Gateway --> n1[Node 1]
    Gateway --> n2[Node 2]
    Gateway --> n3[Node 3]
    subgraph LAN
        Gateway
        n1
        n2
        n3
        subgraph Cluster
            n1
            n2
            n3
        end
    end
</pre>

So I have four Wireguard interfaces I'm maintaining in this design -- one for each device. Since then I inherited a new laptop, and to set it up to connect to all the VPN devices I wanted I needed to modify three different config files. And it only gets worse as more gets added to the network. Something felt bad about it.

Enter hub-and-spoke networking. I learned about this design and implemented it for my VPN. It's much easier to maintain now. Here's how it looks when I add my second laptop to the network:

<pre class="mermaid" style="text-align: center;">
---
title: Hub-and-spoke
---
flowchart LR
    Laptop --> Gateway
    L2[Laptop 2] --> Gateway
    Gateway --> n1[Node 1]
    n1 -. Forward .-> n2[Node 2]
    n1 -. Forward .-> n3[Node 3]
    subgraph LAN
        Gateway
        n1
        n2
        n3
        subgraph Cluster
            n1
            n2
            n3
        end
    end
</pre>

In this new architecture, Node 1 acts as our *server* in that it terminates all our Wireguard traffic and forwards it along within the network. It is our "hub" that all traffic routes through. Not only is it easier to add devices to the network this way, It's also more secure as traffic is only flowing through a single point where all allowed IPs can be configured for the whole network. If I wanted to add a device to my cluster network, no extra configuration is needed. As long as the server has been configured to forward along packets within the network, any new nodes will be immediately accessible.
