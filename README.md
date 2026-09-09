# networking-fundamentals
Hands-on networking labs and notes — building IT support skills from the ground up
# Networking Fundamentals

Hands-on networking labs and study notes, documenting my path from fundamentals
toward an IT support and systems administration career.

Every lab in this repository was built from scratch in Cisco Packet Tracer, tested,
broken on purpose, and documented with the actual output I observed — including the
predictions I got wrong and why.

## Approach

Each lab follows the same structure:

1. **Build** the topology and verify it works
2. **Observe** the underlying protocols with CLI tools rather than assuming
3. **Break** something deliberately and diagnose the failure
4. **Document** the result, including anything that contradicted my expectations

The goal is not to memorise commands. It is to be able to explain *why* a network
behaves the way it does, and to diagnose it in a structured order rather than
guessing.

## Repository Structure

```
networking-fundamentals/
└── phase-1-fundamentals/
    └── labs/
        └── lab-01-same-network/
            ├── README.md
            ├── lab-01-same-network.pkt
            └── screenshots/
```

## Phase 1 — Fundamentals

| Lab | Title | Focus | Status |
|-----|-------|-------|--------|
| [01](phase-1-fundamentals/labs/lab-01-same-network) | Communication Within the Same Network | ARP, MAC learning, subnet mask | ✅ Complete |
| 02 | Connecting Two Networks | Router, default gateway, routing table | 🔜 Planned |
| 03 | DHCP and DNS Services | DORA process, name resolution, APIPA | 🔜 Planned |
| 04 | Inspecting Packets in Simulation Mode | Encapsulation, hop-to-hop delivery | 🔜 Planned |
| 05 | Troubleshooting Hidden Faults | Layered diagnostic methodology | 🔜 Planned |

## Topics Covered So Far

**Layer 2**
Ethernet frames, MAC addressing, switch MAC learning, flooding, aging timers,
broadcast domains

**Layer 3**
IPv4 addressing, subnet masks, CIDR notation, local vs remote destinations,
default gateway, routing tables

**Protocols and Services**
ARP resolution and caching, ICMP, DHCP (DORA), DNS, APIPA

**Troubleshooting**
Bottom-up layered diagnosis, interpreting `ipconfig`, `ping`, `arp -a`, and Cisco
IOS `show` commands, distinguishing physical failures from logical misconfiguration

## Tools

- **Cisco Packet Tracer** — lab environment
- **Cisco IOS CLI** — switch and router configuration
- **Windows command line** — host-side diagnostics

## Notes

Lab files (`.pkt`) are included so anyone can open the exact topology and reproduce
the results.
