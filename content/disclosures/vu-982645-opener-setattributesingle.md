---
title: "OpENer SetAttributeSingle — Truncated Write Accepted and Applied to Live State"
date: 2026-07-13
summary: "OpENer accepts SetAttributeSingle requests with zero- or partial-byte payloads, applies the truncated value to live device state, and reports success — an unauthenticated integrity bug that crash-oriented fuzzing can't see."
vendor: "EIP Stack Group"
product: "OpENer (EtherNet/IP stack)"
caseId: "VU#982645"
status: "unpatched"
reportedDate: 2026-04-08
disclosedDate: 2026-07-13
cves:
  - id: "CVE Pending"
    title: "SetAttributeSingle truncated write accepted and applied to live state"
    cvss3: 8.2
    cvss4: 8.8
    cwe: "CWE-1284"
tags: ["ics", "ot", "vulnerability-disclosure", "opener", "ethernet-ip"]
---

**Advisory ID:** NESR-2026-0002 · **CVE:** requested, pending
**CWE:** CWE-1284 Improper Validation of Specified Quantity in Input
**CVSS v3.1:** 8.2 — `AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:L`
**CVSS v4.0:** 8.8 — `AV:N/AC:L/AT:N/PR:N/UI:N/VC:N/VI:H/VA:L`
**Status:** Coordinated via CISA VINCE (lead case); consolidated CISA ICSA pending. Published by North Echo 2026-07-13.

## Summary

An unauthenticated, remote attacker can send a SetAttributeSingle request carrying a truncated (zero- or partial-byte) payload that OpENer accepts as success and applies to live device state. `SetAttributeSingle()` dispatches to the target attribute's decoder without validating that the request carries enough data. Of its 18 attribute decoders, only the byte-array decoder checks input length; all 17 scalar and string decoders — including the UINT decoder — read their fields unconditionally.

## Affected

- OpENer (EIPStackGroup/OpENer) — all versions through master commit `76b95cf`; no fix merged as of publication (the scalar/string decoders remain unguarded).
- Any product embedding the affected OpENer code.

## Technical detail

The service handler resolves a settable attribute and calls its decoder directly:

```
RegisterSession → SendRRData → CPF → Message Router → SetAttributeSingle → attribute->decode(...)
```

No check enforces a minimum `request_data_size` before `attribute->decode()` (`cipcommon.c` line 823). Of the 18 decoders, only `DecodeCipByteArray` validates length (`request_data_size < data->length` → `kCipErrorNotEnoughData`, and `>` → `kCipErrorTooMuchData`); the 17 scalar/string decoders — e.g. `DecodeCipUint` (line 931) — call `GetUintFromMessage()` and set `general_status = kCipErrorSuccess` unconditionally. A request shorter than the attribute width reads into the zero-initialized receive buffer, applies the truncated value, and returns success. Confirmed against reachable settable attributes including TCP/IP interface configuration control, QoS DSCP values, and the encapsulation inactivity timeout.

## Impact (conservatively characterized)

- **Unauthenticated remote integrity impact:** attacker-forced state changes to live device configuration via zero/partial-byte writes that the stack accepts and acknowledges as successful.
- This is a logic/validation defect, not memory corruption — a truncated write that succeeds produces no crash and is not surfaced by crash-oriented fuzzing. That distinguishes it from the memory-safety reports of the same code path.

## Mitigation

No upstream fix at publication. Restrict TCP/44818 to trusted hosts; segment EtherNet/IP devices. Fix direction: enforce a per-attribute minimum input length in `SetAttributeSingle()` before dispatch, or replicate the existing `DecodeCipByteArray` length check into each scalar/string decoder — the guard already exists in one decoder and simply needs to cover the rest.

## Disclosure timeline

- **2026-04-08** — Reported to CISA VINCE; VU#982645 assigned (lead case for the consolidated advisory).
- **2026-06-10** — CISA re-engaged after a status-check; vendor outreach initiated.
- **2026-06-23** — Vendor onboarded to the VINCE case.
- **2026-06-30** — Proposed consolidated publication date lapsed without release.
- **2026-07-13** — Published by North Echo per prior notice. Consolidated CISA ICSA pending.

## Independent discovery

A memory-safety issue reaching the same SetAttributeSingle decode path (out-of-bounds read/DoS) was independently reported on the public OpENer issue tracker in 2026 (issue #569). North Echo's coordinated report (2026-04-08) predates that filing and characterizes a distinct integrity consequence — an accepted truncated write — not covered by the public crash-oriented report.

## References

- CISA VINCE VU#982645 · CVE requested (pending)
- Companion finding: VU#838758 (/disclosures/vu-838758-opener-getattributelist/)
