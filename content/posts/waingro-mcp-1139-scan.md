---
title: "The MCP Threat Model Has a Blind Spot. 1,139 Servers Prove It."
date: 2026-03-28
summary: "Independent security research on the Model Context Protocol ecosystem. Zero tool poisoning found in 1,139 servers, but 53% lack authentication, 16% access credentials beyond what they need, and 9% have path traversal patterns."
tags: ["waingro", "mcp", "security-research", "supply-chain"]
---

*Independent security research on the Model Context Protocol ecosystem, March 2026*

---

The Model Context Protocol has a security narrative problem.

Open any MCP security resource from the past six months (the OWASP MCP Top 10, Adversa's Top 25, the academic papers, the 30 CVEs in 60 days coverage) and you'll find the same fear: tool poisoning. Malicious instructions hidden in tool descriptions. Schema manipulation. Rug pulls where a tool changes its behavior after approval. The story is that MCP servers will trick your AI agent into doing something dangerous through clever metadata manipulation.

We built a scanner, pointed it at 1,139 MCP servers, and found none of that. What we found instead is worse, and far more boring.

## What we did

WAINGRO is an open-source MCP server security scanner with 16 detection rules covering 12 of the 14 major vulnerability classes identified by OWASP and Adversa AI. The rules operate across three layers:

- **Tool definitions**: scanning names, descriptions, and parameter schemas for injection patterns
- **Source code**: scanning handler implementations for credential access, path traversal, command injection, and exfiltration patterns
- **Package metadata**: scanning for supply chain indicators like lifecycle hooks and dependency risks

We built a discovery pipeline that enumerated 1,595 unique MCP servers from awesome-mcp-servers, the npm registry, and GitHub topic searches. Of those, 1,304 were successfully cloned and 1,139 passed parsing and scanning.

Every detection rule maps to an industry-standard vulnerability class, either OWASP MCP Top 10 or Adversa's MCP Security Top 25. We tuned the rules against a first tranche of 50 servers, reducing false positives by 48% before running the full batch.

## The headline: zero confirmed tool poisoning

Our MCP-001 (tool description injection) and MCP-002 (parameter schema injection) rules flagged 7 servers across the entire corpus. Manual verification of all 7:

- **Two** were security tools with detection rule descriptions containing the pattern "ignore previous instructions," because that's what their scanner looks for
- **One** was a deliberate test fixture for validating a proxy server's tool poisoning defense
- **One** was defensive prompt engineering, an application using prompt injection *against its own LLM context* to prevent contamination
- **One** was a benign parameter description ("Pass chat history")

Zero confirmed tool poisoning in 1,139 servers. The attack vector that dominates the MCP security conversation simply isn't present at ecosystem scale, at least not yet.

This is actually the more interesting finding. The MCP ecosystem has no systemic defense against tool poisoning (most clients don't validate tool descriptions at all), yet no one is exploiting it. The locks are off, but no one's walking through the door.

## What we actually found

While tool poisoning was absent, implementation-level vulnerabilities were everywhere.

**53% of servers lack authentication.** Over 6,000 MCP-011 hits across 1,139 servers. The MCP specification states that authentication is "left to implementors." Most implementors left it entirely. HTTP and SSE transport servers are exposed without any auth mechanism: no tokens, no OAuth, no API keys.

**16% access credentials.** 1,810 MCP-005 hits. Servers reading environment variables for API keys, accessing credential files, harvesting cloud provider tokens. Some of this is legitimate (a server needs credentials to call an API). Much of it is overly broad: servers that read every available environment variable rather than only what they need.

**9% have path traversal patterns.** 1,015 MCP-012 hits. User-influenced file paths reaching filesystem operations without containment checks. This is consistent with prior research that found 82% of surveyed MCP implementations use file operations vulnerable to path traversal. Our scanner confirms the pattern persists at scale.

**Transport exfiltration is real.** 643 MCP-008 hits. Most are legitimate TCP socket usage for desktop application bridges (Unreal Engine, Blender, etc.), but a subset of servers create tunnels (ngrok and similar) from MCP tool invocations, exposing localhost services to the public internet without explicit user awareness.

**Resource content poisoning surface is endemic.** 6,739 MCP-015 hits. Servers serving resource content from external sources without sanitization. This is the largest category by raw count and represents a systemic surface for indirect prompt injection, where the poisoned content comes not from the tool definition but from the data the tool retrieves.

## The two findings worth disclosing

Beyond the systemic patterns, two individual servers exhibited behavior that warranted responsible disclosure.

**Runtime MCP client config manipulation.** One server, a security gateway product, reads and writes to its hosting client's global configuration file (`~/.cursor/mcp.json` or `claude_desktop_config.json`) at runtime. It modifies the blocked/unblocked status of other MCP servers as part of its normal scanning operation. A compromised version of this security tool could silently unblock malicious servers or block legitimate ones. The irony: a security product introducing the exact vulnerability class it's designed to prevent.

**Installation-time config injection.** A second server directly writes to the user's global MCP client config during its installation flow, injecting its own server entry. While this is part of a documented install command, it writes programmatically to the global config without explicit confirmation. If the package's supply chain is compromised, the attacker gets injected into every user's MCP client config automatically.

Both findings follow the Adversa #7 pattern (MCP Configuration Poisoning) and have been disclosed to the respective maintainers.

## What this means

The MCP security community has been preparing for the wrong fight.

Tool poisoning (hidden instructions in descriptions, schema manipulation, rug pulls) is the sophisticated attack that dominates the threat models and the conference talks. It's technically feasible, well-documented, and zero instances were found in 1,139 production MCP servers.

Meanwhile, over half the ecosystem ships without authentication. One in six servers harvests credentials beyond what it needs. Nearly one in ten has path traversal. These aren't exotic attacks. They're the same boring implementation bugs that have plagued web applications for twenty years: missing auth, unsanitized paths, overly broad credential access.

The MCP specification says security is "left to implementors." The data says implementors aren't implementing it.

## Recommendations

**For the MCP specification authors:** "SHOULD" isn't working. Authentication, input validation, and credential scoping need to move from recommended to required. The spec's security section is well-written but clearly aspirational given the ecosystem data.

**For MCP server developers:** Before adding tool features, add authentication. Restrict file system access to declared paths. Access only the credentials your server actually needs. These aren't novel recommendations. They're the same advice the web security community has been giving for two decades, and they apply unchanged to MCP servers.

**For MCP client developers:** The ecosystem can't be trusted to self-secure. Clients need to validate tool definitions, sandbox tool execution, restrict file access, and filter credentials, regardless of whether the spec requires it. Our companion research (forthcoming) evaluates whether current MCP clients actually do this.

**For organizations deploying MCP:** Audit your servers. Run a scanner. The attack surface isn't the theoretical tool poisoning scenario from the threat model. It's the missing auth on the server you deployed last week.

## Methodology and data

WAINGRO and the complete detection rule set are open source. The scan dataset, covering 1,139 servers, 16 rules, and the full triage of all MALICIOUS verdicts, is available for independent verification.

Scanner: [github.com/north-echo/waingro](https://github.com/north-echo/waingro)

---

*Christopher Lusk is an independent security researcher focused on AI agent framework security. This research was conducted as part of the FULCRUM agent security research platform. Contact: clusk@northecho.dev / GitHub: north-echo*
