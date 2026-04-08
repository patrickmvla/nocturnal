# Nocturnal — Problem Statement & Vision

## Date
2026-04-07

## Problem

The market has created a role called "AI Engineer" but most practitioners lack deep understanding of the two hardest problems in LLM-powered systems:

1. **Prompt Architecture** — How to structure, layer, compose, and inject system prompts so that a model behaves correctly across providers, contexts, and conversation states.

2. **Context Degradation** — How to manage the finite context window as conversations grow: when to prune, when to compress, what to preserve, and how to prevent the model from losing critical instructions over time.

These problems exist in every AI system — from a simple chatbot to an advanced coding agent like Claude Code or OpenCode. Yet most engineers treat them as configuration rather than architecture. They copy prompts, hope for the best, and debug by vibes.

## Vision

Nocturnal is a **production-grade general-purpose agent backend** built from scratch to deeply understand and solve these two problems.

It is not a toy. It is not a tutorial. It is an engineer's deliberate, rigorous study of prompt architecture and context management — implemented as a real system, deployed to real infrastructure, validated by real evals.

## Goals

1. **Build a layered prompt architecture** — provider-aware, composable, cacheable — studying production systems like OpenCode as reference
2. **Implement robust context management** — pruning, compaction, overflow detection — with clear strategies for each tier
3. **Understand every line** — no copy-paste, no black boxes. Every decision documented in ADRs, every design reviewed in RFCs
4. **Ship it** — hosted backend with an API that any UI can plug into
5. **Prove it works** — evals that demonstrate the architecture holds under real conditions

## Non-Goals (for now)

- Building a UI (backend-first, UI plugs in later)
- Competing with existing tools (this is about understanding, not market share)
- Solving a specific end-user problem (the use case will emerge from the understanding)

## Approach

Follow the engineering discipline of top-tier orgs:
- ADRs for every technical decision
- RFCs for system design
- API contracts before implementation
- Phased build plan
- Study production codebases (OpenCode) with intent — borrow what makes sense, redesign what doesn't

## Technical Foundation

- **Runtime**: TypeScript
- **Framework**: Hono
- **LLM Integration**: Multi-provider (Anthropic, OpenAI, etc.)

## Success Criteria

1. A deployed backend that can hold multi-turn conversations with layered prompts
2. Context management that demonstrably handles long conversations without losing critical instructions
3. A complete paper trail of decisions (ADRs) and designs (RFCs)
4. An engineer who understands prompt architecture at the level of the people who built the tools
