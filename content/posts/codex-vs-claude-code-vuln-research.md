---
title: "Why I Trust Codex More Than Claude Code for Vulnerability Research"
date: 2026-04-25
summary: "A practitioner's field report comparing Codex CLI and Claude Code for vulnerability research. Claude Code excels at architecture and ideation, but Codex delivers more disciplined, trustworthy findings during active testing — with fewer false positives and less researcher time wasted on invalidation."
tags: ["security-research", "ai-agents", "claude-code", "codex", "vulnerability-research"]
---

*Christopher Lusk · North Echo Security Research · April 2026*

---

I've spent the last several months building offensive security tooling almost entirely through AI coding agents. Fluxgate, ICS-POT, REAPER, CHERITTO: every one of these projects was scaffolded, iterated, and shipped with an AI agent in the terminal. For most of that time, Claude Code was my only tool. It's brilliant at architecture, it understands complex codebases, and it's genuinely fun to work with. Conversational, collaborative, and fast on its feet.

But a few weeks ago, I started using Codex CLI alongside Claude Code. And when it comes to the part of my work that matters most, actually finding vulnerabilities and deciding whether they're real, I trust Codex more. Not because it's smarter. Because it's more disciplined.

This post is a practitioner's comparison from someone who uses both tools daily, across real vulnerability research projects with real disclosure outcomes. It's not a benchmark. It's a field report.

## The Experienced Engineer vs. the Gifted Intern

The best analogy I've found: Codex is like working with an experienced engineer who has a dry personality. Claude Code is like working with a gifted intern who is overly optimistic and needs supervision.

Claude Code is more collaborative. It's conversational, it riffs on ideas, it gets excited about findings. That energy is genuinely useful during the early creative phases of research. Brainstorming attack surfaces, designing campaign methodology, thinking through how components interact. But that same enthusiasm becomes a liability during the phase that actually matters: validating whether something is a real vulnerability.

Codex is more methodical, more conservative, and more practical by default. It doesn't get excited. It checks its work. When Codex tells me it found something, I have significantly more inherent trust in that finding than when Claude Code reports the same. That trust differential isn't based on a vague feeling. It's based on weeks of running both tools against the same class of problems and seeing which one wastes less of my time chasing phantoms.

## Claude Code's Check-the-Box Problem

Here's the pattern I've seen repeatedly across CHERITTO, REAPER, and my SSSD research: Claude Code defaults to a shallow, check-the-box, low-hanging-fruit approach when dynamically testing for flaws. It reaches for the obvious thing, reports it with confidence, and moves on.

Even when I provide foundational guidance in CLAUDE.md (be thorough, be methodical, validate findings against source code and documentation before reporting them, cross-reference designed behavior) I still find myself explicitly redirecting Claude Code during testing sessions. "Go deeper." "Don't just check the obvious path." "Actually trace this through the implementation." "Are you sure this isn't designed behavior?" These aren't occasional nudges. They're a constant part of the workflow.

The effort problem compounds: Claude Code doesn't just skim the surface on its first pass. It resists going deeper unless you push it. I've had sessions where I had to ask three or four times for a more thorough analysis before Claude Code actually engaged with the complexity of the target. That's not a tool accelerating my research. That's a tool I'm dragging through the research while it tries to declare victory early.

Codex doesn't do this. It doesn't need to be told to be thorough. Its default posture is methodical, and when it reports a finding, the analysis behind it is usually proportional to the complexity of the issue. I rarely have to tell Codex to double-check its work.

## The Overstated Findings Problem

This is the issue that costs real time and damages trust: Claude Code routinely overstates the severity and validity of its findings.

During CHERITTO-OCP, my adversarial security campaign against OpenShift 4.21, Claude Code would surface findings that looked significant on first read. Operator RBAC misconfigurations, trust boundary violations, potential privilege escalation paths. Each one described with confidence and specificity.

Then I'd tell Claude Code to verify the finding against the actual documentation, the architecture, and the designed behavior of the component. And the finding would collapse. What looked like a trust boundary violation was actually the intended behavior of the operator reconciliation loop. What looked like a privilege escalation path was gated by an admission controller that Claude Code hadn't checked. What looked like an RBAC misconfiguration was consistent with the documented security model.

After cross-referencing, the majority of Claude Code's initial findings downgraded to non-disclosable. Not edge cases. The majority. The entire CHERITTO-OCP campaign completed with no disclosure-worthy findings, and a significant portion of the research time was spent invalidating findings that Claude Code reported with unwarranted confidence.

This is the fundamental trust problem. When your tool overstates findings by default, you can't use its output as a reliable signal. Every finding requires the same level of manual validation you'd do without the tool, which raises the question of what the tool is actually contributing to the validation phase.

Codex overstates less. It's more conservative in what it reports, more likely to flag uncertainty, and more likely to check its own work before presenting a finding. When Codex surfaces something, I still validate it (I'm not outsourcing judgment to an AI) but the hit rate is meaningfully higher. More of what Codex reports survives the cross-referencing step.

