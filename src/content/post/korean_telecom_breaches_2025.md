---
title: "South Korea's Telecom Breaches: SKT, KT, and LG U+"
description: "A compact comparison of three telecom security incidents, confirmed findings, and the engineering lessons."
publishDate: 2026-10-19
tags: [security, incident-analysis, telecommunications]
draft: false
---

Three telecom investigations illustrate **different security failures**: undetected server compromise, weak network-device authentication, and the loss of forensic evidence.

## At a Glance

| Operator | Incident | Confirmed problem |
| --- | --- | --- |
| **SK Telecom (2025)** | USIM information breach | Long-term malware infection, weak credential controls, and inadequate previous incident response |
| **KT (2025)** | Rogue femtocells and unauthorized payments | Weak device authentication and insufficient protection of signaling information |
| **LG U+ (2025)** | Reported system intrusion | Reinstalled operating systems and discarded servers limited forensic verification |

## 1. SK Telecom: Persistent Malware

**Confirmed impact:** The government identified **28 infected servers**, **33 malware variants** (including BPFDoor), and **9.82 GB of leaked USIM data**, covering approximately **26.96 million IMSI records**.

**Root issues:** Weak account credential management, inadequate response to a 2022 compromise, and insufficient encryption of critical information. Routine checks also failed to detect a web shell.

**Lesson:** Incident response must verify that persistence mechanisms and compromised credentials have been eliminated, not merely that individual malicious files were removed.

## 2. KT: Unauthorized Network Equipment

**Confirmed impact:** The final government investigation identified **22,227 affected subscribers** whose identifying information was exposed through rogue femtocells, with **368 people** suffering unauthorized micro-payment losses totaling approximately **KRW 243 million**.

**Root issues:** Device certificate management, weak network admission controls, inadequate validation of device identity and connection origin, and the ability of rogue equipment to undermine communications encryption. The separate server investigation also found **94 infected servers** and **103 malware variants**.

**Lesson:** A valid-looking certificate is insufficient if the underlying device, origin, and authorization state are not verified.

## 3. LG U+: Missing Forensic Evidence

**Confirmed finding:** Following reports of a possible intrusion, investigators found that relevant servers had been discarded or their operating systems reinstalled, preventing complete forensic examination.

**Limit:** Missing evidence does not itself establish the precise attack path or the amount of data, if any, that was exfiltrated.

**Lesson:** Preserve disk images, logs, and volatile evidence before rebuilding systems. Otherwise the organization may lose the ability to establish what happened.

## What These Cases Have in Common

| Security requirement | Why it matters |
| --- | --- |
| Credential and device lifecycle controls | Prevent reuse of compromised identities |
| Continuous monitoring | Detect stealthy persistence and unusual connections |
| Evidence retention | Establish the attack timeline and actual scope |
| Verified remediation | Check that access paths are closed without breaking services |

**A security control is only useful when its effectiveness can be demonstrated through detection, investigation, and validation.**

## Sources

- [MSIT: SK Telecom final investigation, July 2025](https://www.korea.net/Government/Briefing-Room/Press-Releases/view?articleId=8120&insttCode=&type=O)
- [MSIT: KT and LG U+ final investigation briefing, December 2025](https://www.korea.kr/briefing/policyBriefingView.do?newsId=156737260)

*Confirmed findings are distinguished from investigative limitations and engineering recommendations.*
