---
title: "Building a WireGuard VPN Gateway on an Oracle Cloud VM with WSL2"
date: 2026-10-04
draft: false
subject: "Networking"
topic: "WireGuard"
weight: 1
tags:
  - wireguard
  - networking
  - oracle-cloud
  - wsl2
  - vpn
  - linux
summary: "Using an Oracle Cloud static IP as a WireGuard VPN egress gateway for WSL2 and mobile devices to access IP-whitelisted AWS and office VMs."
---

> **Using an Oracle Cloud static IP as a VPN egress for AWS and office VM access**

Sometimes the problem isn't that you need a VPN to access the Internet.

You need a VPN because **your Internet-facing IP keeps changing**.

In my case, I needed to access AWS and office VMs from a machine sitting behind a CGNAT connection. The public IP assigned by the ISP wasn't reliable enough to whitelist in security groups or firewalls.

The solution was to use an Oracle Cloud VM as a WireGuard VPN server.

The final architecture looks like this:

```text
                         Internet
                            │
                            │
                  Oracle Cloud VM
                  Public IP:
                  129.159.231.129
                            │
                     WireGuard wg0
                       10.50.0.1/24
                            │
              ┌─────────────┴─────────────┐
              │                           │
           WSL2 Laptop                 Phone
           10.50.0.2                   10.50.0.3
              │                           │
              └───────────┬───────────────┘
                          │
                    Internet / VMs
                          │
                Source IP becomes
                129.159.231.129
```

The important result is:

```text
AWS / Office VM
      ↓
sees 129.159.231.129
```

Instead of seeing my ISP's changing CGNAT address.

---

## Why WireGuard?

There are many VPN solutions available, but WireGuard is particularly convenient for this kind of setup.

It is:

* lightweight
* fast
* simple to configure
* available on Linux, Windows, Android and iOS
* based on public/private key pairs
* easy to configure for multiple peers

The architecture is also simple enough that troubleshooting doesn't require a huge VPN stack.

---

## The Environment

The VPN server was an Ubuntu VM running on Oracle Cloud.

The client was Ubuntu running under WSL2.

The important network addresses were:

```text
Oracle VM:
    Private IP: 10.0.0.43
    Public IP: 129.159.231.129

WireGuard network:
    10.50.0.0/24

WireGuard server:
    10.50.0.1

WSL client:
    10.50.0.2
```

The Oracle public IP was the important part.

The goal was to whitelist:

```text
129.159.231.129/32
```

on AWS security groups or office firewalls.

---

## Step 1 — Verify WireGuard Support in WSL2

One concern I had before starting was whether WireGuard would actually work inside WSL2.

There is a common misconception that WSL cannot use WireGuard because it doesn't have the required kernel support.

On a modern WSL2 installation, this isn't necessarily true.

I tested it directly.

First:

```bash
sudo modprobe wireguard
```

Then:

```bash
lsmod | grep wireguard
```

The module loaded successfully.

I also checked the WireGuard CLI:

```bash
which wg
wg --version
```

and got:

```text
/usr/bin/wg
wireguard-tools v1.0.20210914
```

Finally, I tested whether WSL could create a WireGuard interface:

```bash
sudo ip link add wg-test type wireguard
sudo ip link delete wg-test
```

That worked.

So before assuming WSL won't work, test the actual environment.

---

## Step 2 — Generate the Client Keypair

WireGuard uses public/private key pairs.

For the WSL client:

```bash
umask 077

wg genkey | tee client-private.key | wg pubkey > client-public.key
```

The important rule is:

> Never share private keys. Only the public key goes into the other side's configuration.

The WSL client received:

```text
10.50.0.2/24
```

---

## Step 3 — Configure the Oracle WireGuard Server

The WireGuard server configuration was placed at:

```bash
/etc/wireguard/wg0.conf
```

The basic interface looked like:

```ini
[Interface]
Address = 10.50.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY
```

The WSL client was added as a peer:

```ini
[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.50.0.2/32
```

Notice that the server uses `/32` for the peer.

That's important.

The server isn't saying:

```text
10.50.0.0/24 belongs to this client
```

It is saying:

```text
10.50.0.2 belongs to this specific peer
```

---

## Step 4 — Oracle Cloud Networking

Having WireGuard listening on the VM isn't enough.

Oracle Cloud's network security rules also need to allow WireGuard traffic.

The required inbound traffic was:

```text
UDP 51820
```

Once that was allowed, packets were reaching the server.

However, the first handshake still didn't work.

This led to one of the more interesting troubleshooting steps.

---

### Problem #1 — UDP 51820 was reaching the VM but WireGuard wasn't responding

I used `tcpdump` on the Oracle VM:

```bash
sudo tcpdump -ni ens3 udp port 51820
```

Packets were clearly arriving:

```text
1.187.214.31.39740 > 10.0.0.43.51820: UDP
```

