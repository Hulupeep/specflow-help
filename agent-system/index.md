---
layout: default
title: Agent System
nav_order: 4
has_children: true
permalink: /agent-system/
---

# Agent System
{: .no_toc }

23+ specialized agents orchestrated by waves-controller.
{: .fs-6 .fw-300 }

---

## Overview

Specflow uses an **agent-first approach** powered by the **DPAO methodology** (Discovery → Parallel → Analysis → Orchestration):

- **waves-controller**: Orchestrates all agents (8 phases)
- **23+ specialized agents**: Each with specific triggers, inputs, outputs
- **Parallel execution**: 3-5x faster than sequential workflows
- **Agent Teams**: Persistent peer-to-peer teammates via Claude Code 4.6 TeammateTool API

---

## Documentation

- **[waves-controller Deep Dive](/agent-system/waves-controller/)** — 8-phase orchestration methodology
- **[Agent Reference](/agent-system/agent-reference/)** — All 23+ agents and when to use them
- **[Agent Teams](/agent-system/agent-teams/)** — Persistent teammate coordination (new)
- **[DPAO Methodology](/agent-system/dpao/)** — How parallel execution works
- **[Customizing Agents](/agent-system/customizing/)** — Edit agent prompts for your workflow

[View on GitHub](https://github.com/Hulupeep/Specflow)
