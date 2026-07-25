---
title: "What the Linux kernel actually requires of AI-assisted patches"
date: 2026-07-24
summary: "Four documents in Documentation/process/ set the rules for LLM-assisted kernel contributions: a mandatory Assisted-by trailer with a specific format, a hard prohibition on AI signing off, changelog disclosure most people would not guess at, and a rule that AI-found security bugs are already public. Every quotation checked against source, with the commits that put each rule in the tree."
tags: ["linux-kernel", "ai-agents", "disclosure", "policy", "security-research"]
---

*Christopher Lusk, North Echo Security Research*

If you are sending kernel patches with help from an LLM, there are two documents in `Documentation/process/` that you are probably not following, because until recently they did not exist and almost nobody talks about them.

They are [`coding-assistants.rst`][ca], added by Sasha Levin on 2026-01-06 in commit [`78d979db6cef`][ca-commit] ("docs: add AI Coding Assistants documentation"), and [`generated-content.rst`][gc], added by Dave Hansen two weeks later on 2026-01-20 in commit [`a66437c27979`][gc-commit] ("Documentation: Provide guidelines for tool-generated content"). Between them, plus a short section in [`submitting-patches.rst`][sp], they set out what the kernel expects: a mandatory attribution trailer with a specific format, a hard prohibition on AI signing off, and a disclosure requirement in the changelog that most people would not guess at.

Both files are small, 59 and 109 lines respectively, and as of 2026-07-24 neither has been touched since the commit that created it. That is worth knowing in both directions. They are stable enough to quote, and they are new enough that a lot of people submitting AI-assisted patches today started before the rules existed and have never gone back to check.

Every quotation below was checked verbatim against the source files on 2026-07-24. Links to each document, and to the four commits that put these rules in the tree, are collected at the end.

I found all of this the way you would expect: by auditing my own patch against the process docs before sending it, and discovering that two of the binding documents were ones I had never read. This post is the summary I wish I had had first.

One caveat on freshness. The two AI documents have not changed since they landed, but the other two move constantly. [`security-bugs.rst`][sb] took **eight** commits in 2026 alone, most recently on 2026-05-13, and nearly all of them were Willy Tarreau reworking how the security team wants reports written. If you are reading this some months after publication, check the current text before relying on any quotation here.

## 1. The `Assisted-by:` trailer is mandatory

[`submitting-patches.rst`][sp] is direct about it. The "Using Assisted-by:" section is newer than either AI document, added by Jonathan Corbet in commit [`6252e5c1c20e`][sp-commit] and landing 2026-04-07:

> If you used any sort of advanced coding tool in the creation of your patch, you need to acknowledge that use by adding an Assisted-by tag. Failure to do so may impede the acceptance of your work.

Note "may impede the acceptance of your work." That is not a style note. Omitting it is grounds for a maintainer to bounce you.

The format is specified in [`coding-assistants.rst`][ca]:

```
Assisted-by: AGENT_NAME:MODEL_VERSION [TOOL1] [TOOL2]
```

with the documented example:

```
Assisted-by: Claude:claude-3-opus coccinelle sparse
```

`AGENT_NAME` is the tool or framework, `MODEL_VERSION` the specific model. The optional trailing entries are *specialized analysis tools*, meaning things like coccinelle, sparse, smatch or clang-tidy. The doc says explicitly that basic development tools, git, gcc, make and your editor, should not be listed.

Two practical points the doc does not spell out. First, use one trailer per agent if more than one was involved; there is no syntax for combining them. Second, the model version should be the model that actually did the work on *this* patch. It is easy to copy the trailer forward from your last submission and quietly misattribute the work to a model you did not use. I did exactly that, and caught it only because I was auditing the patch line by line for something else.

## 2. AI must not sign off. Ever.

[`coding-assistants.rst`][ca], under "Signed-off-by and Developer Certificate of Origin":

> AI agents MUST NOT add Signed-off-by tags. Only humans can legally certify the Developer Certificate of Origin (DCO).

The doc then lists what the human submitter is responsible for: reviewing all AI-generated code, ensuring licensing compliance, adding their own `Signed-off-by`, and taking full responsibility for the contribution.

