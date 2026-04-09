---
title: "Intro to Wireguard VPN"
layout: post
category: homelab
tags:
    - tech
    - networking
    - homelab
date: 2026-04-08
---

In the last few months I set up an extremely simple "homelab" so I could have an environment to practice working with various technologies. The lab consists of three old mini Lenovo ThinkCenters connected to a switch. Once these were set up with Ubuntu Server the first thing I wanted was a VPN so I could securely connect to the lab when I was away from my house. The VPN I settled on was Wireguard, and I found the protocol so simple and elegant that I wanted to write a brief intro on the subject. Honestly, this post is a bit of a rip-off from the [Wireguard homepage][wireguard], which is very well written, so check that page out for more information.

## What is Wireguard?

<div align="center"><img src="/assets/img/wireguard.webp" alt="Wireguard" width=250 /></div>
<br />

[Wireguard][wireguard] is a VPN protocol comparable to [OpenVPN][openvpn], but with a emphasis on simplicity. Wireguard has ~4k lines of code, whereas OpenVPN is in the hundreds of thousands. Wireguard has taken off in popularity over the last decade, with managed VPN solutions such as [Tailscale][tailscale] built on top of the Wireguard protocol.

## How it works

Wireguard runs off a virtual network interface and a config file, sending packets over UDP. It is up to the VPN admin to determine the network's address space and the IP addresses to designate to each peer in the network. Wireguard is only concerned with sending and receiving encrypted packets over the virtual interface.

Wireguard is based off the concept of Cryptokey Routing. Each peer on the network owns a private and public key pair, where public keys are associated with private network addresses or address ranges. Each interface has an associated config file that contains the interface's private key, the port it's listening on, and the IP addresses and public keys of the peers the interface can send traffic to or receive traffic from.

A packet travelling over a Wireguard VPN follows the following process:

1. Wireguard matches the destination IP to one of the address ranges in its list of peers
2. Once the destination IP is matched to a peer, the outgoing packet is encrypted with both the peer's public key and the host's private key
3. The encrypted packet is sent over the public network to the peer's endpoint (public) IP
4. The destination interface receives the packet, but rejects it if the source private IP is not in the list peer allowed IP ranges
5. The endpoint IP of the source interface is updated in the destination interface's peer list if necessary
6. The packet is decrypted with the private key of the destination interface and public key of the source interface, then consumed by the host machine

I find Wireguard elegant for a couple reasons:

- The dual meaning of what an "allowed IP" is. For egress traffic, the allowed IPs acts as a routing table to find the public key to encrypt with, and the destination public endpoint. For ingress traffic, the allowed IPs act as an access control list of source IPs Wireguard can receive traffic from.
- As long as the destination's endpoint IP remains stable, traffic received will automatically update the endpoint IP of the source interface. Only the peer that is generally receiving packets needs to have a stable public IP.

## Setting up

Install Wireguard with

```
sudo apt install wireguard
```

You'll need to determine your VPN's network address. We'll keep it simple here and use `192.168.99.0/24`. We'll address our own machine as `192.168.99.100` and our secondary machine as `192.168.99.200`.

## The config file

We'll start with a skeleton Wireguard config file:

```
[Interface]
ListenPort = ...
PrivateKey = ...

[Peer]
PublicKey = ...
AllowedIPs = ...
Endpoint = ...
```

The format of the config file is very simple:

- One `Interface` block for the host.
- Many `Peer` blocks for other peers on the network. In our example we only have one peer.

In the interface block the mandatory entries are:

- `ListenPort`: Port the Wireguard interface listens for traffic on
- `PrivateKey`: The private key for decrypting ingress traffic
- `Address`: The interface's IP address and the subnet mask

And in the peer block:

- `PublicKey`: The key for encrypting egress traffic to the peer
- `Endpoint`: The public IP address of the peer
- `AllowedIPs`: Means different things for ingress and egress traffic:
    - Egress: The range of private IPs that get sent to the peer's endpoint IP
    - Ingress: Acts as an ACL where only traffic from one of the peer allowed IP ranges will be accepted

## Generating key pairs

In this example we'll create two Wireguard interfaces on our machine and test sending traffic between them. Here's key pair number one:

```
% wg genkey
EGK2cbXWZ9CitSZiMMASWfJFrsE/Qm/JJMBLFVfohHc=
% echo "EGK2cbXWZ9CitSZiMMASWfJFrsE/Qm/JJMBLFVfohHc=" | wg pubkey
GpxRmzaMJFt7SNythiHaNtP4fKjJ9s9pRRNksQayGgM=
```

