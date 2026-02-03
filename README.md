# Specflow Documentation

> **TypeScript rejects type errors. Specflow rejects architecture errors.**

Official documentation site for [Specflow](https://github.com/Hulupeep/Specflow) — architecture contracts for AI-generated code.

**Live Site:** [https://hulupeep.github.io/specflow-help](https://hulupeep.github.io/specflow-help)

---

## What is Specflow?

Specflow enforces architectural contracts like a compiler enforces types. It's designed for the LLM era, where code generation is infinite but human review capacity is not.

### Key Features

- **Feature Contracts**: Architectural invariants that must hold (e.g., "Leave approval MUST debit balance")
- **Journey Contracts**: End-to-end workflows that define "done"
- **Agent-First Workflow**: 18+ specialized LLM agents orchestrated by `waves-controller`
- **Automated Testing**: Playwright E2E tests enforce all contracts
- **DPAO Methodology**: Discovery → Parallel → Analysis → Orchestration (3-4x faster)

---

## Running Locally

This site is built with [Jekyll](https://jekyllrb.com/) and the [Just the Docs](https://just-the-docs.com/) theme.

### Prerequisites

- Ruby 3.1 or higher
- Bundler (`gem install bundler`)

### Setup

```bash
# Install dependencies
bundle install

# Serve the site locally
bundle exec jekyll serve

# View at http://localhost:4000
```

---

## Site Structure

```
specflow-help/
├── index.md (Home page)
├── getting-started/
│   ├── index.md
│   ├── first-wave.md
│   └── ...
├── core-concepts/
│   ├── contracts.md
│   ├── agents.md
│   └── journeys.md
├── agent-system/
│   ├── waves-controller.md
│   ├── agent-reference.md
│   └── ...
├── background/
│   ├── origins.md
│   └── why-now.md
├── advanced/
│   ├── manual-contracts.md
│   └── custom-agents.md
└── reference/
    ├── contract-schema.md
    └── glossary.md
```

---

## Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Quick tips:**
- All code examples must be tested
- Include the compiler analogy on major pages
- Lead with the agent-first workflow (manual setup is advanced)

---

## Deployment

This site deploys automatically to GitHub Pages when changes are pushed to `main`.

**Deployment workflow:** `.github/workflows/deploy.yml`

---

## Positioning

The core message is simple:

> **"Stop trusting. Start verifying."**

Specflow synthesizes:
- Design by Contract (Eiffel, 1986)
- Property-Based Testing (QuickCheck, 2000)
- Static Analysis (Lint, 1970s)

Applied to the new failure mode: **probabilistic code generation** (LLMs).

See [POSITIONING.md](../POSITIONING.md) for the full narrative.

---

## License

MIT License - See [LICENSE](LICENSE) for details.

---

## Questions?

- **Issues:** [Report bugs or request features](https://github.com/Hulupeep/specflow-help/issues)
- **Discussions:** [Ask questions](https://github.com/Hulupeep/specflow-help/discussions)
- **Main Repo:** [Specflow framework](https://github.com/Hulupeep/Specflow)

---

**The compiler doesn't trust your types. Why trust your architecture?**