This is the one rule with actual legal weight behind it rather than process preference. The DCO is an assertion about provenance and rights that a model cannot make. If your tooling is configured to add sign-offs automatically, turn that off before it embarrasses you.

The practical reading: your patch should carry exactly one `Signed-off-by`, yours, plus however many `Assisted-by` trailers the work warrants.

## 3. The changelog has to disclose more than you think

This is the requirement people miss, and it lives in [`generated-content.rst`][gc]. The document applies whenever "a meaningful amount of content in a kernel contribution was not written by a person in the Signed-off-by chain."

Trivial tool use is out of scope, and the doc is sensible about it: spelling fixes, identifier completion, mechanical renames, `clang-format`. In scope is anything substantive, and the examples given include a chatbot writing a function, a file generated by an assistant and cleaned up by hand, and, notably, "the changelog was generated by handing the patch to a generative AI tool and asking it to write the changelog."

What it asks you to put in the cover letter or changelog:

- What tools were used.
- The input to those tools. For a Coccinelle script, the script. For a model, "if code was largely generated from a single or short set of prompts, include those prompts. For longer sessions, include a summary of the prompts and the nature of resulting assistance."
- Which portions of the content the tool affected.
- **How the submission was tested, and with what.**

There is also a sentence that is easy to skim past and that I think is the most interesting requirement in either document:

> Detection of a problem and testing the fix for it is also part of the development process; if a tool was used to find a problem addressed by a change, that should be noted in the changelog.

So if a static analyser, a fuzzer, or a model found the bug, say so in the changelog. Not in the cover letter, not in a follow-up: the changelog, which is the part that becomes permanent git history.

That cuts against the instinct to present a finding as though you reasoned your way to it unaided. It is also, if you are doing this seriously, straightforwardly good for you. Naming the analysis that found the bug is credit for the analysis.

## 4. A patch with no testing statement is a defect

Be careful about where this requirement actually comes from. [`submit-checklist.rst`][sc] has long told you to *test*, at length and in detail: `CONFIG_PREEMPT`, lockdep, fault injection, linux-next, and so on. What it does not tell you to do is *say what you did*. The obligation to state your testing in the submission is new, and it comes from [`generated-content.rst`][gc], which asks flatly: "How is the submission tested and what tools were used to test the fix?"

That distinction matters because it is the thing I most nearly got wrong.

If you do not say how you tested, the maintainer will assume you tested. Silence is not neutral. "Compile-tested only, no hardware available" is a perfectly respectable statement and takes one line. Shipping without it, on a patch to a driver you cannot run, is letting a reviewer believe something untrue by omission.

Say which architecture, which config, which compiler, whether you ran it, and what you could not verify. If you have no hardware, say which behaviours are therefore unverified, specifically. A reviewer who knows exactly what you did not test can decide whether to test it themselves.

## 5. Maintainers may treat your patch differently, and that is written down

[`generated-content.rst`][gc] is explicit that maintainers retain full discretion, and it enumerates the options: treat it like any other contribution, reject it outright, ask for extra testing, review it with extra scrutiny, **review it at lower priority than human-generated content**, ask you to explain how the model was trained, or ask you to demonstrate that you understand your own submission.

It closes with three separate paragraphs, which I quote separately because they escalate:

> If tools permit you to generate a contribution automatically, expect additional scrutiny in proportion to how much of it was generated.

> As with the output of any tooling, the result may be incorrect or inappropriate. You are expected to understand and to be able to defend everything you submit. If you are unable to do so, then do not submit the resulting changes.

> If you do so anyway, maintainers are entitled to reject your series without detailed review.

Being transparent about tool use is not a shield. It buys you a fair hearing, not a fast one. Plan for your patch to sit longer than an equivalent hand-written one, and do not read that as hostility.

## 6. If AI helped you find a security bug, it is already public

[`security-bugs.rst`][sb] is blunter than anything in the other two documents. The rule below arrived in commit [`a03ef333fbd6`][sb-public], Willy Tarreau, landing 2026-05-12:

> **If you resorted to AI assistance to identify a bug, you must treat it as public**. While you may have valid reasons to believe it is not, the security team's experience shows that bugs discovered this way systematically surface simultaneously across multiple researchers, often on the same day.

