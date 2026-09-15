# LangGraph 1.0 vs 0.x: What Was Added

## Introduction

LangGraph 1.0 was officially released on October 20, 2025, following an alpha announcement in early September 2025 [1][2][3]. Rather than being a feature-packed "big bang" release, LangGraph 1.0 is explicitly characterized by its maintainers as a **stability-focused release for the agent runtime** [4]. Its purpose is to refine type safety, documentation, and developer ergonomics while keeping the core graph APIs and execution model unchanged [4], which makes upgrading from the 0.x line straightforward [4].

LangGraph 1.0 was designed to work hand-in-hand with LangChain 1.0, whose `create_agent` function is built on top of LangGraph [4][2]. By the time of this release, LangGraph had already been "battle tested" in production by companies including Uber, LinkedIn, and Klarna [2].

## Release Timeline & Maturity

The road to 1.0 began with parallel alpha releases of LangChain and LangGraph 1.0 announced on September 2, 2025, targeting an official release candidate status with the full 1.0 expected in late October [1][2]. The official changelog confirms `v1.0.0` for both `langchain` and `langgraph` under October 20, 2025 [3].

A key framing point is that **LangGraph 1.0 is not a "new feature drop" release** [5]. Industry analysis describes the release as carrying "no breaking changes — all the hard-won lessons" from years of production use [5]. The core philosophy is captured in the official "What's new in LangGraph v1" page: "LangGraph v1 is a stability-focused release for the agent runtime. It keeps the core graph APIs and execution model unchanged, while refining type safety, docs, and developer ergonomics" [4].

## The Main Change: `create_react_agent` → `create_agent`

The single most important change in LangGraph 1.0 is the deprecation of the prebuilt `create_react_agent` in favor of LangChain's `create_agent` [4][6].

- The LangGraph prebuilt `create_react_agent` (from `langgraph.prebuilt`) is deprecated [4][6].
- It is replaced by `create_agent` from `langchain.agents`, which runs on LangGraph, provides a simpler interface, and offers significantly greater customization through the new **middleware** system [4][2].
- The same high-level interface had already been battle-tested in LangGraph for roughly a year before the 1.0 migration [2].
- Usage:
  - **Python:** `from langchain.agents import create_agent`
  - **JS:** `import { createAgent } from "langchain"` [2]
- A notable parameter change accompanies this: `prompt=` becomes `system_prompt=` [6].
- Tools now automatically validate their input when used with `create_agent`, which replaces the older `ValidationNode` pattern [6].

### Official Deprecation/Replacement Table from the Migration Guide

| Deprecated item (0.x) | Alternative in 1.0 |
|---|---|
| `create_react_agent` | `langchain.agents.create_agent` |
| `AgentState` | `langchain.agents.AgentState` |
| `AgentStatePydantic` | `langchain.agents.AgentState` (no more pydantic-only state) |
| `AgentStateWithStructuredResponse` | `langchain.agents.AgentState` |
| `AgentStateWithStructuredResponsePydantic` | `langchain.agents.AgentState` (no more pydantic-only state) |
| `HumanInterruptConfig` / `ActionRequest` | `langchain.agents.middleware.human_in_the_loop.InterruptOnConfig` |
| `HumanInterrupt` | `langchain.agents.middleware.human_in_the_loop.HITLRequest` |
| `ValidationNode` | Tools validate input automatically via `create_agent` |
| `MessageGraph` | `StateGraph` with a `messages` key (as `create_agent` provides) |

## Breaking Changes

The **only** explicit breaking change listed in the official migration guide is the drop of **Python 3.9 support** [6]:

- All LangChain/LangGraph packages now require **Python 3.10 or higher** [6].
- This is consistent with Python 3.9 reaching end-of-life in October 2025 [6].
- The recommended upgrade command is `pip install -U langgraph langchain-core` [6].

Beyond that, LangGraph 1.0 introduces no breaking changes to the core graph primitives (state, nodes, edges) or the underlying execution/streaming model [4].