But there was no response.

The reason was an existing iptables rule:

```text
REJECT all
reject-with icmp-host-prohibited
```

The firewall looked roughly like this:

```text
INPUT
1 ACCEPT established
2 ACCEPT ICMP
3 ACCEPT loopback
4 ACCEPT SSH
5 REJECT everything
```

WireGuard UDP/51820 was therefore reaching the operating system but getting rejected before WireGuard could process it.

The fix was:

```bash
sudo iptables -I INPUT 1 -p udp --dport 51820 -j ACCEPT
```

After that, tcpdump showed traffic in both directions and the WireGuard handshake succeeded.

#### Lesson

If a VPN handshake doesn't work:

Don't immediately assume WireGuard is broken.

Check the entire path:

```text
Client
 ↓
Internet
 ↓
Cloud security rules
 ↓
VM network interface
 ↓
iptables/firewalld
 ↓
WireGuard
```

`tcpdump` is extremely useful for determining which layer is actually failing.

---

## Step 5 — Enable IP Forwarding

The Oracle VM isn't just a VPN endpoint.

It is acting as a router.

Therefore Linux must allow packet forwarding.

Check:

```bash
sudo sysctl net.ipv4.ip_forward
```

The required value is:

```text
net.ipv4.ip_forward = 1
```

If necessary:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

For persistence, create:

```text
/etc/sysctl.d/99-wireguard.conf
```

with:

```text
net.ipv4.ip_forward=1
```

---

## Step 6 — Configure Forwarding and NAT

The traffic path needs to be:

```text
wg0 → ens3 → Internet
```

and return traffic needs to come back:

```text
Internet → ens3 → wg0
```

I used:

```bash
iptables -I FORWARD 1 -i wg0 -o ens3 -j ACCEPT
iptables -I FORWARD 2 -i ens3 -o wg0 \
    -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

Then NAT:

```bash
iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
```

The important concept here is that the Oracle VM translates the WireGuard client's private VPN address into its public IP.

So:

```text
10.50.0.2
    ↓
Oracle NAT
    ↓
129.159.231.129
```

---

### Problem #2 — The firewall rules were in the wrong order

This was a classic iptables mistake.

Initially, the Oracle FORWARD chain looked like:

```text
1 REJECT all
2 ACCEPT wg0 → ens3
3 ACCEPT ens3 → wg0 RELATED,ESTABLISHED
```

It looks like forwarding is allowed.

But iptables evaluates rules from top to bottom.

Therefore traffic hit:

```text
REJECT all
```

before it ever reached the WireGuard forwarding rule.

The result was:

```text
WireGuard handshake       ✅
Ping VPN server           ✅
Internet through VPN      ❌
```

The fix was simply to put the ACCEPT rules before the REJECT:

```bash
sudo iptables -I FORWARD 1 -i wg0 -o ens3 -j ACCEPT

sudo iptables -I FORWARD 2 \
    -i ens3 -o wg0 \
    -m conntrack --ctstate RELATED,ESTABLISHED \
    -j ACCEPT
```

The final order became:

```text
1 ACCEPT wg0 → ens3
2 ACCEPT ens3 → wg0 RELATED,ESTABLISHED
3 REJECT all
```

This was a good reminder:

> With iptables, a correct rule in the wrong position is effectively an incorrect rule.

---

### Problem #3 — Duplicate iptables rules

During troubleshooting, some forwarding and NAT rules were accidentally duplicated.

For example:

```text
ACCEPT wg0 → ens3
ACCEPT ens3 → wg0
ACCEPT wg0 → ens3
ACCEPT ens3 → wg0
```

and:

```text
MASQUERADE → ens3
MASQUERADE → ens3
```

The duplicates weren't necessary.

They were removed using:

```bash
sudo iptables -D FORWARD -i wg0 -o ens3 -j ACCEPT
sudo iptables -D FORWARD -i ens3 -o wg0 \
    -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

