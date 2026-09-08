# Lab 1 — Communication Within the Same Network

## Objective

Build a basic local network with two hosts and one switch, and verify how devices
communicate at Layer 2. The goal was to observe ARP resolution, switch MAC learning,
and confirm that two hosts in the same network can communicate without a router.

## Topology

Two PCs connected to a single Cisco 2960-24TT switch using Copper Straight-Through
cables. No router is present, and no default gateway is configured on either host.

```
   PC0                                    PC1
192.168.1.10                        192.168.1.20
     |                                     |
     | Fa0                             Fa0 |
     +----------- Switch0 (2960) ----------+
              Fa0/1        Fa0/2
```

## Addressing Table

| Device | Interface | IP Address   | Subnet Mask   | Default Gateway |
|--------|-----------|--------------|---------------|-----------------|
| PC0    | Fa0       | 192.168.1.10 | 255.255.255.0 | Not configured  |
| PC1    | Fa0       | 192.168.1.20 | 255.255.255.0 | Not configured  |

The default gateway was intentionally left empty to prove that hosts in the same
network do not need a router to communicate.

## Hardware Notes

| Device   | Model          | Details                                        |
|----------|----------------|------------------------------------------------|
| Switch0  | WS-C2960-24TT-L| 24 FastEthernet ports, 2 Gigabit Ethernet ports |
| PC0      | PC-PT          | MAC address: 00E0.F943.25E9                    |
| PC1      | PC-PT          | MAC address: 0001.C754.CBCC                    |

## Steps

1. Placed two PCs and one 2960 switch in the logical workspace.
2. Connected PC0 to Fa0/1 and PC1 to Fa0/2 using Copper Straight-Through cables.
3. Waited for the link lights to turn from orange to green (STP convergence).
4. Configured static IP addresses on both PCs, leaving the default gateway empty.
5. Checked the ARP table on PC0 before any communication.
6. Ran `ping` from PC0 to PC1.
7. Checked the ARP table again to confirm ARP resolution.
8. Inspected the switch MAC address table via the CLI.

## Observations

### Link status messages

When the switch finished booting, it reported both interfaces coming up:

```
%LINK-5-CHANGED: Interface FastEthernet0/1, changed state to up
%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/1, changed state to up
```

The first message confirms Layer 1 (physical signal). The second confirms Layer 2
(frame communication). Both must be up for the port to work.

### ARP table before communication

```
C:\>arp -a
No ARP Entries Found
```

The ARP table was empty even though both PCs were configured and cabled correctly.
This confirms that ARP only runs when a host actually needs to send data.
Configuring an IP address does not generate traffic by itself.

### Ping test

```
C:\>ping 192.168.1.20

Reply from 192.168.1.20: bytes=32 time<1ms TTL=128
Reply from 192.168.1.20: bytes=32 time<1ms TTL=128
Reply from 192.168.1.20: bytes=32 time<1ms TTL=128
Reply from 192.168.1.20: bytes=32 time<1ms TTL=128

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

All four packets succeeded. I expected the first packet to time out while ARP
resolved the destination MAC address, but Packet Tracer completed the ARP exchange
fast enough that no timeout occurred.

`TTL=128` is the default starting value on Windows hosts. Because it was not
decremented, this confirms the packet did not cross a router.

### ARP table after communication

```
C:\>arp -a
  Internet Address      Physical Address      Type
  192.168.1.20          0001.c754.cbcc        dynamic
```

The entry appeared automatically. PC0 sent an ARP Request as a broadcast, PC1
replied with its MAC address, and PC0 cached the result. The `dynamic` type means
the entry was learned, not manually configured, and it will expire after a timeout.

### Switch MAC address table

The first time I checked the table it was empty, because the switch had just
rebooted and no traffic had passed through it yet. After generating traffic with
a ping, the entries appeared:

```
Switch#show mac address-table
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0001.c754.cbcc    DYNAMIC     Fa0/2
   1    00e0.f943.25e9    DYNAMIC     Fa0/1