Read the reasoning, not just the rule. This is not a statement about AI-found bugs being lower quality. It is a statement about *simultaneity*: the security team has observed that when these bugs surface, they surface to several people at once. Your embargo is theatre if three other people are looking at the same function this week.

That matches my own experience uncomfortably well. Across four research cycles I have been independently beaten to the same finding three times, twice by days rather than months. The document is describing something real.

The consequence for process is that the exception in "Identifying contacts" explicitly covers you: do not send it privately unless you have reason to think it is genuinely not public, where the listed exceptions include the "result of a widely available automated vulnerability scanning tool that can be repeated by anyone, or use of AI-based tools."

The same section adds a caveat that is easy to get backwards, and I did get it backwards in an earlier draft of this post. The reproducer rule is not "publish the reproducer." It is the opposite:

> In this case, do not publicly share a reproducer, as this could cause unintended harm; just mention that one is available and maintainers might ask for it privately if they need it.

So: report publicly, but hold the reproducer and say you have one.

The companion section, "Responsible use of AI to find bugs," added by the same author the same day in commit [`4bf85afb9f3e`][sb-commit], lists five things that get reports ignored. Paraphrasing, with the doc's own emphasis preserved:

- **Length.** AI reports run long. Put a clear summary and the critical details first. "Configure your tools to produce concise, human-style reports."
- **Formatting.** "always convert your report to plain text." Markdown decorations do not survive quoting and forwarding.
- **Impact evaluation.** Read [`threat-model.rst`][tm] and stick to verifiable facts rather than inventing theoretical consequences. The doc suggests having your tool read that documentation as part of the evaluation.
- **Reproducer.** Produce one and "test it thoroughly." If your tool cannot produce a working one, "the validity of the report should be seriously questioned." Share it only on request, per above.
- **Propose a fix.** Ask your tool for one and "test it" before reporting. If the fix cannot be tested because it needs rare hardware or an almost extinct protocol, "the issue is likely not a security bug." Any proposed fix must follow [`submitting-patches.rst`][sp] and carry a `Fixes:` tag.

That last one has teeth beyond formatting. It is effectively saying that untestability is evidence against the finding being a security bug at all.

## A worked example

The patch I audited all of this against was a small etnaviv fix, two files, honoring a read-only flag that the GPU MMU mapping path ignored. What the rules produced, concretely:

- Two `Assisted-by:` trailers, one per agent, with the model versions that did this specific work.
- One `Signed-off-by`, mine, last in the block.
- A `Suggested-by:` for the maintainer whose review comment on a parallel patch supplied the approach, which the docs permit without asking first when the suggestion was made publicly.
- A changelog paragraph naming the static-analysis pass that found the bug, stating which parts were drafted with model assistance, and recording that a specific technical claim in an earlier draft was wrong and was corrected by reading the source directly.
- A testing line stating compile-tested only, with architecture, config and compiler, and an explicit list of what remained unverified for lack of hardware.

None of that is onerous. It took perhaps forty minutes, most of it spent reading the two documents for the first time.

## One trap that is not in any document

When you add attribution trailers, `git send-email` starts deriving recipients from them.

A `Suggested-by:` line adds that person to the Cc. So does a `Cc: stable@vger.kernel.org` line in the commit message. This means the customary rehearsal send, the one you address to yourself to check that your mailer is not mangling whitespace, quietly resolves to more than one recipient. Mine expanded to three: me, the maintainer, and a public list, for a patch I had not finished checking.

`sendemail.suppresscc=self` does not help, since it suppresses only you. Use `--suppress-cc=all` on the rehearsal, and dry-run it first, and confirm the log shows exactly one `RCPT TO:` before you send for real.

The irony is worth stating: the better your attribution hygiene, the more addresses your practice round leaks to. Doing the credit properly is what creates the hazard.

## Summary

- `Assisted-by:` is mandatory, format-specified, one per agent, and should name the model that did *this* work.
- AI must never add `Signed-off-by`. Only you can certify the DCO.
- Disclose in the changelog: what tools, what prompts in summary, which parts they touched, what found the bug, and how you tested.
- State your testing explicitly, including what you could not test.
- Expect slower review, and do not mistake it for hostility.
- Security findings from AI tooling go to the public list.
- Use `--suppress-cc=all` on your practice send.

