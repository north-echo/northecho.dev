---
title: "Two OpENer Findings, and a Coordinated Disclosure That Almost Worked"
date: 2026-07-13
summary: "Two coordinated-disclosure vulnerabilities in OpENer, a widely embedded EtherNet/IP stack — a stack-based overflow and an accepted truncated write — and the two silences it took a public forcing function to break."
tags: ["ics", "ot", "vulnerability-disclosure", "opener", "ethernet-ip", "disclosure-process"]
---

Between April and July of 2026 we took two vulnerabilities in [OpENer](https://github.com/EIPStackGroup/OpENer) — a widely embedded open-source EtherNet/IP stack — through coordinated disclosure with CISA. This post is about both the bugs and the process, because the process is the more instructive half. Coordination worked, eventually, but only after a forcing function; and then it went quiet again at the finish line. We're publishing on the date we said we would.

The canonical, machine-linkable records are the two advisories:

- [VU#838758 — GetAttributeList out-of-bounds read and stack response overflow](/disclosures/vu-838758-opener-getattributelist/)
- [VU#982645 — SetAttributeSingle truncated write accepted and applied to live state](/disclosures/vu-982645-opener-setattributesingle/)

## The two bugs

Both are reachable by an unauthenticated attacker who can complete a RegisterSession on TCP/44818 — which, on OpENer, always succeeds.

GetAttributeList (VU#838758) is a memory-safety bug in the same function as CVE-2022-43604. A 16-bit `attribute_count_request` is read from the wire and used as a loop bound with no check against the actual request size, so the handler reads past the request buffer. Worse, the response-space accounting that the 2022 patch added undercounts: it never includes the fixed per-entry metadata (attribute-ID + status + reserved) that is always written back, undercounting by 2–4 bytes per entry depending on whether the attribute is gettable. On a default build the malformed request reliably wedges the attacker's connection; under AddressSanitizer it drives an oversized stack-backed read into the `send()` path — a stack-disclosure primitive. We don't claim code execution, and we don't claim a clean, bounded leak size on a non-ASAN build. What's solidly true is: remote, unauthenticated, single packet, and it corrupts response assembly on the stack.

SetAttributeSingle (VU#982645) is the one we care about more, because it isn't a crash at all. `SetAttributeSingle()` dispatches to an attribute's decoder without checking that the request actually carries enough bytes. So a truncated — even zero-byte — write is accepted, applied to live device state, and acknowledged as success. We confirmed it against real settable attributes: TCP/IP interface configuration control, QoS DSCP, the encapsulation inactivity timeout. This is an integrity bug (CWE-1284), not memory corruption.

## Anatomy of the truncated write

Here's what makes VU#982645 concrete. The Encapsulation Inactivity Timeout is attribute 13 (0x0D) of the TCP/IP Interface Object (class 0xF5, instance 1) — a 2-byte UINT. A well-formed Set_Attribute_Single request to change it looks like this at the CIP layer (carried inside the usual SendRRData → CPF → Message Router envelope):

```
service   0x10                 Set_Attribute_Single
path      20 F5 24 01 30 0D    Class 0xF5, Instance 0x01, Attribute 0x0D
data      3C 00                the new value (0x003C = 60), 2 bytes
```

Now send the same request with the data field removed:

```
service   0x10                 Set_Attribute_Single
path      20 F5 24 01 30 0D    Class 0xF5, Instance 0x01, Attribute 0x0D
data      (empty)              zero bytes
```

A correct implementation rejects this — the attribute is a UINT, the request carries no UINT. OpENer does not. `SetAttributeSingle()` resolves attribute 0x0D and calls its decoder, `DecodeCipUint`, which reads its two bytes without ever asking whether two bytes are present. (Tellingly, OpENer does length-check exactly one decoder, `DecodeCipByteArray`; the guard was simply never applied to the 17 scalar and string decoders, this one included.) The two bytes come from the zero-initialized receive buffer, the value is written, and the device answers:

```
reply     0x90                 Set_Attribute_Single response
status    00                   success
```

Success. The device just accepted a configuration change it had no data for. Point the same shape at TCP/IP configuration control or QoS DSCP and you're mutating live network behavior of the device, unauthenticated, from a single short packet — with the device reporting that everything went fine.

## Why a fuzzer never finds this

There's a reason this bug sat in a well-scrutinized, previously-CVE'd stack. Look at the two findings side by side: the GetAttributeList overflow is a crash. Throw malformed length fields at the parser and AddressSanitizer lights up — a fuzzer finds it, and in fact several memory-safety issues in this code have been found exactly that way and filed publicly.

The truncated write is the opposite. Nothing crashes. No sanitizer fires. The code path completes cleanly and returns 0x00. To a coverage-guided, crash-oriented fuzzer, a request that is accepted and acknowledged is indistinguishable from correct behavior — it's a green light, not a finding. The bug is only visible if you're asking a different question: not "what input makes this fall over?" but "what is this handler willing to accept that it never should?"

That question comes from reading the dispatch code and noticing there is no length precondition before `attribute->decode()` — and then asking what happens if you simply don't send the data. It's a static-first, logic-level lens, and it's why North Echo leads with the code rather than the harness. Memory-safety and integrity bugs live in the same functions here; they just answer to different methods. The overflow was co-discovered in the open. The accepted truncated write was not — and that asymmetry is not a coincidence.

## The disclosure

We filed both with CISA VINCE on 2026-04-08. Then nothing — for 64 days. The expected-public date came and went with no coordinator contact and no vendor outreach.

On 2026-06-09 we posted a status-check to both case threads announcing a unilateral publication target. That was the forcing function, and it worked: within 48 hours CISA re-engaged, reached out to the maintainer for the first time in the case's life, and began drafting a consolidated advisory. Real coordination followed — we pushed back on the CWE for VU#838758, and CISA re-tagged it to CWE-121 (stack-based buffer overflow) within days. The vendor was onboarded to the case on 2026-06-23. Publication was set for 2026-06-30.

And then it went quiet again. The 06-30 date lapsed with no advisory, no revised date, no vendor review comment. On 2026-07-06 we posted notice of a firm publication date of 2026-07-13. That date is today. So here we are.

None of this is a knock on the individuals involved — coordination is hard and under-resourced, and when it engaged it engaged in good faith. But the pattern is worth naming for other researchers: silence is the default failure mode, and a clear, dated, unilateral-publish commitment is what breaks it. Twice, here.

## On independent discovery

We weren't the only ones looking at OpENer. Through 2026 a number of related issues were filed publicly on the project's issue tracker with proof-of-concept code (for example #567, #569, #576, and more recently #592/#593). Those are real findings and worth reading. Two things are true at once: the root causes in this code are being co-discovered in the open, and our coordinated report predates the public filings (2026-04-08). On VU#982645 in particular, the public reports describe a memory-safety crash on the same path; the accepted truncated write — the integrity consequence — is ours, for the reasons the methodology section lays out.

**Update, July 14, 2026:** A third independent researcher, GitHub user [MrAlaskan](https://github.com/MrAlaskan), has been filing OpENer issues since April 23, 2026 (issues #562–#565, #568, #582) — distinct bugs from both of our findings and from the issues referenced above. On July 13, five of these were assigned CVE IDs (CVE-2026-51536, -51537, -51538, -51540, -51541) through what appears to be an automated CNA pipeline; none have been analyzed or vetted by NVD as of this writing (all show "Received" or "Deferred" status). None overlap with VU#838758 or VU#982645. That's at least three independent research efforts finding unauthenticated remote defects in this codebase since April 2026, none patched, none acknowledged by the maintainer. Our own CVE requests for both findings remain pending.

## Detection and mitigation

There is no upstream fix merged as of publication. If you run OpENer, or a product that embeds it, restrict TCP/44818 to trusted hosts and segment your EtherNet/IP devices. The specific code fixes are in each advisory.

Both findings are detectable on the wire without device cooperation:

- **GetAttributeList overflow (VU#838758):** flag any Get_Attribute_List (service 0x03) request whose declared attribute count is implausible against the request's actual payload length — i.e. count × 2 exceeds the bytes present. Legitimate clients never over-declare.
- **SetAttributeSingle truncated write (VU#982645):** flag any Set_Attribute_Single (service 0x10) request whose CIP data field is shorter than the target attribute's defined width — most starkly, a Set with a zero-length data field. A well-formed configuration write always carries the full value.

Either signature is cheap to express in a passive EtherNet/IP monitor and neither produces false positives against conformant traffic.

CVE identifiers have been requested and are pending; we'll update the advisories when they land.

---

Findings reported under North Echo's coordinated disclosure practice. Advisories: [VU#838758](/disclosures/vu-838758-opener-getattributelist/) · [VU#982645](/disclosures/vu-982645-opener-setattributesingle/).