```

Both MAC addresses match exactly what `ipconfig /all` reported on each PC, mapped
to the correct ports. The `DYNAMIC` type confirms the switch learned them on its
own — no configuration was required.

## Troubleshooting Challenge

### The change

I changed PC1's IP address from `192.168.1.20` to `192.168.2.20`, keeping the same
subnet mask and leaving the cabling untouched.

### The result

```
C:\>ping 192.168.2.20

Request timed out.
Request timed out.
Request timed out.
Request timed out.

Packets: Sent = 4, Received = 0, Lost = 4 (100% loss)
```

```
C:\>arp -a
No ARP Entries Found
```

### Root cause

The failure is purely logical, not physical. Layer 1 and Layer 2 were still fully
functional — the cables were connected and the link lights stayed green.

Before sending anything, PC0 compares its own network with the destination network
using the subnet mask:

```
PC0:         192.168.1.10 /24  ->  network 192.168.1.0
Destination: 192.168.2.20      ->  network 192.168.2.0
```

The networks do not match, so PC0 concluded the destination was remote and needed
to be forwarded to a default gateway. Since no gateway was configured, PC0 had
nowhere to send the packet and dropped it internally.

### Why the ARP table stayed empty

This was the most important detail of the lab. The ARP table was empty **not because
ARP failed, but because ARP never ran at all**.

The host follows this order:

1. Is the destination on my network?
2. If yes, use ARP to resolve its MAC address.
3. If no, forward the packet to the default gateway.

PC0 failed at step 1, so it never reached step 2. No ARP Request was ever sent.

This distinction matters for troubleshooting:

| Empty ARP table because... | Problem type | Where to look              |
|----------------------------|--------------|----------------------------|
| ARP was sent, no reply     | Physical     | Cable, port, device powered|
| ARP never ran              | Logical      | IP, subnet mask, gateway   |

### Switch table after the change

```
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0001.c754.cbcc    DYNAMIC     Fa0/2
```

PC1's MAC address was still present, but PC0's had disappeared.

I initially predicted the opposite — I thought changing the IP would remove PC1's
entry. That was wrong. **The switch operates at Layer 2 and has no awareness of IP
addresses at all.** Changing an IP address cannot affect the MAC address table.

PC0's entry disappeared because of the aging timer: its pings never left the PC, so
it generated no traffic reaching the switch, and the entry expired from inactivity.

### The fix

To make the two networks communicate, two things are required:

1. A **router** connected between them, with one interface in each network.
2. A **default gateway** configured on each PC, pointing to the router's IP address
   in that PC's own network.

## What I Learned

- ARP only runs when a host actually needs to send data. Configuring an IP address
  does not trigger it.
- An empty ARP table means two very different things depending on whether the
  destination is local or remote. That distinction narrows the problem down fast.
- A switch never looks at IP addresses. Changing an IP has zero effect on the MAC
  address table.
- MAC address table entries expire from inactivity, not from configuration changes.
- Without a default gateway, a host will not even attempt to reach a remote network.
  The packet is dropped before it ever reaches the cable.
- `TTL=128` proves the packet never crossed a router, since routers decrement TTL
  at every hop.
- Green link lights only prove Layer 1 and Layer 2 are working. They say nothing
  about whether the network is configured correctly.

## Commands Used

| Command                   | Purpose                                          |
|---------------------------|--------------------------------------------------|
| `ipconfig /all`           | Show IP, subnet mask, gateway and MAC address    |
| `arp -a`                  | Display the ARP cache                            |
| `ping <ip>`               | Test connectivity to a destination               |
| `enable`                  | Enter privileged EXEC mode on the switch         |
| `show mac address-table`  | Display the switch MAC address table             |

## Files

- `lab-01-same-network.pkt` — Packet Tracer topology file
- `screenshots/` — Terminal output and topology captures
