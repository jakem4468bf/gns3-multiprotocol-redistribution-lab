# GNS3 Multi-Protocol Redistribution Lab

A self-built network lab combining IS-IS, RIP v2, EIGRP, OSPF, and BGP into a single end-to-end topology, with full two-way route redistribution across every protocol boundary. The lab proves connectivity by running an actual client-server HTTP request across the entire chain.

Core routing and redistribution are done and verified. project continues from here with security-focused work.

## What this demonstrates

Most labs configure one protocol in isolation. This lab chains five together, with traffic required to flow in both directions across all of them, from a client machine to a web server and back. Getting a single ping through this topology requires every boundary to correctly translate and re-advertise routes without creating loops.

## Topology

Five protocol clusters, connected in sequence:

```
Client PCs (DHCP/static) — IS-IS — RIP v2 — EIGRP — OSPF — BGP — Web Server
```

- **IS-IS cluster:** 4 routers, 3 PC segments
- **RIP v2 cluster:** 3 routers
- **EIGRP cluster:** 3 routers
- **OSPF cluster:** 3 routers
- **BGP cluster:** 3 routers (iBGP, single AS), plus the web server segment

Each boundary between clusters is a dedicated transit link, and the router sitting on that boundary runs both adjoining protocols so it can redistribute between them locally.

## IP addressing

- IS-IS PC segments: 10.10.x.0/24
- IS-IS internal links: 10.10.100.0/30 range
- RIP segment: 10.20.x.0/30
- EIGRP segment: 10.30.x.0/30
- OSPF segment: 10.40.x.0/30
- BGP segment: 10.50.x.0/30, web server subnet 10.50.3.0/24
- Redistribution boundary links: 10.100.1.0/30 through 10.100.4.0/30

## Redistribution and loop prevention

Two-way redistribution at four boundaries means routes can loop back into the protocol they originated from if left unchecked. Solved with a route-tagging scheme, one tag per protocol of origin:

| Protocol | Tag |
|---|---|
| IS-IS | 10 |
| RIP | 20 |
| EIGRP | 30 |
| OSPF | 40 |
| BGP | 50 |

Each boundary router tags routes on the way into its own protocol, preserves tags already present from earlier boundaries, and uses route-maps to block any route from re-entering the protocol it originated from. This keeps an eight-point redistribution scheme (two directions x four boundaries) loop-free.

## Key issues found and fixed

- **IS-IS to RIP redistribution silently dropped PC segments.** Cause: `redistribute isis level-2` only pulls level-2 routes; PC segments were level-1 internal routes. Fixed by redistributing `level-1-2`.
- **IS-IS `set tag` syntax rejected under `router isis`.** IS-IS redistribution doesn't support inline tag keywords the way RIP and OSPF do. Fixed by moving tagging into a route-map.
- **Intermittent missing routes during testing.** Traced to powering off routers to save resources, which aged out their LSPs/routes from neighboring tables. Documented as a testing constraint: all routers must stay powered on during verification.
- **BGP redistribution into OSPF rejected `set tag` in the route-map.** BGP-into-IGP redistribution doesn't support tag-setting the same way; resolved by permitting the route without an explicit set, relying on tags already assigned upstream.
- **BGP routes not propagating past a directly-connected iBGP peer.** Root cause: default iBGP split-horizon behavior blocks a router from re-advertising iBGP-learned routes to another iBGP peer unless it's a route reflector. Fixed by extending route-reflector-client relationships to cover the full peer chain.
- **BGP next-hop unreachable after redistribution.** Classic next-hop-preservation issue: redistributed and reflected routes retained their original next-hop, which downstream iBGP-only routers couldn't resolve. Fixed with `next-hop-self`, and for client-reflected routes specifically, the `next-hop-self all` variant was required since plain `next-hop-self` doesn't rewrite next-hop for routes already learned from another reflector client.
- **iBGP-learned routes silently excluded from IGP redistribution.** The single most important find in this lab. Cisco IOS blocks iBGP-learned routes from being redistributed into an IGP by default, as a built-in loop-prevention measure, regardless of correct redistribute commands and route-maps. Fixed with `bgp redistribute-internal`. This behavior is not obvious from the running-config and only surfaces through targeted verification of advertising-router IDs in the OSPF database.

## Client-server verification

Two Kali Linux VMs were integrated into the topology as the final proof of connectivity:

- **Server VM:** running FastAPI under Uvicorn, serving a basic HTML response on port 8000.
- **Client VM:** using `curl` to send a real HTTP GET request across the full protocol chain.

A successful `curl` response confirms the request and its return traffic correctly traversed IS-IS, RIP, EIGRP, OSPF, and BGP, through all four redistribution boundaries, in both directions.

## Skills demonstrated

- Multi-protocol routing design and configuration: IS-IS, RIP v2, EIGRP, OSPF, BGP (iBGP)
- Two-way route redistribution across four protocol boundaries
- Route tagging and route-map based loop prevention
- BGP route reflection and iBGP propagation troubleshooting
- BGP next-hop-self and next-hop reachability diagnostics
- Diagnosing IOS default behaviors not visible in running-config (iBGP redistribution restriction)
- Structured, methodical network troubleshooting using `show`, `debug`, and targeted route queries rather than guesswork
- GNS3 topology design, including Qemu/VMware VM integration
- Linux system administration (Ubuntu, Kali) for lab support roles: network configuration, package management, Python virtual environments
- Deploying and testing a FastAPI web service, including binding, routing, and cross-network HTTP verification
- End-to-end network verification methodology: ping, traceroute, and application-layer testing layered together

## Tools used

GNS3, Cisco IOS (routers), VMware Workstation, Kali Linux, Python 3, FastAPI, Uvicorn, curl
