---
layout: default
title: "Acknowledgments"
nav_order: 9
permalink: /acknowledgments/
---

# Acknowledgments
{: .no_toc }

Specflow builds on the work of others. This page credits the projects and people whose ideas were adapted, extended, or inspired features in Specflow.
{: .fs-6 .fw-300 }

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## forge by Ikenna N. Okpala

**Repository:** [github.com/ikennaokpala/forge](https://github.com/ikennaokpala/forge)

forge is an AI-assisted development framework that introduced several patterns Specflow has adapted for its contract-driven workflow.

### Specific Adaptations

| forge Concept | Specflow Adaptation |
|---------------|---------------------|
| **Security quality gates** | SEC-001 through SEC-005 default contract rules. forge's approach to embedding security checks into the development loop was adapted into Specflow's YAML contract format with `forbidden_patterns` and `required_patterns`. See [Security & Accessibility Gates](/core-concepts/security-accessibility/). |
| **Accessibility quality gates** | A11Y-001 through A11Y-004 default contract rules. forge's accessibility checks were translated into Specflow contract patterns that run as part of the standard contract test suite. |
| **Self-healing fix loops** | The heal-loop agent. forge's concept of automated violation repair was adapted into Specflow's confidence-tiered fix pattern system with score evolution and the Platinum/Gold/Silver/Bronze tier model. See [Self-Healing Fix Loops](/advanced/self-healing/). |
| **Single-file SKILL.md packaging** | The concept of packaging an entire agent's capabilities into a single markdown file informed Specflow's agent prompt format (`scripts/agents/*.md`). |
| **No-mock testing philosophy** | TEST-001 and TEST-002 contract rules. forge's stance against mock-heavy tests influenced Specflow's default test integrity contracts that flag excessive mocking. |

---

## V3 QE Skill by Mondweep Chakravorty

The **TinyDancer** model routing pattern from the V3 QE Skill introduced the idea of routing different agent tasks to different model tiers based on task complexity.

### Adaptation in Specflow

Specflow's [Model Routing](/agent-system/model-routing/) system assigns each of the 23+ agents to a recommended model tier (Haiku, Sonnet, or Opus), producing a 40-60% reduction in token costs. The routing table and tier definitions were adapted from the TinyDancer pattern.

---

## Timebreez Project

The Timebreez project served as Specflow's primary production validation environment. Over 280+ GitHub issues were delivered using Specflow's wave execution system, providing the real-world feedback that shaped every feature.

### What Timebreez Validated

- Wave orchestration at scale (40+ issue backlogs)
- Three-tier journey gates in practice
- Agent Teams mode with persistent teammates
- Contract completeness enforcement
- Parallel execution savings (3-5x faster than sequential)

---

## What Is Original to Specflow

While Specflow draws on the work above, the following features are original contributions:

| Feature | Description |
|---------|-------------|
| **Contract YAML schema** | The specific YAML format for feature and journey contracts, including `forbidden_patterns`, `required_patterns`, scope negation, severity levels, and compliance checklists |
| **Journey verification hooks** | Git hooks that automatically extract issue numbers from commits, look up journey contracts, and run only the relevant E2E tests |
| **Wave orchestration** | The 8-phase pipeline (analyze, plan, spawn, execute, test, validate, close, report) with dependency-based parallel execution |
| **23+ agent library** | The full set of specialized agents, each with defined triggers, inputs, outputs, and quality gates |
| **Agent Teams mode** | Persistent peer-to-peer teammate coordination via Claude Code TeammateTool API, with issue-lifecycle, db-coordinator, quality-gate, and journey-gate agents |
| **Three-tier journey gates** | Tier 1 (issue), Tier 2 (wave), Tier 3 (regression) enforcement with baseline management and defer journal |
| **Contract completeness enforcement** | CI gate that verifies CONTRACT_INDEX.yml stays in sync with contract files on disk |
| **DPAO methodology** | Discovery, Parallel, Analysis, Orchestration -- the methodology for parallel agent execution |

---

## Academic Foundations

Specflow also builds on established computer science research. See [Background & Academic Foundation](/background/) for the full intellectual lineage, including:

- Design by Contract (Meyer, 1986)
- Property-Based Testing (QuickCheck, Claessen & Hughes, 2000)
- Behavior-Driven Development (North, 2003)
- Continuous Delivery (Humble & Farley, 2010)

---

[View on GitHub](https://github.com/Hulupeep/Specflow)
