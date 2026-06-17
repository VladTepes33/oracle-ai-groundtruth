# 🗄️ Oracle 19c HA/DR — AI Knowledge Base & Prompt Framework

> A structured knowledge base and system prompt framework to turn Claude (or any LLM) into an expert Oracle 19c High Availability and Disaster Recovery assistant — grounded in official Oracle documentation.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Oracle 19c](https://img.shields.io/badge/Oracle-19c-red.svg)](https://docs.oracle.com/en/database/oracle/oracle-database/19/)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

---

## 🎯 What This Is

Most LLM prompts for DBAs are generic. This repo is different:

- **Expert-level system prompt** that instructs the AI to behave as a senior Oracle DBA/architect
- **Structured knowledge base** covering Oracle 19c HA/DR from database layer down to storage/network/OS
- **Mandatory reference system** — every AI response must cite the source document, chapter, and page
- **Designed for Claude Projects** — upload your official Oracle PDFs alongside this knowledge base for a fully grounded assistant

---

## 📦 Contents

```
oracle19c-dba-ai-knowledge/
├── README.md                          ← You are here
├── LICENSE                            ← MIT
├── CONTRIBUTING.md                    ← How to contribute
├── CHANGELOG.md                       ← Version history
│
├── prompts/
│   └── system-prompt.md               ← The system prompt to paste into your AI project
│
├── knowledge-base/
│   └── oracle19c_ha_knowledge_base.md ← Full technical knowledge base (3000+ lines)
│
├── examples/
│   ├── switchover-example.md          ← Example Q&A: RAC+DG switchover procedure
│   ├── asm-troubleshooting-example.md ← Example Q&A: ASM disk group recovery
│   └── ib-interconnect-example.md     ← Example Q&A: InfiniBand tuning for RAC
│
└── docs/
    ├── getting-started.md             ← How to set up your Claude Project
    ├── recommended-pdfs.md            ← Which Oracle PDFs to add to the project
    └── architecture.md                ← How the knowledge base is structured
```

---

## 🚀 Quick Start

### 1. Set up your Claude Project

1. Go to [claude.ai](https://claude.ai) → **Projects** → **New Project**
2. Open `prompts/system-prompt.md` → copy the full content → paste into **Project Instructions**
3. Upload `knowledge-base/oracle19c_ha_knowledge_base.md` as a project document
4. Upload your Oracle 19c official PDFs (see [recommended-pdfs.md](docs/recommended-pdfs.md))

### 2. Ask your first question

```
How do I perform a Data Guard switchover on a RAC+DG environment?
```

Every response will end with a mandatory `📚 References` block citing the exact PDF, chapter, and page used.

---

## 🧠 Knowledge Base Coverage

| Domain | Coverage |
|--------|----------|
| Oracle Clusterware / Grid Infrastructure 19c | ✅ Full |
| Oracle ASM 19c | ✅ Full |
| Oracle RAC 19c (Cache Fusion, Services, TAF, AC) | ✅ Full |
| Oracle Data Guard 19c (Physical, Logical, ADG, Broker) | ✅ Full |
| RAC + Data Guard combined (Far Sync, cascaded) | ✅ Full |
| Oracle GoldenGate 19c (Classic + Microservices) | ✅ Full |
| OS: RHEL / Oracle Linux 7/8/9 | ✅ Full |
| OS: Windows Server | ✅ Partial |
| Network: Ethernet bonding/LACP, VLAN, QoS | ✅ Full |
| Network: InfiniBand / RDMA / RoCE | ✅ Full |
| Storage: FC SAN + Multipath | ✅ Full |
| Storage: iSCSI | ✅ Full |
| Storage: NVMe-oF (RDMA, TCP) | ✅ Full |
| Operational checklists (health check, patching, RMAN) | ✅ Full |
| Troubleshooting (CRS, ASM, RAC, DG, GoldenGate) | ✅ Full |

---

## 💡 How the Reference System Works

Every AI response is structured as follows:

```
The MAXIMUM AVAILABILITY protection mode requires synchronous redo transport [1].
Standby Redo Logs must be sized identically to Online Redo Logs [2].
The FAL_SERVER parameter enables automatic gap resolution [KB].

📚 References
┌──────────────────────────────────────────────────────────────────────┐
│ [1]  Oracle Data Guard Concepts 19c — Chapter 6                      │
│      "Redo Transport Services" — p. 6-4                              │
│ [2]  Oracle Data Guard Concepts 19c — Chapter 7                      │
│      "Standby Redo Log" — p. 7-2                                     │
│ [KB] Knowledge Base oracle19c_ha — Part 3.2 (DG Parameters)         │
└──────────────────────────────────────────────────────────────────────┘
```

If no project document covers the topic, the AI explicitly warns:
> ⚠️ No project document covers this point. Response based on general Oracle 19c knowledge.

---

## 🤝 Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).

Areas where help is most needed:
- Additional troubleshooting scenarios
- GoldenGate Microservices coverage
- Oracle 21c / 23c differences
- CDB/PDB specifics in HA context
- Exadata-specific sections

---

## 📄 License

MIT — see [LICENSE](LICENSE).

---

## ⚠️ Disclaimer

This knowledge base is provided for educational and operational assistance purposes.
Always validate procedures in a test environment before applying to production.
Oracle official documentation always takes precedence.
