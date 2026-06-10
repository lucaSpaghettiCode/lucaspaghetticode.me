## Context

Most home networks are one flat L2 domain where a smart lightbulb, a work laptop, and a guest's phone are all peers. Mine used to be too. This project replaced it with something closer to a small enterprise edge: segmented, default-deny, monitored — and, because the best way to learn a routing protocol is to be responsible for one, peered over eBGP with a friend's homelab AS.

The word *autonomous* in the title is doing double duty. It's autonomous in the operational sense — the network heals, alerts, and updates without me babysitting it — and in the literal BGP sense: it behaves like a tiny autonomous system, with its own ASN, its own announced prefixes, and its own routing policy.

---

## Topology

```
Internet (FTTH, PPPoE over VLAN 7)
        │
  Protectli VP2420 ── OPNsense 24.7
  (4× Intel i226-V 2.5GbE)
        │ 802.1Q trunk
  Mikrotik CRS326-24G-2S+
        │                    ├─ SFP+ #1 ── NAS      (10G DAC)
        │                    └─ SFP+ #2 ── desk     (10G DAC)
  2× Ubiquiti U6 Pro (PoE, trunked)
```

The **VP2420** (Celeron J6412, four Intel i226-V 2.5GbE ports) terminates the FTTH line. German fibre detail that costs everyone an evening: Deutsche Telekom requires the PPPoE session to ride **VLAN 7** on the WAN port, so the "WAN interface" in OPNsense is actually a VLAN child interface with PPPoE on top — a stacking detail that matters again later, when Suricata enters the picture.

The **CRS326** runs RouterOS with **bridge VLAN filtering** — the one correct way to do VLANs on RouterOS; the legacy per-port methods bypass the switch chip. Its Marvell 98DX3236 can hardware-offload L3 for directly connected hosts (it cannot offload prefix routes — the FIB on this chip doesn't do LPM), but I deliberately run it as **pure L2**. If the switch routes between VLANs, the firewall never sees that traffic, and the entire security model evaporates for the convenience of one hop. Policy lives at the firewall; the switch forwards frames.

The cost of router-on-a-stick is that inter-VLAN traffic hairpins through a 2.5GbE trunk port. This is fine, because the design keeps heavy east-west traffic *within* a VLAN: the NAS and the workstation sit on the same storage-facing segment, linked through the two SFP+ cages at 10G, and their traffic crosses only the switch fabric.

---

## Segmentation: Four VLANs, Default Deny

| VLAN | Name    | Trust model |
|------|---------|-------------|
| 10   | mgmt    | Switch, APs, IPMI. Reachable only from one workstation. |
| 20   | trusted | Laptops, workstation, NAS. |
| 30   | iot     | Things that phone home. No east-west, no LAN-bound initiation. |
| 40   | guest   | Internet only, client isolation on the AP side. |

Every inter-VLAN rule is an explicit allow on top of default-deny, and the allows are narrow: *iot → MQTT broker, TCP/8883, that host only* — not "iot can reach trusted". The recurring tax of this design is **multicast discovery**: AirPlay and Chromecast assume one L2 domain. The `mdns-repeater` plugin reflects mDNS between trusted and iot, which re-enables discovery while the actual media streams still have to pass the firewall rules. It's the standard compromise, and it's the right one — but it's worth being honest that mDNS reflection is a small, deliberate hole in the isolation story.

Wi-Fi maps onto this directly: the two **U6 Pro** units broadcast three SSIDs (trusted, iot, guest), each tagged to its VLAN at the AP, trunked through the switch. The UniFi controller runs self-hosted in a container — no cloud account in the control plane. Radio settings are boring and intentional: 80 MHz on 5 GHz with the two APs on non-overlapping channels, 20 MHz fixed on 2.4 GHz, minimum RSSI set so devices roam instead of clinging.

---

## Suricata: Where IPS Actually Works

OPNsense runs **Suricata inline** using netmap. The naive deployment — IPS on the WAN — fails twice here: netmap can't sit inline on a PPPoE interface, and even if it could, it would inspect traffic pre-NAT, so every alert would point at the router's own address. So Suricata runs inline on the **internal interfaces**, where flows carry real LAN source addresses and the PPPoE/VLAN stacking is invisible.

The ruleset is **ET Open**, pruned hard. Out of the box it's tens of thousands of signatures, many irrelevant (Windows-server exploits headed for a network with no Windows servers) and some actively harmful as false positives. The active policy is ~6,000 signatures: full **IPS (drop)** on the iot VLAN — the segment whose devices I trust least and whose traffic is most predictable — and **IDS (alert-only)** on trusted, because dropping a false positive on my own work traffic costs more than the marginal protection. The J6412 holds line-rate on the trusted segments in IDS mode; full inline inspection is the budget I spend only where the threat model justifies it.

DNS is **Unbound** doing full recursion with DNSSEC validation — no forwarding to Google or Cloudflare — plus blocklists at the resolver level. Every VLAN gets DNS from the firewall and *only* from the firewall; outbound 53/853 from anything else is dropped, which is the only thing that makes resolver-level policy mean anything (hardcoded `8.8.8.8` in IoT firmware is endemic).

---

## The BGP Session

The part of the project that exists purely because routing protocols are better learned in anger. A friend runs his own homelab AS; we built a **WireGuard tunnel** between our edges and run **eBGP across it**, using the FRR plugin (`os-frr`) on OPNsense.

- **ASNs** from the RFC 6996 32-bit private range — we are not troubling any RIR.
- Each side announces an **aggregate of its lab prefixes** (RFC 1918 v4 summaries and ULA v6 `fd00::/8` prefixes — nothing here touches the DFZ, ever).
- **Prefix-lists in both directions**: I accept only his registered aggregates, he accepts only mine. A route-map rejects anything carrying a third ASN in the path, so neither of us can accidentally become transit if one of us peers with someone else.
- **BFD over the tunnel**, 300 ms intervals: when the tunnel dies, routes withdraw in under a second instead of waiting out the BGP hold timer.

What this buys, practically: his prefixes appear in my routing table as eBGP routes and vice versa, no static routes to maintain, and the **syncoid replication** for the [NAS project](#project/homelab-nas) rides this path — `zfs send` to a prefix learned via BGP, over WireGuard, surviving endpoint changes on either side without anyone editing a config. What it actually bought: a working education. The first time the session flapped and took the backup path with it, I learned more about hold timers, session re-establishment, and why BFD exists than any amount of reading had taught me.

---

## Operations

The autonomous part. **Prometheus** scrapes node and interface metrics from the firewall and switch; **Grafana** dashboards show per-VLAN throughput, Suricata alert rates, BGP session state, and PPPoE reconnects (Telekom's forced 24h-ish resets show up as a tidy vertical line). Alerting goes to my phone for exactly three things: WAN down longer than the PPPoE re-dial, BGP session down longer than five minutes, and any Suricata *drop* on the trusted VLAN — events I would act on. Everything else is a dashboard, not a page. Config is backed up on every change; OPNsense updates are applied manually, on a weekend, after reading the changelog — because the deliberate machine is the one whose operator knows what changed.