## Stable Core Runtime Features (Stabilized, Not New)

These runtime capabilities remain first-class in LangGraph 1.0 and are presented as the stable foundation of the release [4][2]:

1. **Durable Execution** — checkpointing saves state at every node execution, so an agent can resume exactly where it left off after a server restart, including for agents running over hours or days [4].
2. **Streaming** — the runtime streams LLM tokens, tool calls, state updates, and node transitions [4].
3. **Human-in-the-Loop** — the runtime can pause execution, save state, and wait for human input without blocking threads, then resume from the exact pause point [4].
4. **Memory** — short-term memory is managed in state, while long-term memory is provided by persistent checkpointers that plug into databases [4][2].

## Introducing Middleware

Middleware is positioned as the defining new enabling feature that underpins the customization claims of 1.0 [7]. It allows developers to inject logic before and after the LLM call, around tool execution, or before and after an entire agent run. Prebuilt middleware covers use cases such as PII detection, summarization, and human-in-the-loop, and this is the mechanism behind the new `InterruptOnConfig`/`HITLRequest` APIs that replaced `HumanInterrupt` [7][6].

## Post-1.0 Release Policy

LangGraph introduced a formal release policy after reaching 1.0 [8]:

- **LangGraph 1.0 is an LTS line**, active until 2.0, after which it enters maintenance mode for at least one year [8].
- **LangGraph 0.4** (the last 0.x line) remains in maintenance mode until December 2026, receiving security patches and critical bug fixes only [8].
- Major releases are spaced 6–12 months apart; minor releases come every 1–2 months; patches are weekly [8].
- Breaking changes only occur in major versions; deprecated features remain functional for at least one minor version and are removed only in major versions [8].

The GitHub releases page confirms the post-1.0 trajectory with minor releases such as 1.1.0 (March 10, 2026) and 1.2.0 (May 12, 2026), and a current version around `1.2.11` [9][3]. These later minor releases added opt-in type-safe streaming/invoke (`version="v2"`), Pydantic/dataclass coercion, a beta `DeltaChannel` for reduced checkpoint overhead, per-node timeouts, node-level error handlers, and a new v3 streaming API — all backwards compatible [3].

## Conclusion

LangGraph 1.0 is best understood as the maturation of the framework rather than a headline feature release. It unifies the framework around LangChain 1.0, deprecates the older `create_react_agent`/prebuilt API in favor of the more customizable `create_agent` with middleware, and makes Python 3.10+ a hard requirement. The core graph runtime and execution model are deliberately unchanged, which keeps upgrades low-risk while the new middleware system opens the door to significantly richer agent orchestration.

### Sources

[1] LangChain & LangGraph 1.0 alpha releases (blog): https://www.langchain.com/blog/langchain-langchain-1-0-alpha-releases
[2] LangChain 1.0 alpha announcement (blog): https://www.langchain.com/blog/langchain-langchain-1-0-alpha-releases
[3] LangGraph changelog (v1.0.0 on Oct 20, 2025 + post-1.0 releases): https://docs.langchain.com/oss/python/releases/changelog
[4] What's new in LangGraph v1 (official docs): https://docs.langchain.com/oss/python/releases/langgraph-v1
[5] Medium analysis — "LangGraph 1.0 released; no breaking changes": https://medium.com/@romerorico.hugo/langgraph-1-0-released-no-breaking-changes-all-the-hard-won-lessons-8939d500ca7c
[6] LangGraph v1 migration guide (official, deprecations table): https://docs.langchain.com/oss/python/migrate/langgraph-v1
[7] Agent middleware deep dive (blog): https://blog.langchain.com/agent-middleware/
[8] LangChain / LangGraph release policy: https://docs.langchain.com/oss/python/release-policy
[9] GitHub releases — langchain-ai/langgraph: https://github.com/langchain-ai/langgraph/releases