## The Semgrep Data Confirms This

Semgrep published a comparison of Claude Code and Codex for vulnerability discovery across 11 open-source Python applications in 2025. Claude Code found more total findings (46 vs. 21) but with a lower true positive rate (14% vs. 18%). Both had high false positive rates, but Claude Code's was higher: 86% vs. 82%.

That data maps directly to my experience. Claude Code casts a wider net and catches more things. But it also catches more things that aren't real. Codex reports fewer findings, but a higher proportion of them are actual vulnerabilities. In vulnerability research, precision matters more than recall. A tool that gives you 20 real findings out of 100 reported is more useful than a tool that gives you 14 real findings out of 100 reported, because the false positives each consume researcher time to invalidate.

For Claude Code, the numbers get worse in practice because the false positives aren't just wrong. They're confidently wrong. They come with detailed explanations that sound credible, which means the invalidation process takes longer than it would for a finding that was obviously weak.

## Where Claude Code Still Wins

I still use Claude Code daily. The criticism above is specific to one phase of security research, dynamic testing and finding validation. In other phases, Claude Code is the stronger tool.

**Architectural reasoning.** When I'm designing a new project, laying out the module structure, defining data flows, choosing patterns, Claude Code with Opus is in a different league. The agent-skill architecture behind BREEDAN, the six-parser protocol ingestion daemon for ICS-POT, the four-controller Kubernetes operator design for Fluxgate Operator: all of these were designed in Claude Code sessions where the model demonstrated genuine understanding of how components should interact at scale. Codex is good at implementing. Claude Code is better at thinking about what to implement and why.

**Collaboration and ideation.** Claude Code's conversational nature is a genuine strength during the early phases of research. When I'm mapping an attack surface, brainstorming abuse-by-design scenarios for CHERITTO methodology, or thinking through how trust boundaries should be tested, Claude Code is a better thinking partner. It engages, it pushes back, it suggests angles I hadn't considered. Codex gives you answers. Claude Code gives you a conversation.

**Documentation.** Every North Echo project has a detailed spec written in Markdown. Claude Code produces better technical documentation. It captures nuance, maintains consistency across long documents, and structures information in a way that's useful for both human readers and downstream agent consumption. My CHERITTO-OCP methodology doc, the Fluxgate Operator spec, the BREEDAN architecture pattern: all Claude Code output.

**Complex codebase navigation.** Claude Code's ability to hold a large codebase in context and reason across file boundaries is stronger. When I was doing SSSD vulnerability research, tracing integer underflow conditions through the sss_client protocol implementation, Claude Code could follow the data flow across multiple translation units and identify where a controlled value crossed a trust boundary. That kind of deep, cross-file taint analysis is where Claude's context handling shines.

## The Classifier: A Footnote, Not the Story

I want to mention the classifier issue briefly because it's relevant context, even though it's not the reason I prefer Codex for active research.

Claude Code has a content classifier that can block security-related requests. I went through the process of filing a Cyber Use Case form and received an account-level safeguards adjustment. Since then, the classifier hasn't been a problem for me. Anthropic deserves credit for offering a path that works.

But it's worth noting that many researchers in the community are still hitting these blocks, especially since the Opus 4.7 rollout. The frustration is real and widespread. If you're doing security research with Claude Code and haven't filed for the exemption, do it. It makes a meaningful difference.

The classifier isn't why I reach for Codex during active testing. I reach for Codex because it does the testing better.

## How I Actually Use Both

My workflow has settled into a pattern that plays to each tool's strengths:

**Claude Code** handles project design, specification writing, architecture decisions, and the creative/ideation phases of research campaigns. It's where I think through what I'm going to test and why, design the methodology, and structure the project.

**Codex** handles the actual testing, finding validation, and implementation of security tooling. It's where I do the work that produces disclosure-worthy findings. When Codex reports something, I trust it enough to invest time in a full write-up and verification. When Claude Code reports something, I trust it enough to investigate further, but I expect most findings to dissolve under scrutiny.

**Both tools together** on any serious project. I scaffold architecture in Claude Code, implement in Codex, validate findings with Codex, and write up results in Claude Code. The tools are complementary if you understand where each one's judgment should be trusted.

## What This Means for Security Researchers

If you're choosing between these tools for vulnerability research, here's what I'd say after several months of daily Claude Code use and a few weeks of running Codex alongside it:

Don't pick one. Use both. But understand that Claude Code will try to impress you with findings that don't survive validation, and you need to build that expectation into your workflow. Codex won't impress you. It'll just quietly give you better signal.

The tool that finds fewer things but is right more often is the tool you want validating your findings. The tool that's more fun to talk to is the tool you want designing your approach. They're different skills, and right now, they live in different products.

---

*Christopher Lusk is a Principal Product Security Engineer and independent security researcher operating North Echo Security Research. His work spans ICS/OT security, CI/CD supply chain vulnerabilities, container security, and vulnerability management.*

*Tools and projects referenced in this post are available at [github.com/north-echo](https://github.com/north-echo).*