And key pair number two:

```
% wg genkey
sOK3kcCZ66M5MD1ktyWsekIbdxWTQ32ew1pgQ18ifnE=
% echo "sOK3kcCZ66M5MD1ktyWsekIbdxWTQ32ew1pgQ18ifnE=" | wg pubkey
SikRjPi3wUPoKTT08bC8R4e0fInp/Pdf225IueHqN2w=
```

We're going to eventually set up two virtual network interfaces `wg1` and `wg2`. Here's how the config files for the two interfaces are going to look so far:

```
# wg1
[Interface]
ListenPort = 51820
PrivateKey = EGK2cbXWZ9CitSZiMMASWfJFrsE/Qm/JJMBLFVfohHc=
Address = 192.168.99.100/24

[Peer]
PublicKey = SikRjPi3wUPoKTT08bC8R4e0fInp/Pdf225IueHqN2w=
AllowedIPs = 192.168.99.200/32
Endpoint = 127.0.0.1:51821

# wg2
[Interface]
ListenPort = 51821
PrivateKey = sOK3kcCZ66M5MD1ktyWsekIbdxWTQ32ew1pgQ18ifnE=
Address = 192.168.99.200/24

[Peer]
PublicKey = GpxRmzaMJFt7SNythiHaNtP4fKjJ9s9pRRNksQayGgM=
AllowedIPs = 192.168.99.100/32
Endpoint = 127.0.0.1:51820
```

So we've set up each interface to have

- Its own private key
- A unique port to listen for traffic on
- Its IP address and network address space
- The peer interface's private IP and public key
- The peer's public endpoint IP to send encrypted traffic to

51820 is the typical port Wireguard listens for traffic on. Since we're going to be running two interfaces on the same machine, they'll need to listen on unique ports, so 51821 seemed as good of a port as any other.

## Creating the interfaces

Now that the config files are generated, we need interfaces to point them at. I'm only going to run through the exercise for `wg1` but we'll need `wg2` as well to test connectivity within the network.

I'm running this on Ubuntu Server 24, so I'm going to use the `ip` command for setting up these interfaces.

First we're going to create the virtual network interface:

```
% ip link add dev wg1 type wireguard
```

After this we need to assign the `wg1` config file to interface we just created. One way would be to save the above configs in two separate files and assign them each to an interface with `wg setconf wg1 <wg1-filename.conf>`. Another would be to just create the `wg1.conf` file in `/etc/wireguard/wg1.conf`.

Now that the interface is configured we can bring it up with

```
% ip link set up dev wg1
```

Now let's confirm the interface is configured correctly. Run `sudo wg show wg1` and you should see something like this:

```
interface: wg1
  public key: GpxRmzaMJFt7SNythiHaNtP4fKjJ9s9pRRNksQayGgM=
  private key: (hidden)
  listening port: 51820

peer: SikRjPi3wUPoKTT08bC8R4e0fInp/Pdf225IueHqN2w=
  endpoint: 127.0.0.1:51821
  allowed ips: 192.168.99.200/32
```

When you run `ip -4 addr show` you should also see a `wg1` interface similar to this:

```
wg1: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 192.168.99.100/24 scope global wg1
       valid_lft forever preferred_lft forever
```

The `UP` in `<POINTOPOINT,NOARP,UP,LOWER_UP>` confirms that the interface is running.

## Testing the connection

We're now good to go. Test the interface and test the connection from 1 to 2 with `% ping 192.168.99.200`. You should get something like this:

```
PING 192.168.99.200 (192.168.99.200): 56 data bytes
64 bytes from 192.168.99.200: icmp_seq=0 ttl=64 time=40.424 ms
64 bytes from 192.168.99.200: icmp_seq=1 ttl=64 time=394.618 ms
64 bytes from 192.168.99.200: icmp_seq=2 ttl=64 time=176.697 ms
64 bytes from 192.168.99.200: icmp_seq=3 ttl=64 time=404.156 ms
--- 192.168.99.200 ping statistics ---
4 packets transmitted, 4 packets received, 0.0% packet loss
```

Success! That's all there is to it. Wireguard an extremely simple VPN protocol that's perfect for a homelab.

---

[wireguard]: https://www.wireguard.com/
[openvpn]: https://openvpn.net/
[tailscale]: https://tailscale.com/
