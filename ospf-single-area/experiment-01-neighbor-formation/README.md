# Experiment 01 — OSPF Neighbor Formation After Process Reset

## Objective

Observe how Cisco IOS rebuilds OSPF adjacencies after the OSPF process is reset, and correlate the router's adjacency debug output with the packet sequence captured in Wireshark.

This is a three-router, single-area OSPF lab. All links and loopbacks shown in the topology are in Area 0. The captured console evidence is from R1.

## Topology and addressing

![Three-router Area 0 topology](evidence/topology-three-router-area-0.png)

| Router | Interface / link | Address shown in the topology |
|---|---|---|
| R1 | Gi0/0 to R2, 10.0.12.0/30 | 10.0.12.1/30 |
| R1 | Gi0/1 to R3, 10.0.13.0/30 | 10.0.13.1/30 |
| R2 | Link to R1, 10.0.12.0/30 | 10.0.12.2/30 |
| R2 | Link to R3, 10.0.23.0/30 | 10.0.23.2/30 |
| R3 | Link to R1, 10.0.13.0/30 | 10.0.13.2/30 |
| R3 | Link to R2, 10.0.23.0/30 | 10.0.23.1/30 |
| R1 | Loopback0 | 1.1.1.1/32 |
| R2 | Loopback0 | 2.2.2.2/32 |
| R3 | Loopback0 | 3.3.3.3/32 |

## Experiment

The lab was already configured: OSPF neighbors were FULL and router reachability had been verified. With adjacency debugging active on R1 and a packet capture running on the R1–R2 link, the OSPF process was reset:

```cisco
R1# clear ip ospf process
Reset ALL OSPF processes? [no]: yes
```

The reset was confirmed in the console output. R1 reported its interfaces going down for OSPF and both neighbors, 2.2.2.2 and 3.3.3.3, moving from FULL to DOWN.

## Observed sequence

The capture and console output show the database synchronization sequence that followed the reset:

1. **Hello and neighbor discovery** — R1 and its peers exchanged Hellos. R1's capture showed a Hello from `10.0.12.1` to `224.0.0.5`. The captured R2 Hello was from `10.0.12.2` to `10.0.12.1`; it showed R2 as DR and R1 as an active neighbor.
2. **ExStart / DBD negotiation** — The routers began Database Description (DBD) exchange. The capture included empty DBD packets used during master/slave negotiation and an initial sequence number. R1's adjacency debug reported that it became the slave in the displayed exchanges.
3. **Exchange** — DBD packets carried LSA header summaries, including Type 1 (Router) and Type 2 (Network) LSAs. The debug then reported exchange completion with neighbors 2.2.2.2 and 3.3.3.3.
4. **Loading** — R1 sent LS Requests for Type 1 and Type 2 LSAs. R2 sent the requested LSA information in an LS Update, and R1 sent an LS Acknowledgment.
5. **FULL** — R1's debug reported synchronization with both neighbors and transitions from LOADING to FULL (Loading Done).

```text
clear ip ospf process
        ↓
Neighbor state FULL → DOWN
        ↓
Hello / neighbor discovery
        ↓
ExStart: DBD master-slave negotiation
        ↓
Exchange: DBD LSA summaries
        ↓
Loading: LS Request → LS Update → LS Acknowledgment
        ↓
Neighbor state FULL
```

## Cisco debug observations

The adjacency debug provides the router-side view of the same convergence:

- Both adjacencies were reported as FULL to DOWN during the process reset.
- R1 logged DR/BDR election activity on the links. On the R1–R2 link, the output identifies R2 (2.2.2.2) as DR and R1 (1.1.1.1) as BDR. On the R1–R3 link, it identifies R3 (3.3.3.3) as DR and R1 (1.1.1.1) as BDR.
- R1 logged DBD negotiation, becoming the slave in the shown exchanges, then building the summary list and completing Exchange.
- R1 sent LS Requests, received LS Updates, and logged synchronization with both neighbors to FULL.

Evidence: [process reset](evidence/01-clear-ospf-process.png) · [adjacency debug](evidence/02-cisco-ospf-adjacency-debug.png)

## Wireshark findings

| OSPF packet | What the captured evidence shows |
|---|---|
| Hello | R1 source 10.0.12.1 to 224.0.0.5; R2 source 10.0.12.2 to 10.0.12.1. The R2 Hello identifies R2 as DR and lists R1 as an active neighbor. |
| Database Description (DBD) | Empty DBD packets during ExStart negotiation, followed by DBD summaries including Type 1 and Type 2 LSA information during Exchange. |
| Link State Request | R1 requests Type 1 and Type 2 LSAs. |
| Link State Update | R2 sends requested LSA information. |
| Link State Acknowledgment | R1 acknowledges the LSAs. |

Evidence: [DBD exchange](evidence/03-wireshark-dbd-exchange.png) · [LS Request](evidence/04-wireshark-ls-request.png) · [LS Update](evidence/05-wireshark-ls-update.png) · [LS Acknowledgment](evidence/06-wireshark-ls-acknowledgment.png)

## Final verification

The saved final-state screenshots show:

- R1's OSPF neighbors in FULL state.
- OSPF-learned routes for the other routers' loopbacks and transit networks.
- The final OSPF database.

Evidence: [FULL adjacency](evidence/07-final-full-adjacency.png) · [OSPF routes](evidence/08-final-routing-table.png) · [OSPF database](evidence/09-final-ospf-database.png)

## Takeaway

`clear ip ospf process` reset the OSPF state and caused the established adjacencies to drop. OSPF then rebuilt neighbor relationships, synchronized database summaries, requested and received missing LSAs, acknowledged the updates, and returned the neighbors to FULL. The Cisco debug and packet capture show complementary views of this process: internal state changes on R1 and the OSPF packets exchanged on the link.