Read the two documents. They are short, they are clearer than this summary, and they are the actual authority.

## Sources

All quotations in this post were checked verbatim against the source files on 2026-07-24. Where I paraphrase rather than quote, I have tried to make that obvious.

**The documents**

- [`Documentation/process/coding-assistants.rst`][ca] : the `Assisted-by:` format and the DCO prohibition.
- [`Documentation/process/generated-content.rst`][gc] : changelog disclosure, testing statement, maintainer discretion.
- [`Documentation/process/submitting-patches.rst`][sp] : the "Using Assisted-by:" section making the trailer mandatory, and the rules on `Fixes:`, `Suggested-by:` and tagging people.
- [`Documentation/process/security-bugs.rst`][sb] : treat AI-found bugs as public, and the "Responsible use of AI to find bugs" section.
- [`Documentation/process/submit-checklist.rst`][sc] : what to test, as distinct from what to say about testing.
- [`Documentation/process/threat-model.rst`][tm] : referenced by the impact-evaluation rule above.

**The commits that created the AI documents**

- [`78d979db6cef`][ca-commit] "docs: add AI Coding Assistants documentation", Sasha Levin, landed 2026-01-06, +59 lines.
- [`a66437c27979`][gc-commit] "Documentation: Provide guidelines for tool-generated content", Dave Hansen, landed 2026-01-20, +109 lines.
- [`6252e5c1c20e`][sp-commit] "docs: add an Assisted-by mention to submitting-patches.rst", Jonathan Corbet, landed 2026-04-07. This is what makes the trailer mandatory rather than merely documented.
- [`a03ef333fbd6`][sb-public] "Documentation: security-bugs: explain what is and is not a security bug", Willy Tarreau, landed 2026-05-12. Source of both "you must treat it as public" and the "or use of AI-based tools" exception clause.
- [`4bf85afb9f3e`][sb-commit] "Documentation: security-bugs: clarify requirements for AI-assisted reports", Willy Tarreau, landed 2026-05-12, +57 lines. Source of the "Responsible use of AI to find bugs" section.

Those last two are separate commits, landed the same day, and it is worth not conflating them. Both reached mainline in Linus's merge [`36d49bba19f2`][docs-merge] of `docs-7.1-fixes`, which is the commit some writeups cite, including two of my own; the authoring commits are more precise.

Dates given are when each commit landed. Author dates run a little earlier in each case, most notably `78d979db6cef`, authored 2025-12-23.

**The worked example**

- [RFC v1][etna-v1], dri-devel, 2026-05-14.
- [PATCH v2][etna-v2], dri-devel, 2026-07-24, the patch this post describes auditing.
- [Ziyi Guo's parallel patch][etna-ziyi], 2026-05-08, and [Lucas Stach's review of it][etna-lucas], 2026-05-11, which supplied the `Suggested-by:`.

[ca]: https://docs.kernel.org/process/coding-assistants.html
[gc]: https://docs.kernel.org/process/generated-content.html
[sp]: https://docs.kernel.org/process/submitting-patches.html
[sb]: https://docs.kernel.org/process/security-bugs.html
[sc]: https://docs.kernel.org/process/submit-checklist.html
[tm]: https://docs.kernel.org/process/threat-model.html
[ca-commit]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=78d979db6cef557c171d6059cbce06c3db89c7ee
[gc-commit]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a66437c27979577fe1feffba502b9eadff13af7d
[sp-commit]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6252e5c1c20e
[sb-commit]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4bf85afb9f3e
[sb-public]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a03ef333fbd6
[docs-merge]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=36d49bba19f2
[etna-v1]: https://lore.kernel.org/all/20260514131401.2660079-1-clusk@northecho.dev/
[etna-v2]: https://lore.kernel.org/all/20260724222124.537101-1-clusk@northecho.dev/
[etna-ziyi]: https://lore.kernel.org/all/20260508180518.1417371-1-n7l8m4@u.northwestern.edu/
[etna-lucas]: https://lore.kernel.org/all/3e298ed6a361a0aa5526d859b0f3a98c0cd47090.camel@pengutronix.de/