sudo iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
```

#### Lesson

When troubleshooting firewall rules:

```bash
iptables -L -n -v --line-numbers
```

is much more useful than simply looking at the configuration file.

Packet counters also tell you whether a rule is actually being hit.

---

## Step 7 — Test the VPN Before Full Tunneling

Initially, the client was configured with:

```ini
AllowedIPs = 10.50.0.0/24
```

This meant only traffic destined for the WireGuard network used the VPN.

For example:

```bash
ping 10.50.0.1
```

worked.

But Internet traffic still used the normal WSL connection.

Checking the route showed:

```text
54.169.173.187 via 172.19.48.1 dev eth0
```

And the public IP was still the ISP's CGNAT address:

```text
1.187.214.31
```

That was expected.

---

## Step 8 — Switch to a Full Tunnel

The actual goal was to use Oracle as the Internet egress.

So the client configuration was changed to:

```ini
AllowedIPs = 0.0.0.0/0
```

This tells WireGuard that IPv4 traffic should use the VPN.

After bringing the tunnel up:

```bash
sudo wg-quick up wg0
```

the route became:

```text
1.1.1.1 dev wg0 table 51820
```

while the WireGuard endpoint itself was still routed outside the tunnel.

---

### Problem #4 — Understanding WireGuard's policy routing

At first glance, a full-tunnel configuration can look strange.

Running:

```bash
ip route show table 51820
```

showed:

```text
default dev wg0 scope link
```

And:

```bash
ip rule
```

showed:

```text
0:      from all lookup local
32764:  from all lookup main suppress_prefixlength 0
32765:  not from all fwmark 0xca6c lookup 51820
32766:  from all lookup main
32767:  from all lookup default
```

WireGuard was using a firewall mark:

```text
0xca6c
```

and a dedicated routing table:

```text
51820
```

This is part of how `wg-quick` implements a full tunnel without creating a routing loop.

The important practical test was:

```bash
ip route get 129.159.231.129
```

which returned:

```text
129.159.231.129 via 172.19.48.1 dev eth0
```

while:

```bash
ip route get 1.1.1.1
```

returned:

```text
1.1.1.1 dev wg0 table 51820
```

This is exactly what we wanted.

The VPN endpoint must remain reachable through the normal network, while general Internet traffic goes through WireGuard.

---

## Step 9 — Verify the Public Egress IP

This was the most important test:

```bash
curl -4 https://checkip.amazonaws.com
```

With WireGuard enabled:

```text
129.159.231.129
```

Success.

At this point, the complete path was working:

```text
WSL
 ↓
10.50.0.2
 ↓
WireGuard
 ↓
10.50.0.1
 ↓
Oracle NAT
 ↓
Internet
 ↓
129.159.231.129
```

This is the IP that can be whitelisted on AWS security groups and office firewalls.

---

## Step 10 — Don't Automatically Add a Kill Switch

A common WireGuard recommendation is to configure a kill switch so that Internet traffic is blocked whenever the VPN is disconnected.

That isn't always the right design.

For this particular use case, I wanted normal Internet connectivity to remain available.

The desired behavior was:

```text
VPN UP
   ↓
VM traffic → Oracle static IP
   ↓
129.159.231.129
```

But if WireGuard goes down:

```text
VPN DOWN
   ↓
Normal WSL networking
   ↓
CGNAT IP
```

The VM simply won't accept the connection unless its security group/firewall temporarily allows the CGNAT IP.

This is actually useful operationally.

If the VPN is unavailable, I can temporarily whitelist my current IP rather than losing all Internet connectivity.

---

## Step 11 — Verify the Fallback Behavior

This was tested explicitly.

First:

```bash
sudo wg-quick down wg0
```

Then:

```bash
curl -4 https://checkip.amazonaws.com
```

returned:

```text
1.187.214.31
```

That's the normal ISP/CGNAT address.

Then:

```bash
sudo wg-quick up wg0
```

and:

```bash
curl -4 https://checkip.amazonaws.com
```

returned:

```text
129.159.231.129
```

So the fallback behavior was confirmed.

No complicated fail-closed firewall was necessary.

---

## Step 12 — Run WireGuard Through systemd

Once the VPN was working manually, I wanted it to start automatically.

The configuration was already at:

```text
/etc/wireguard/wg0.conf
```

So on WSL:

```bash
sudo systemctl enable wg-quick@wg0
```

This created the systemd symlink.

One initial mistake was starting the service while `wg0` was already manually running.

The service failed with:

```text
wg-quick: `wg0' already exists
```

That's not a WireGuard configuration error.

The interface was simply already up.

The correct sequence was:

```bash
sudo wg-quick down wg0
sudo systemctl start wg-quick@wg0
```

Then:

```bash
sudo systemctl status wg-quick@wg0
```

and:

```bash
sudo wg show
```

should confirm the interface is managed by systemd.

Check whether it is enabled:

```bash
systemctl is-enabled wg-quick@wg0
```

Expected:

```text
enabled
```

---

## Adding a Phone as Another WireGuard Peer

WireGuard makes adding another device straightforward.

The phone doesn't need to generate its own keys.

A keypair can be generated on another trusted machine.

For example:

```bash
umask 077

wg genkey | tee phone-private.key | \
    wg pubkey > phone-public.key
```

The phone gets its own VPN address:

```text
10.50.0.3
```

while WSL remains:

```text
10.50.0.2
```

The Oracle server configuration then contains another peer:

```ini
[Peer]
PublicKey = PHONE_PUBLIC_KEY
AllowedIPs = 10.50.0.3/32
```

The phone configuration looks like:

