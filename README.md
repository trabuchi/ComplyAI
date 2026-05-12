# 🔒 ComplyAI

**Agentic compliance evidence for European regulated companies.**  
One plugin per `regulation × industry × country`. Installs into your existing observability stack. Audit-ready in 24 hours.

![Status](https://img.shields.io/badge/status-discovery%20%2F%20pilot-yellow)
![Regulations](https://img.shields.io/badge/regulations-DORA%20%7C%20NIS2%20%7C%20GDPR-blue)
![Data residency](https://img.shields.io/badge/data%20residency-EU%20only-green)
![Stack](https://img.shields.io/badge/stack-Datadog%20%7C%20Grafana%20%7C%20AWS-lightgrey)

---

## Table of Contents

- [The Problem](#-the-problem)
- [The Solution](#-the-solution)
- [Why This is an Agent Problem](#-why-this-is-an-agent-problem)
- [Handling Non-Determinism](#-handling-non-determinism)
- [Architecture](#-architecture)
- [Defensibility vs DIY + Claude Code](#-defensibility-vs-diy--claude-code)
- [Who This is For](#-who-this-is-for)
- [North Star Metric](#-north-star-metric)
- [MVP Status](#-mvp-status)

---

## 🔥 The Problem

SecOps and compliance teams at European mid-market companies spend an average of **11 hours per engineer per week** gathering, normalizing, and reformatting evidence that already exists somewhere in their stack.

| The reality today | Scale |
|---|---|
| Tools pivoted to answer one regulator question | 7+ |
| EU entities newly regulated under NIS2 (2025) | 160,000 (up from 15,000) |
| Financial firms pulled into DORA continuous compliance | 22,000 |
| Max GDPR fine on a failed audit | 4% of global revenue |
| Cost of a Big 4 audit per cycle | €50k–€200k, leaves nothing continuous |

The data to prove compliance already exists in your stack. The integration, the per-regulator mapping, and the audit calendar do not.

> Every infrastructure migration, regulation update, or scope change becomes a fire drill because there is no shared audit calendar and no continuous evidence layer.

---

## 💡 The Solution

A **plugin per `regulation × industry × country` triple**, installed directly inside Datadog, Grafana, or AWS. The engineer installs it once and within 24 hours has audit-ready dashboards, regulator-specific alerts, and automated response playbooks.

### What each plugin ships with

| Capability | Description |
|---|---|
| 📋 Control library | Regulator-specific clauses mapped to live infrastructure signals |
| 📊 Dashboard pack | Pre-built in the regulator's exact format |
| 🚨 Alert rules | Fire on control drift, naming the specific clause at risk |
| 🤖 SOAR playbooks | Auto-remediation (dry-run by default, opt-in for live execution) |
| 📦 Audit export | One-click package with full citation chain per claim |
| 📅 Forward calendar | Pre-stages evidence packages weeks before regulator deadlines |

### Example plugins

```
DORA × Fintech × Germany
NIS2 × Healthcare × France
GDPR × Public Sector SaaS × Spain
NIS2 × Construction × Italy
```

---

## 🤖 Why This is an Agent Problem

Mapping regulatory text to live infrastructure signals across 3–5 overlapping frameworks is a multi-step tool-calling workflow. Three properties make agentic AI the right fit:

**1. Regulation reading**  
Regulations are written in natural language. Hand-coded rules break every time a regulation is amended. An LLM reads the text directly and proposes the mapping.

**2. Planning across tools**  
One DORA control may require querying cloud audit logs, pulling access manifests, joining the ticketing system, fetching security events, formatting to the regulator's template, and triggering remediation. A classifier flags risk. An agent closes it.

**3. Cross-regulation deduplication**  
One technical control (e.g. encryption at rest with customer-held keys) satisfies clauses in DORA, NIS2, ISO 27001, and SOC 2 simultaneously, but each regulator wants it formatted differently. That is reasoning, not lookup.

---

## 🛡️ Handling Non-Determinism

Regulator-facing output cannot hallucinate. The agent loop is constrained at every layer:

| Guardrail | What it does |
|---|---|
| **Citation or suppress** | Every claim must cite a real log line, config, or ticket. No citation, no render. Ever. |
| **Human-in-the-loop** | The LLM drafts and routes novel mappings. A human decides before anything enters the evidence base. |
| **Behavioral contracts** | Agent must never silently skip a field. Below-threshold confidence flags for human review, not auto-tagging. Failing loudly is fine. Being quietly wrong is not. |
| **Full tool call logging** | All intermediate plans and tool calls are logged for forensic replay. |
| **Dry-run default** | Playbooks execute in dry-run mode unless the user explicitly opts in per playbook. |

> **Target: 0% hallucination rate on regulator-facing output**, enforced by citation-or-suppress at the export layer.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────┐
│   Customer observability stack                  │
│   Datadog / Grafana / AWS / Azure               │
└────────────────────┬────────────────────────────┘
                     │ OAuth · read-only
                     ▼
┌─────────────────────────────────────────────────┐
│   Discovery Agent                               │
│   Reads: logs · configs · IAM · network · tickets│
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│   Mapping Agent                                 │
│   Regulation clauses → live infrastructure signals│
│   Novel mappings → human review queue           │
│   Validated mappings → evidence base            │
└────────────────────┬────────────────────────────┘
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
┌──────────────────┐  ┌──────────────────────────┐
│   Operate layer  │  │   Export layer            │
│   Dashboards     │  │   Regulator-formatted     │
│   Clause alerts  │  │   audit package           │
│   SOAR playbooks │  │   Full citation chain     │
└──────────────────┘  └──────────────────────────┘
```

**LLM strategy:** Mistral or Aleph Alpha for regulated reasoning steps (EU data residency by architecture). Claude for non-regulated reasoning where appropriate. Data never leaves the customer's country.

---

## 🏰 Defensibility vs DIY + Claude Code

A team with 2 engineers and one quarter can wire Claude Code subagents to Datadog, AWS, and Jira to pull live evidence and draft DORA packages. That is real. Here is what they still cannot replicate:

| Moat | Why DIY cannot close this gap |
|---|---|
| 🧠 **Regulator institutional knowledge** | BaFin expects a specific XML schema with millisecond timestamps. AMF checks 12 specific DORA controls. This is not in any model's training data and not on the public web. It accumulates through paid audit cycles. |
| 🔄 **Audit feedback flywheel** | One customer gets one audit per regulation per year. We learn from N audits × N customers × N regulators. By year two, our mappings are sharper at no extra cost to any customer. |
| 🔧 **Permanent regulatory maintenance** | EU regulators amend guidance constantly. DIY means owning a permanent 0.5 FTE reading the Official Journal of the EU. We absorb this in the subscription. |
| 🗺️ **Cross-regulation deduplication graph** | One control evidenced once, formatted differently per regulator. A DIY team re-evidences the same control three times. |
| 🤝 **Auditor partnership network** | Plugins ship pre-validated with named external auditors per country. DIY negotiates format with the auditor from scratch every cycle, adding 4–8 weeks per audit. |

> **Honest concession:** a company with 10+ dedicated SecOps engineers, budget for a permanent compliance engineering function, and patience to spend two audit cycles tuning subagents could build a credible in-house version. That company is explicitly outside our ICP and we will not chase them.

---

## 🎯 Who This is For

### In scope

- Lean security and platform teams of **2–5 people**
- European mid-market companies, **200–500 employees**
- Regulated industries: fintech, healthcare, construction, public sector
- Facing **3+ overlapping EU frameworks** simultaneously
- Currently spending **€50k+ per audit cycle** on consultants or equivalent manual labor
- Operating in high-complexity reporting countries: DE, FR, IT, ES

### Out of scope

| Segment | Why |
|---|---|
| Single-regulation companies | Our offering would be overkill |
| Teams with 10+ dedicated SecOps engineers | They can build this themselves |
| Companies already on AWS Audit Manager / Azure Purview | Already have a working baseline |
| US-centric companies on Vanta or Drata | Purchasing sits outside our EU data residency advantage |

---

## 📈 North Star Metric

> **% of audit evidence that was auto-collected and audit-ready, per customer per quarter**

This captures the full transformation from manual fire drills to continuous, regulator-ready evidence.

| | Baseline (manual) | Year 1 target |
|---|---|---|
| Auto evidence coverage | 10–20% | **70%+** per installed plugin |
| Engineer hours per audit cycle | ~11 hrs/week ongoing | **60% reduction** within 2 quarters |
| Time from audit announced to package ready | Several weeks | **Under 5 business days** |

**Guardrails that cannot slip:**
- Evidence accuracy rate (claims surviving auditor challenge): 99%+
- Hallucination rate on regulator-facing output: 0%
- False positive alert rate: under 5%

---

## 🚀 MVP Status

Building toward an MVP scoped to **`DORA × Fintech × Germany`** on a reference stack of AWS + Datadog + Jira.

**Launch criteria:**

- [ ] 70%+ auto evidence coverage on DORA / fintech vertical
- [ ] 99%+ evidence accuracy verified against a labeled benchmark
- [ ] Under 24 hours time-to-first-evidence on reference stack
- [ ] 0% hallucination rate on regulator-facing output
- [ ] At least one paid pilot has passed a real audit with the exported package

---

*Building this in public. Open to conversations with SecOps engineers, compliance leads, and anyone who has survived a DORA or NIS2 audit and has opinions.*
