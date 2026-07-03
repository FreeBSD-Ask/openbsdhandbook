# Advanced Networking

## Synopsis

This section presents advanced, production-ready patterns for building and operating complex OpenBSD networks. It focuses on high availability, multi-homing, tunneling, IPv6 at scale, quality of service, MPLS, large-scale network services, virtualization-aware designs, L2/L3 architecture, telemetry, hardening, reference topologies, and troubleshooting playbooks. Each chapter provides concise design guidance, annotated configurations, verification steps, and operator-focused troubleshooting procedures.

## Relation to “Networking” and “OpenBGPD”

The **Networking** section covers foundational interfaces, routing, and packet filtering concepts suitable for most single-host or small deployments. **Advanced Networking** builds on those foundations to address redundancy, scale, and operational safety across larger environments.

The **OpenBGPD** section focuses on external and internal BGP policy, peering, and route distribution. Where appropriate, this section references BGP as a control plane input (for example, anycast services or MPLS VPNs), but defers detailed BGP policy and session management to the dedicated **OpenBGPD** chapter at `/openbgpd`.

## Contents

The following chapters are ordered for incremental complexity. Each item links to its chapter and includes a one-line summary.

- **[High Availability and State Replication](high-availability.md)**— Redundant gateways and firewalls using CARP, pfsync, relayd, and event-driven failover.
- **[Multi-WAN and Policy-Based Routing](multi-wan-policy-routing.md)**— Route-to, reply-to, monitoring, and inbound considerations for multi-provider edges.
- **[VPN and Cryptographic Tunneling](vpn-and-crypto-tunneling.md)**— IKEv2 with iked, WireGuard-style tunnels, NAT traversal, and performance tuning.
- **[Classic and Lightweight Tunnels](classic-tunnels.md)**— gif, gre, and etherip for pragmatic overlays and migrations.
- **[IPv6 at Scale](ipv6-at-scale.md)**— Router Advertisements, ND, prefix delegation, NPTv6, and large-segment IPv6 operations.
- **[QoS and Traffic Shaping](qos-traffic-shaping.md)**— Classification and queuing with PF for latency-sensitive and bulk traffic.
- **[MPLS and Label Distribution](mpls.md)**— mpls(4) data plane with LDP via ldpd, PE interfaces (mpe), and L2 VPLS pseudowires.
- **[Network Services at Scale](network-services-at-scale.md)**— Authoritative DNS with nsd, validating recursion with unbound, DHCP roles, and time services.
- **[Virtualization and Host Networking](virtualization-networking.md)**— vmd, tap/vether, bridging/VLANs, and micro-segmentation with PF anchors.
- **[Large-Scale L2 and L3 Design](l2-l3-design.md)**— VLAN strategy, routed access, first-hop redundancy, and anycast service placement.
- **[Telemetry, Logging, and Flow Export](telemetry-logging-flow.md)**— pflog analysis, pflow (NetFlow/IPFIX), snmpd, and time-series integration.
- **[Hardening and Operational Safety](hardening-operations.md)**— Anti-spoofing, RPF, secrets management, and safe rollout procedures.
- **[Reference Architectures](reference-architectures.md)**— End-to-end designs: redundant Internet edge, campus/branch, and POP patterns.
- **[Troubleshooting Playbooks](troubleshooting-playbooks.md)**— Diagnosing HA flips, asymmetric paths, tunnel MTU issues, IPv6 ND, and QoS verification.

## How to Use This Section

1. Read chapters in order on a first pass to establish common patterns and terminology.
2. For implementation, jump directly to the relevant chapter based on your objective (for example, HA, IPv6, or QoS).
3. Apply the **Design Considerations** before touching configuration.
4. Use the **Verification** steps immediately after deployment to confirm behavior.
5. Keep the **Troubleshooting** playbooks nearby during rollouts and incident response.

## Next Steps

Proceed to **[High Availability and State Replication](high-availability.md)**, then continue through the chapters in order for a complete tour. You can also jump directly to the chapter that matches your current task.