```ini
[Interface]
PrivateKey = PHONE_PRIVATE_KEY
Address = 10.50.0.3/24
DNS = 1.1.1.1

[Peer]
PublicKey = ORACLE_SERVER_PUBLIC_KEY
Endpoint = 129.159.231.129:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

The same Oracle public IP can therefore be used as the egress point for both devices.

```text
WSL ────────┐
10.50.0.2  │
            ├── WireGuard ── Oracle ── Internet
Phone ──────┘                    │
10.50.0.3                        ↓
                          129.159.231.129
```

For phones, a QR code is usually the easiest way to import the configuration into the WireGuard app.

---

## A Note About Private Keys

During troubleshooting, it's very easy to accidentally expose a private key while running commands such as:

```bash
sudo wg showconf wg0
```

Unlike:

```bash
sudo wg show
```

`wg showconf` can display the private key.

Never publish that output in a blog post, Git repository, support ticket, or chat.

If a private key has actually been exposed, rotate it.

The safe rule is:

```text
Private key → stays on the device
Public key  → can be shared with the other WireGuard endpoint
```

---

## Oracle VM Restart Considerations

Because the VPN server is running on an Oracle Cloud VM, the configuration needs to survive reboots.

The WireGuard service can be enabled with:

```bash
sudo systemctl enable wg-quick@wg0
```

The `wg0.conf` configuration contains the `PostUp` rules for forwarding and NAT.

Conceptually:

```text
VM boots
   ↓
systemd
   ↓
wg-quick@wg0
   ↓
wg0 created
   ↓
PostUp
   ↓
iptables forwarding + MASQUERADE
```

The Oracle public IP should also be a reserved/static public IP if it is being used for firewall allowlisting.

That means the whitelist can remain:

```text
129.159.231.129/32
```

instead of having to change whenever the VM restarts.

---

## Troubleshooting Checklist

If the VPN doesn't work, troubleshoot it from the bottom up.

### 1. Is WireGuard running?

```bash
sudo wg show
```

Look for:

```text
latest handshake
```

If there's no handshake, don't troubleshoot Internet routing yet.

---

### 2. Can the client reach the VPN server?

```bash
ping 10.50.0.1
```

If this fails, fix the tunnel before looking at NAT.

---

### 3. Is UDP 51820 reaching the server?

On Oracle:

```bash
sudo tcpdump -ni ens3 udp port 51820
```

If packets arrive but there is no response, check:

* iptables
* firewalld
* WireGuard configuration
* Oracle Cloud security rules

---

### 4. Is forwarding enabled?

```bash
sudo sysctl net.ipv4.ip_forward
```

It should return:

```text
net.ipv4.ip_forward = 1
```

---

### 5. Are forwarding rules in the right order?

```bash
sudo iptables -L FORWARD -n -v --line-numbers
```

Make sure the WireGuard ACCEPT rules appear before a general REJECT rule.

---

### 6. Is NAT configured?

```bash
sudo iptables -t nat -L POSTROUTING -n -v --line-numbers
```

You should see a MASQUERADE rule for the external interface.

---

### 7. What route is actually being used?

For the VPN endpoint:

```bash
ip route get 129.159.231.129
```

It should use:

```text
eth0
```

For normal Internet traffic:

```bash
ip route get 1.1.1.1
```

With the full tunnel active, it should use:

```text
wg0
```

---

### 8. What public IP does the Internet see?

```bash
curl -4 https://checkip.amazonaws.com
```

With VPN:

```text
129.159.231.129
```

Without VPN:

```text
your normal ISP/CGNAT IP
```

---

## Final Architecture

After all the troubleshooting, the final setup is surprisingly simple:

```text
                         ┌──────────────────────┐
                         │     Oracle Cloud     │
                         │                      │
                         │  Public IP           │
                         │ 129.159.231.129      │
                         │                      │
                         │  WireGuard            │
                         │  10.50.0.1           │
                         └──────────┬───────────┘
                                    │
                         NAT / MASQUERADE
                                    │
                              Internet
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                 WSL2                            Phone
              10.50.0.2                        10.50.0.3
                    │                               │
                    └────────── WireGuard ──────────┘
```

The practical outcome is:

```text
VPN connected:
    VM sees 129.159.231.129

VPN disconnected:
    normal Internet continues
    VM sees the normal CGNAT IP
```

This gives you a small, inexpensive VPN gateway that can provide a **stable outbound IP for accessing resources that require IP allowlisting**, without forcing all of your normal connectivity to depend on the VPN.

The biggest lesson from the setup wasn't actually how to configure WireGuard.

It was learning to troubleshoot the complete network path:

```text
Cloud firewall
      ↓
Linux firewall
      ↓
WireGuard handshake
      ↓
Routing
      ↓
IP forwarding
      ↓
NAT
      ↓
Internet
      ↓
Target VM security group
```

When those layers are checked one by one, WireGuard becomes much less mysterious.
