---
title: "What I Learned Auditing 9 AI Agent Frameworks"
date: 2026-04-05
summary: "I spent a month auditing 9 AI agent frameworks using consistency analysis. 18 advisories filed across familiar bug classes — SSRF, auth gaps, path traversal — all rooted in the same pattern: abstractions constrain, but engines accept."
tags: ["fulcrum", "security-research", "ai-agents", "disclosure"]
---

*Independent security research on agentic AI frameworks, April 2025*

---

I spent the last month auditing AI agent frameworks for security vulnerabilities. I'm relatively new to security research — my background is in vulnerability management, containers and Linux to be specific — and this project started as a way to learn by doing. I chose agent frameworks because they sit at an interesting intersection: they handle credentials, execute code, make HTTP requests, and manage sensitive data, but they're new enough that the security research community hasn't given them much attention yet.

What I found surprised me. Not because the bugs were exotic, but because the same pattern kept showing up across completely different frameworks, languages, and architectures. This post documents what I learned — the methodology, the patterns, and the takeaways — in case it's useful to other researchers or to the framework developers working on these projects.

## The Approach

I developed a testing methodology I call FULCRUM, built around **consistency auditing**: for every security-relevant check in a codebase, I verify it exists at every entry point that handles the same data. This turned out to be more productive than fuzzing or dynamic testing alone — most of the bugs I found weren't in parsing or memory safety but in missing checks at parallel code paths.

The workflow for each target:

1. **Recon** — Read the source, map API routes, understand the architecture. Pay special attention to the boundary between the user-facing abstraction (UI, CLI, DSL) and the execution engine underneath.
2. **Consistency audit** — For each security check, ask: does it exist at the API layer? The engine layer? Both? Neither?
3. **Dynamic testing** — Craft inputs that bypass the abstraction layer and go directly to the API or engine. See what gets accepted.
4. **Validation** — Reproduce every finding at least 3 times. Build PoCs that the maintainer can run.
5. **Disclosure** — File individually through GitHub Security Advisories or vendor-specific channels. Wait for response before publishing details.

## The Pattern

The finding that repeated across multiple frameworks was what I started calling the "abstraction bypass" — a classic problem in a new context.

Agent frameworks use visual editors, graph builders, YAML DSLs, and other abstractions to let users define agent behaviors. These abstractions impose constraints: the workflow editor only shows certain node types, the form validates URLs, the graph builder prevents certain connections.

But the execution engine underneath doesn't always enforce those same constraints. When you bypass the abstraction — by calling the API directly, by importing a crafted DSL, or through prompt injection that influences code generation — the constraints disappear.

This is the "client-side validation" problem that web developers learned about 20 years ago, but it has a new wrinkle in agent frameworks: the LLM itself can generate inputs that bypass the abstraction. A prompt injection that influences a tool builder, a poisoned document that changes agent behavior, or a crafted API call that the UI would never produce — these all exploit the gap between what the abstraction allows and what the engine accepts.

I found this pattern in frameworks I tested across different languages, architectures, and deployment models. I'm withholding specific technical details for findings that are still under responsible disclosure, but the general shape was consistent: the UI constrains, the API accepts, the engine executes.

## What I Found (Summary)

Across 9 frameworks, I filed 18 advisories. Several have already been accepted and patched by maintainers — I'm grateful for the responsive teams at OpenClaw and others who took the reports seriously and shipped fixes quickly.

The bug classes broke down into familiar categories:

- **SSRF** — Agent execution environments reaching internal infrastructure through proxy misconfigurations or missing URL validation
- **Missing authentication** — Test or internal endpoints exposed on public API namespaces
- **Authorization gaps** — Blocklists that covered some credential paths but missed others in the same class
- **Path traversal** — File operations that validated paths in one code path but not a parallel one
- **Unsafe deserialization and weak cryptography** — The usual suspects

