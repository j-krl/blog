---
title: "Wireguard Pt. 2: Hub and Spoke Architecture"
layout: post
tags:
    - networking
    - homelab
    - wireguard
date: 2026-05-08
---

In my last post I demonstrated how to set up a simple 1:1 Wireguard VPN. On my home server I have a Kubernetes cluster with four nodes that I wanted to be able to ssh into individually. I set up a Wireguard interface on each node as well as my laptop. The architecture looked like this:

<pre class="mermaid" style="text-align: center;">
flowchart TD
    Laptop
    Laptop --> n1[Node 1]
    Laptop --> n2[Node 2]
    Laptop --> n3[Node 3]
    Laptop --> n4[Node 4]
    subgraph LAN
        n1
        n2
        n3
        n4
    end
</pre>
