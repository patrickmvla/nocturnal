# Production Problems in LLM-Based Systems — Research

## Date
2026-04-07

## Purpose
Inform Nocturnal's architecture by understanding what actually breaks in production.

---

## The Numbers That Matter

| Problem | Impact | Source |
|---------|--------|--------|
| Agent production failure rate | 41-86% | Composio |
| AI pilot failure rate | 95% | MIT/Fortune |
| Multi-agent error amplification | up to 17.2x | Google DeepMind |
| Multi-turn performance drop | 39% | Research |
| Context middle accuracy drop | 30%+ | Chroma |
| Tool output vs user message tokens | 100x | Shopify |
| Production agents needing human within 10 steps | 68% | dbreunig.com |

---

## 1. Context Degradation Is Real and Measured

- **U-shaped attention**: Models are accurate at the start and end of context, but **30%+ less accurate for information in the middle**
- **Cost explosion**: One company went from $2K to $47K/month because conversations carried 150K tokens of history — 10-token responses cost $8 each
- **Multi-turn decay**: 39% performance drop in multi-turn conversations
- **What eats context**: System prompts (500-5K), conversation history (500-2K per turn), RAG docs (5-50K), tool schemas (500-2K). After 50 exchanges: 50K+ tokens
- **Latency correlation**: Small (<5K) = 1-3s, Medium (10-30K) = 3-8s, Large (50K+) = 8-20s. Power users get 5x worse latency

## 2. Prompt Architecture Failures

- **Attribution errors**: Zalando's LLM blamed technologies just because they appeared in text, not because they caused problems. ~10% error rate persisted even with Claude Sonnet
- **Provider differences are real**: OpenAI prefers markdown, Anthropic prefers XML. Saying "think hard" in GPT-5 literally triggers the reasoning model
- **Prompts don't scale**: Prompt tweaks alone don't fix reliability — retrieval, tool calling, and verification layers are required
- **No version control**: Teams editing prompts directly in production without tracking changes

## 3. Agentic Loop Failures

- **Infinite loops**: GetOnStack had agents loop for 11 days undetected, burning $47K. Agent A asked Agent B, which asked Agent A
- **Tool overload**: Adding more tools actually worsened performance — excessive false positives until developers stopped trusting it
- **Cost amplification**: Agents consume 4x more tokens than chat, up to 15x in multi-agent. Tool outputs are 100x larger than user messages
- **Agents take shortcuts**: "This test doesn't pass? Let's skip it" — trading correctness for task completion

## 4. Multi-Agent Error Amplification

- Google DeepMind tested 180 configurations: unstructured multi-agent networks amplify errors up to 17.2x
- Failure breakdown: specification (42%), coordination (37%), verification (21%)
- Natural language agent-to-agent communication passes validation even when semantically wrong
- 40% of multi-agent pilots fail within 6 months

## 5. Cost and Latency

- 40% of orgs spend $250K+/year on LLM initiatives
- Output tokens cost 3-5x more than input
- Delays beyond 2-3 seconds cause user abandonment
- **Wins**: Care Access cut costs 86% through prompt caching. Robinhood went from 55s to <1s with hierarchical tuning

## 6. Eval and Testing

- **The vibes trap**: NurtureBoss tested a few inputs, deployed if it "looked good" — three failure categories were invisible without systematic eval
- 52.4% run offline evals, only 37.3% do online evals
- Most LLM apps are tested less rigorously than login forms
- **What works**: Binary PASS/FAIL over scales, code-based evals for deterministic outputs, LLM-as-judge for subjective, evals on every commit

## 7. Model Switching Pain

- Each provider switch = days/weeks of dev time
- What breaks: prompt formatting, output parsing, token costs, reasoning quality, tool calling schemas
- Provider details leak into prompts, tools, logging, cost controls

---

## What Production Teams Wish They'd Built

1. **Durable execution from day one** — resume from failure points, not restart
2. **Tool masking over tool abundance** — expose 3 relevant fields from 100-field APIs
3. **Circuit breakers** — stop at P95 cost or turn limits, fail gracefully
4. **Just-in-time context** — inject context when needed, remove when done (not kitchen-sink)
5. **Model abstraction layer** — switching providers shouldn't require a rewrite
6. **Observability as architecture** — 94% of production agents have it; those without can't diagnose failures
7. **Progressive autonomy** — suggestions first, autonomous actions for high-confidence only

---

## Sources

- ZenML: 1,200 Production Deployments (zenml.io)
- LangChain: State of Agent Engineering (langchain.com)
- Chroma: Context Rot Research (trychroma.com)
- Context Window Management at Scale (ilovedevops.substack.com)
- Google DeepMind: 17x Error Trap (towardsdatascience.com)
- Pragmatic Engineer: LLM Evals (newsletter.pragmaticengineer.com)
- Sketch.dev: Agent Loop (sketch.dev/blog)
- Fortune: AI Agents Reliability Lagging (fortune.com)
- OWASP ASI08: Cascading Failures (adversa.ai)
- VentureBeat: Model Migration Costs (venturebeat.com)
