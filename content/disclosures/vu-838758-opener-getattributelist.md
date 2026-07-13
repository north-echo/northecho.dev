---
title: "OpENer GetAttributeList — Out-of-Bounds Read and Stack Response Overflow"
date: 2026-07-13
summary: "An unauthenticated attacker can send a single crafted GetAttributeList request that reads past the request buffer and corrupts stack-based response assembly in OpENer, reaching a stack-disclosure primitive under ASAN."
vendor: "EIP Stack Group"
product: "OpENer (EtherNet/IP stack)"
caseId: "VU#838758"
status: "unpatched"
reportedDate: 2026-04-08
disclosedDate: 2026-07-13
cves:
  - id: "CVE Pending"
    title: "GetAttributeList out-of-bounds read and stack response overflow"
    cvss3: 9.1
    cvss4: 8.8
    cwe: "CWE-121"
tags: ["ics", "ot", "vulnerability-disclosure", "opener", "ethernet-ip"]
---

- **Advisory ID:** NESR-2026-0001 · **CVE:** requested, pending
- **CWE:** CWE-121 — Stack-based Buffer Overflow
- **CVSS v3.1:** 9.1 — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:H`
- **CVSS v4.0:** 8.8 — `AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:N/VA:H`

Coordinated via CISA VINCE; consolidated CISA ICSA pending. Published by North Echo 2026-07-13.

## Summary

An unauthenticated, remote attacker can send a single crafted EtherNet/IP GetAttributeList request (after the unauthenticated RegisterSession) that is processed by `GetAttributeList()` in `cipcommon.c`. The handler reads a 16-bit `attribute_count_request` from the network and iterates that many times without validating it against the actual request size, and its per-entry response-space accounting omits fixed metadata bytes. The result is an out-of-bounds read of the request stream and malformed response assembly on a stack-backed buffer.

## Affected

- OpENer (EIPStackGroup/OpENer) — all versions through master commit `76b95cf`; no fix merged as of publication.
- Any product embedding the affected OpENer code, default `PC_OPENER_ETHERNET_BUFFER_SIZE = 512`.

## Technical detail

`GetAttributeList()` reads the attacker-controlled count and loops without bounds:

```c
CipUint attribute_count_request = GetUintFromMessage(&message_router_request->data);
for (size_t j = 0; j < attribute_count_request; j++) {
    attribute_number = GetUintFromMessage(&message_router_request->data);  // OOB read past request
    ...
}
```

Two defects compound:

1. **Request-side:** `attribute_count_request` is never checked against `request_data_size`, so once the count exceeds the attribute IDs actually present, each iteration reads 2 bytes beyond the request buffer.
2. **Response-side:** the space check (`needed_message_space > remaining_message_space`) omits the fixed per-entry metadata that is always written — attribute-ID (2) + status (1) + reserved (1). It undercounts by 2–4 bytes per entry (2 bytes for a not-gettable/not-supported attribute where `needed = 2*sizeof(CipSint)`; 4 bytes for a gettable attribute where `needed` counts only the value length). Corruption of `current_message_position` propagates into the later `send()` path on the stack-allocated `ENIPMessage`.

This is the same function as CVE-2022-43604 (TALOS-2022-1637); that fix added the space check but undercounts as above and does not bound the request count.

## Impact (conservatively characterized)

- **Confirmed across builds:** reliable per-connection denial of service — the malformed request wedges the attacker's connection on a default non-ASAN RelWithDebInfo build (fresh sessions still succeed).
- **Confirmed on ASAN-instrumented build:** stack-backed over-read reaching `send()` (READ of size 524288 reported by AddressSanitizer), returning oversized stack-backed data to the client while the encapsulation header still advertises only the declared length — i.e., a stack-disclosure primitive.
- **Not demonstrated:** code execution. We do not claim a precisely bounded leak size on a non-ASAN build.

## Mitigation

No upstream fix at publication. Restrict TCP/44818 to trusted hosts; segment EtherNet/IP devices. Suggested code fix:

```c
CipUint attribute_count_request = GetUintFromMessage(&message_router_request->data);
if (attribute_count_request * 2 > message_router_request->request_data_size - 2) {
    message_router_response->general_status = kCipErrorNotEnoughData;
    return kEipStatusOkSend;
}
```

…and correct the per-entry response accounting to include the attribute-ID + status + reserved bytes actually written.

## Disclosure timeline

- **2026-04-08** — Reported to CISA VINCE; VU#838758 assigned.
- **2026-05-26** — Original expected-public date lapsed with no coordination.
- **2026-06-10** — CISA re-engaged after a status-check; vendor outreach initiated.
- **2026-06-15** — CWE re-tagged to CWE-121 at researcher's request; draft ICSA revised.
- **2026-06-30** — Proposed consolidated publication date lapsed without release.
- **2026-07-13** — Published by North Echo per prior notice. Consolidated CISA ICSA pending.

## Independent discovery

Related defects in this function were independently reported on the public OpENer issue tracker in 2026 (e.g. issues #567, #576). North Echo's coordinated report (2026-04-08) predates those public filings.

## References

- CISA VINCE VU#838758 · CVE requested (pending)
- Related: CVE-2022-43604 / TALOS-2022-1637
- Companion finding: VU#982645 (/disclosures/vu-982645-opener-setattributesingle/)
