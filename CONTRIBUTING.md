# Contributing

Contributions are welcome and appreciated.

## How to Contribute

### Reporting Issues

Use GitHub Issues to report:
- Incorrect or outdated technical information
- Missing coverage on a topic
- Broken links in `recommended-pdfs.md`
- Improvements to the system prompt behavior

### Submitting Changes

1. Fork the repository
2. Create a branch: `git checkout -b fix/your-topic` or `feat/your-topic`
3. Make your changes
4. Submit a Pull Request with a clear description

### Contribution Guidelines

**For knowledge base changes (`oracle19c_ha_knowledge_base.md`)**
- All technical content must be verifiable against official Oracle 19c documentation
- Include the MOS Note number if referencing a known bug or patch
- Maintain the existing structure (concept → commands → verification → troubleshooting)
- Commands must include the execution context (which user, which node)

**For the system prompt (`prompts/system-prompt.md`)**
- Changes must be tested against several realistic DBA questions before submitting
- The mandatory References block behavior must not be weakened

**For examples (`examples/`)**
- Examples must reflect realistic DBA scenarios
- Reference block must be included and correctly formatted

## Areas Where Help Is Most Needed

- [ ] Additional troubleshooting scenarios (real-world cases)
- [ ] GoldenGate Microservices deeper coverage
- [ ] Oracle 21c / 23c delta documentation
- [ ] CDB/PDB specifics in HA context (Active Data Guard with PDBs)
- [ ] Exadata-specific sections (Smart Scan, cell offloading, IORM)
- [ ] English translation of the knowledge base
- [ ] Additional example Q&As

## Code of Conduct

Be respectful. Technical disagreements should reference Oracle documentation.
