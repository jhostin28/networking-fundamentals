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

<img width="523" height="208" alt="01-topology" src="https://github.com/user-attachments/assets/dc284383-0dac-42b2-832c-2e0b5799ad42" />


## Addressing Table

| Device | Interface | IP Address   | Subnet Mask   | Default Gateway |
|--------|-----------|--------------|---------------|-----------------|
| PC0    | Fa0       | 192.168.1.10 | 255.255.255.0 | Not configured  |
| PC1    | Fa0       | 192.168.1.20 | 255.255.255.0 | Not configured  |

The default gateway was intentionally left empty to prove that hosts in the same
network do not need a router to communicate.

<img width="521" height="397" alt="02-pc0-ip-config" src="https://github.com/user-attachments/assets/dd6312db-b2ff-4e48-8f50-b6b284d6ff74" />

<img width="542" height="412" alt="03-pc1-ip-config" src="https://github.com/user-attachments/assets/946ecfa3-0702-439d-8ad9-12830ce157f4" />


## Hardware Notes

| Device   | Model           | Details                                         |
|----------|-----------------|-------------------------------------------------|
| Switch0  | WS-C2960-24TT-L | 24 FastEthernet ports, 2 Gigabit Ethernet ports |
| PC0      | PC-PT           | MAC address: 00E0.F9C7.DE49                     |
| PC1      | PC-PT           | MAC address: 00D0.BA55.7B05                     |

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

<img width="547" height="406" alt="04-arp-empty" src="https://github.com/user-attachments/assets/b56dcf5f-dec7-4443-b6fc-3aafc4908dc5" />


The ARP table was empty even though both PCs were configured and cabled correctly.
This confirms that ARP only runs when a host actually needs to send data.
Configuring an IP address does not generate traffic by itself.

### Ping test and ARP resolution

```
C:\>ping 192.168.1.20

Reply from 192.168.1.20: bytes=32 time=1ms TTL=128
Reply from 192.168.1.20: bytes=32 time=1ms TTL=128
Reply from 192.168.1.20: bytes=32 time<1ms TTL=128
Reply from 192.168.1.20: bytes=32 time<1ms TTL=128

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)

C:\>arp -a
  Internet Address      Physical Address      Type
  192.168.1.20          00d0.ba55.7b05        dynamic
```

<img width="636" height="463" alt="05-ping-success-arp-populated" src="https://github.com/user-attachments/assets/64f6c369-480d-44b9-bbdc-5cb3c952dc62" />


This single capture shows the full ARP cycle: empty table, successful communication,
populated table. PC0 sent an ARP Request as a broadcast, PC1 replied with its MAC
address, and PC0 cached the result. The `dynamic` type means the entry was learned,
not manually configured, and it will expire after a timeout.

All four packets succeeded. I expected the first packet to time out while ARP
resolved the destination MAC address, but Packet Tracer completed the ARP exchange
fast enough that no timeout occurred.

`TTL=128` is the default starting value on Windows hosts. Because it was not
decremented, this confirms the packet did not cross a router.

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
   1    00d0.ba55.7b05    DYNAMIC     Fa0/2
   1    00e0.f9c7.de49    DYNAMIC     Fa0/1
```

<img width="615" height="452" alt="06-switch-mac-address-table" src="https://github.com/user-attachments/assets/b6499754-47e4-43f2-ae70-eca75fe8114e" />


Both MAC addresses match exactly what `ipconfig /all` reported on each PC, mapped
to the correct ports. The `DYNAMIC` type confirms the switch learned them on its
own — no configuration was required.

## Troubleshooting Challenge

### The change

I changed PC1's IP address from `192.168.1.20` to `192.168.2.20`, keeping the same
subnet mask and leaving the cabling untouched.

### The result

Two different pings were tested, and both failed with identical output:

```
C:\>ping 192.168.1.20          <- IP that no longer exists on the network
Request timed out. (x4)        100% loss

C:\>ping 192.168.2.20          <- IP on a different network
Request timed out. (x4)        100% loss

C:\>arp -a
  Internet Address      Physical Address      Type
  192.168.1.20          00d0.ba55.7b05        dynamic
```

<img width="857" height="625" alt="07-ping-failures-comparison" src="https://github.com/user-attachments/assets/4e2b8933-9675-44a5-b56d-e8e598b692cb" />


### Same symptom, different causes

This was the most valuable part of the lab. Both pings produced the exact same
output on screen, but the underlying failures were completely different:

| Ping target    | Why it failed                                      | Did ARP run? |
|----------------|----------------------------------------------------|--------------|
| 192.168.1.20   | Local network, but no host holds that IP anymore   | Yes, no reply |
| 192.168.2.20   | Different network, and no default gateway is set   | No, never ran |

The ARP table proves it. After pinging `192.168.2.20`, that address does **not**
appear in the cache — only the stale entry for `192.168.1.20` from the earlier
successful ping. PC0 never asked for the MAC address of `192.168.2.20`.

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

### Why ARP never ran

The host follows this order:

1. Is the destination on my network?
2. If yes, use ARP to resolve its MAC address.
3. If no, forward the packet to the default gateway.

PC0 failed at step 1, so it never reached step 2. No ARP Request was ever sent.

This distinction matters for troubleshooting:

| Empty ARP entry because... | Problem type | Where to look               |
|----------------------------|--------------|-----------------------------|
| ARP was sent, no reply     | Physical     | Cable, port, device powered |
| ARP never ran              | Logical      | IP, subnet mask, gateway    |

### Switch behaviour during the change

The switch MAC address table was unaffected by the IP change. **The switch operates
at Layer 2 and has no awareness of IP addresses at all**, so a Layer 3
reconfiguration cannot alter what it has learned.

Entries only disappear from the table when the aging timer expires after a period
of inactivity, when the switch reboots, or when the table is cleared manually.

### The fix

To make the two networks communicate, two things are required:

1. A **router** connected between them, with one interface in each network.
2. A **default gateway** configured on each PC, pointing to the router's IP address
   in that PC's own network.

## What I Learned

- ARP only runs when a host actually needs to send data. Configuring an IP address
  does not trigger it.
- Two completely different failures can produce identical output on screen. The ARP
  table is what tells them apart.
- An empty ARP entry means very different things depending on whether the
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