Two frameworks came back clean: CrewAI and IronClaw (NEAR AI's Rust agent framework). The clean results are as informative as the findings — IronClaw in particular is interesting because it reimplements the same architecture as OpenClaw (where I found multiple bugs) in Rust, and the Rust implementation avoids the entire class of issues I found in the TypeScript original. Language and type system choices have real security consequences.

For IronClaw, instead of filing an advisory, I opened a feature request proposing a security improvement to their WASM tool builder — a more constructive outcome than a disclosure.

## The Experiment: RAG Poisoning

Separately from the framework audits, I ran an experiment on LlamaIndex to test whether a poisoned document in a RAG index could influence an agent's tool selection. Prior research by Prompt Security (2025) had demonstrated RAG poisoning in LangChain, but nobody had published empirical data on LlamaIndex specifically.

The setup was simple: a ReAct agent with multiple tools, a vector index of benign documents plus one document containing embedded tool-selection instructions, and a local LLM.

The key finding was a meaningful difference between two retrieval approaches:

- **RetrieverTool** (passes raw document text into the agent's reasoning chain): 50-83% tool misdirection rates with explicit injection templates
- **QueryEngineTool** (summarizes retrieved content before the agent sees it): matched the baseline — no measurable effect from poisoning

The summarization step in QueryEngineTool effectively dilutes injected instructions before they reach the agent. This is an unintentional defense — the security implication of choosing one retrieval approach over the other isn't documented. If you're building RAG-powered agents with LlamaIndex, this is worth considering.

I didn't file this as a vulnerability — LlamaIndex's security policy explicitly scopes prompt injection out, and I think that's a reasonable position. But the empirical data is useful for developers making architectural decisions about their RAG pipelines.

## By the Numbers

| Metric | |
|---|---|
| Frameworks audited | 9 |
| Advisories filed | 18 |
| Accepted and patched | 6 |
| Closed (by design / disputed) | 2 |
| Under disclosure timeline | 7 |
| Honest negatives | 2 |
| Unique CWE classes | 8 |
| MCP servers scanned | 1,139 |
| PoC test assertions | 120+ |

The most common bug class was SSRF — agent execution environments reaching internal infrastructure through misconfigured proxies or missing URL validation. Authorization gaps (blocklists that covered some credential paths but missed parallel ones) were second. Path traversal, missing authentication, unsafe deserialization, and race conditions rounded out the rest.

The two honest negatives (CrewAI and IronClaw) account for about 22% of the targets — a useful reminder that not every framework has these issues, and that architectural choices like language, type system, and sandbox model meaningfully affect the outcome.

## What I Learned

**Consistency auditing is underrated.** The most productive technique in my toolkit wasn't fuzzing or automated scanning — it was reading code and asking "does this check exist everywhere it should?" Most of the bugs I found were in the *absence* of checks at parallel code paths, not in the *presence* of flawed logic.

**Security fix batches are research opportunities.** When a project publishes a batch of security fixes, each fix reveals a bug class the maintainers identified. Going back after a fix batch and checking for variant instances of the same classes is high-yield work. Two of my OpenClaw findings came from variant analysis of their own security hardening commits.

**Honest negatives matter.** Two of my nine targets came back clean. Those results are just as important as the findings — they tell us what architectural choices make a framework more resistant to these bug classes. If I only published the findings and not the negatives, the picture would be misleading.

**Maintainer relationships matter.** The OpenClaw team accepted and patched three findings within hours. That kind of responsiveness makes the entire disclosure process worthwhile. Other teams had different perspectives — LangGraph closed my finding as "by design," which I initially disagreed with but came to understand their reasoning. Not every security gap is a vulnerability from the maintainer's perspective, and that's a legitimate position when the behavior is documented.

**I'm still learning.** I made mistakes during this project — PoCs that needed revision, findings that turned out to be documented behavior, scoping decisions I'd make differently in hindsight. The security research community has been generous with feedback, and I've tried to incorporate lessons from each engagement into the next one.

## Tools

Two tools I built for this research are available:

- **WAINGRO** ([github.com/north-echo/waingro](https://github.com/north-echo/waingro)) — A static scanner with 46 detection rules for agent frameworks and MCP servers. I used it for a scan of 1,139 MCP servers in the ecosystem.
- **FULCRUM MCP Test Harness** — An adversarial MCP server for testing how clients handle malicious tool definitions. Used for testing Claude Code, Cline, and Cursor.

## Responsible Disclosure

All findings were reported through coordinated responsible disclosure — GitHub Security Advisories where available, vendor-specific channels otherwise. Some findings are still under disclosure timelines and are not detailed in this post. I'll update with additional technical details as embargoes expire.

I'm grateful to the maintainer teams who engaged constructively with these reports. Building secure agent frameworks is hard, and the people working on these projects are doing important work under real constraints.

---

*Christopher Lusk is a security researcher at North Echo Security Research, focused on AI agent frameworks and MCP server security. This work was conducted independently under the FULCRUM research platform.*

*Contact: clusk@northecho.dev*
*Tools: [WAINGRO](https://github.com/north-echo/waingro)*
