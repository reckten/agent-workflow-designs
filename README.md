# Agent Workflow Designs

A collection of agent workflow definitions, Claude skills, and AI pipeline documentation from production engineering work.

These artifacts represent real systems built to automate complex engineering tasks — designed with deliberate phase structure, defined guardrails, and explicit handoffs between stages.

## Contents

| File | Type | Summary |
|---|---|---|
| [`skills/source-driven-pom-agent.md`](skills/source-driven-pom-agent.md) | Claude Skill | 8-phase agent that scaffolds Playwright page objects directly from application source code. Reduces per-page POM scaffolding from 2-3 days to under an hour. |
| [`skills/api-capture-agent.md`](skills/api-capture-agent.md) | Claude Skill | Multi-agent pipeline that captures API payloads via HAR, validates against Java backend source, and packages a clean handoff for the api-generator-agent. Reduces API discovery from 40-90 minutes to under 15. |
| [`pipelines/graphrag-pipeline.md`](pipelines/graphrag-pipeline.md) | Pipeline | GraphRAG system that extracts domain entities from product documentation, builds a traversable knowledge graph, and exports Claude-consumable context via hybrid semantic + graph retrieval. |
| [`pipelines/shift-left-monitor.md`](pipelines/shift-left-monitor.md) | Monitor | Scheduled, event-driven, and on-demand monitor that cross-references product source changes against the test suite, scores risk level, and produces structured impact reports before CI failures surface. |

## About

These workflows were built as part of an AI-forward engineering practice — treating agents not as chat assistants but as structured pipeline workers with defined inputs, outputs, safety rails, and failure modes.

Each workflow is designed to be composable, inspectable, and opinionated: it makes deliberate decisions so engineers don't have to re-litigate them on every run.
