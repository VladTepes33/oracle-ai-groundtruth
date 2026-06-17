# Knowledge Base Architecture

## How It Is Structured

The knowledge base (`oracle19c_ha_knowledge_base.md`) is organized in 11 parts,
covering every layer from the database down to the physical infrastructure.

```
Part 1  — Oracle Grid Infrastructure 19c
          Architecture, components, OCR, Voting Disk, ASM, SRVCTL, CRSCTL

Part 2  — Oracle RAC 19c
          Cache Fusion, GCS/GES, init parameters, services, TAF,
          Application Continuity, monitoring (GV$ views), AWR RAC

Part 3  — Oracle Data Guard 19c
          Physical/Logical/Snapshot Standby, protection modes,
          init parameters (primary + standby), TNS, Broker (DGMGRL),
          monitoring views, switchover/failover procedures, ADG

Part 4  — RAC + Data Guard Combined
          Topologies, Far Sync, cascaded standby, SRL for multi-thread,
          SRVCTL role-based services

Part 5  — Oracle GoldenGate 19c
          Architecture, processes, GGSCI commands, Extract/Replicat params,
          topologies (uni/bi-directional, cascade, consolidation),
          Microservices Architecture

Part 6  — Troubleshooting
          CRS/GI diagnostics, ASM issues, RAC interconnect problems,
          Data Guard lag/gap resolution, GoldenGate abend handling

Part 7  — Operational Checklists
          Daily health check (SQL scripts), patching procedure (GI + RDBMS),
          RMAN backup strategy with Data Guard

Part 8  — Quick Reference
          Default ports, environment variables, MOS Notes index,
          important log file paths

Part 9  — System: RHEL / OEL & Windows Server
          RPM prerequisites, kernel parameters, user/group structure,
          Huge Pages, THP disabling, systemd services, OS monitoring

Part 10 — Network
          Ethernet bonding/LACP, NIC tuning (MTU, ring buffers, IRQ affinity),
          VLAN architecture for Oracle, QoS for redo transport,
          InfiniBand (OFED, OpenSM, RDMA, RoCE), benchmarking

Part 11 — Storage
          FC SAN (HBA, WWN, multipath.conf, zoning),
          iSCSI (Open-iSCSI, CHAP, optimizations),
          NVMe-oF (RDMA, TCP, native multipath, udev rules),
          ASM disk groups layout, storage benchmarking
```

## Design Principles

**Layered coverage** — every topic goes from concept to commands to troubleshooting.

**Production-ready commands** — every command includes options and execution context
(which user, which node, pre/post checks).

**Cross-references** — the knowledge base is referenced as `[KB]` in AI responses,
with the specific part number (e.g. `[KB] Part 3.5`).

**PDF priority** — the knowledge base is the fallback. When official Oracle PDFs
are uploaded to the project, they take precedence and are cited first.

## Versioning

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-06 | Initial release — all 11 parts |
