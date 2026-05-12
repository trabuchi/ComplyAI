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
- [Agent Evaluation Design](#-agent-evaluation-design)
- [Handling Non-Determinism](#-handling-non-determinism)
- [Architecture](#-architecture)
- [ROI Threshold](#-roi-threshold)
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

## 🔬 Agent Evaluation Design

You cannot unit-test an LLM. With deterministic code you assert outputs. With agents you assert qualities: was the evidence accurate, was it grounded in the right infrastructure signal, did the agent call the right tools in the right order, did it stay within the clause's scope? These are judgment calls, which means evaluation is a system you have to build and maintain, not a test suite you write once.

ComplyAI runs five evaluation layers, each tied to a specific stage in the agent pipeline.

### Layer 1: Golden dataset evals (Mapping Agent)

Before any plugin ships, we build a labeled dataset of known mappings: clauses we know map to a specific log pattern, clauses we know have no matching signal in a standard AWS + Datadog stack, and edge cases specific to that regulation and country (e.g. BaFin's millisecond timestamp requirement for DORA key rotation evidence).

Every model change and every regulation update gets scored against this set before it touches a customer environment. A regression on the golden set is a deploy blocker, not a warning.

```
golden_dataset/
  dora_fintech_de/
    known_mappings.jsonl       # clause → signal, expected citation format
    known_gaps.jsonl           # clauses with no standard signal → must flag, not guess
    edge_cases.jsonl           # country-specific formatting, threshold, schema rules
```

The discipline is building these datasets before you need them, and treating them like a product artifact that gets maintained as regulations change.

### Layer 2: Trajectory evaluation (all agent stages)

For agentic systems, the final answer is not enough to evaluate. A confident answer built on a bad tool call is worse than admitting uncertainty. For each agent run, we log and score the full reasoning path:

| Check | What we verify |
|---|---|
| Tool call correctness | Did the agent query the right data source for this clause? |
| Tool call order | Did it retrieve evidence before attempting to map, not after? |
| Context use | Did it use the retrieved signal, or did it ignore it and reason from weights? |
| Scope containment | Did it stay within the clause boundary or drift into adjacent controls? |
| Failure handling | When a tool call returned empty, did it flag the control as `needs_review` rather than guessing? |

Trajectory logs feed a weekly review queue. Any run where the agent reached the right answer via a wrong path gets flagged for prompt refinement, not celebrated as a win.

### Layer 3: LLM-as-judge (Export layer)

Before any claim renders in an audit package, a second model pass scores every output against a structured rubric:

```
Evaluation rubric (per claim):
  - Grounded: does the claim reference a real, timestamped source? (binary)
  - Accurate: does the claim correctly represent what the source says? (0–1 score)
  - Scoped: is the claim within the boundary of the cited regulatory clause? (binary)
  - Formatted: does the output match the regulator's required schema? (binary)
```

Any claim that fails `Grounded` (binary) is suppressed before export, not flagged for review. A claim in the audit package that cannot be traced to a source is worse than a gap.

The judge model is separate from the mapping model and runs on a stricter prompt. We use this as one signal, not the only one. Judge models tend to favor outputs that sound like themselves, so it runs alongside the golden dataset score, not instead of it.

### Layer 4: Behavioral contracts (runtime, all stages)

Accuracy metrics tell you how often the agent is right on known cases. Behavioral contracts tell you how it fails. These are non-negotiable invariants that run at every stage regardless of model version:

| Contract | Stage | Enforcement |
|---|---|---|
| Never silently skip a field | Discovery | Hard assertion, pipeline fails if violated |
| Never render a claim without a citation | Export | Suppressed at render time, logged as gap |
| Novel mappings must route to human queue | Mapping | Enforced by the mapping agent's output schema |
| Failed tool calls must set control status to `needs_review` | All | Hard assertion in tool call wrapper |
| Confidence below threshold must escalate, not auto-tag | Mapping | Threshold configured per regulation, not per model |

The distinction between behavioral contracts and accuracy metrics matters for a compliance product. An auditor does not care that the agent was right 99% of the time. They care that when it was wrong, it said so.

### Layer 5: Shadow mode + canary evaluation (pre-release)

New plugin versions and model updates run in shadow mode against real customer infrastructure before going live. The new version runs in parallel with the current one, outputs are compared, and differences surface in a review queue. The new version does not reach production until:

- Golden dataset score is equal or higher than the current version
- No new behavioral contract violations
- Trajectory quality score is equal or higher on the last 30 days of real runs
- At least one compliance lead has reviewed the diff on a real customer dataset

This is how we catch regressions before they hit an audit package.

### The north star for evaluation

> **Would a compliance lead submit this audit package to their regulator without reviewing it line by line?**

That is the question every evaluation layer is proxying. Everything else is instrumentation to get closer to a confident yes.

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

## 💰 ROI Threshold

A product is only worth buying if the value it delivers exceeds its cost by a margin that justifies the switching effort. For ComplyAI, the ROI calculation is concrete enough to put a number on.

### The baseline cost of doing nothing

For a 3-person SecOps team at a European mid-market fintech facing DORA, NIS2, and GDPR simultaneously:

| Cost category | Annual estimate |
|---|---|
| Engineer time on evidence gathering | 3 engineers × 11 hrs/week × €80/hr fully loaded = **€137k/year** |
| External audit consultant fees | 2 audits/year × €75k avg = **€150k/year** |
| Risk of a failed audit (expected value) | GDPR fine floor at 2% of €50M ARR = **€1M exposure** |
| Opportunity cost (engineering diverted from product) | 1 FTE equivalent absorbed = **€120k/year** |
| **Total annual cost of status quo** | **~€407k/year in direct costs, €1M+ in tail risk** |

### What ComplyAI needs to deliver to justify the purchase

The product earns its subscription if it clears **two thresholds simultaneously**:

**Threshold 1: Time payback under 6 months**  
The customer recovers the annual subscription cost through saved engineer hours alone within one audit cycle. At a €24k/year subscription (€2k/month), the break-even is 30 engineer hours saved. ComplyAI targets 60% reduction in audit prep time. On a 3-person team that is roughly 20 hours/week recovered. Break-even in under 2 weeks of use.

**Threshold 2: First audit package passes without a material finding**  
If the first exported audit package survives regulator review without a citation challenge or a gap finding that the customer has to scramble to close, the product has proven its core claim. One passed audit at €75k consultant replacement value justifies the annual subscription by itself.

### The minimum viable ROI the product must not fall below

| Signal | Minimum bar | How we measure it |
|---|---|---|
| Engineer hours recovered | 40% reduction by end of quarter 1 | Time-tracked in customer kickoff survey vs. 90-day check-in |
| Audit package acceptance | First exported package passes without material citation challenge | Customer-reported post-audit debrief |
| Time to first evidence | Under 24 hours from install on reference stack | Automated telemetry from plugin install event |
| Compliance lead trust score | Would submit the package without line-by-line review | Quarterly NPS-style survey, single question |

If a customer hits quarter 2 without clearing threshold 1, that is a product failure, not a customer success problem. It means the auto-coverage rate for their specific regulation and stack did not reach the promised level. We treat that as a P1.

### Why the ROI calculation matters for pricing

The unit economics above set a floor on what we can charge and a ceiling on what we need to promise. A €24k/year plugin subscription is defensible when the customer is replacing €75k+ in consultant fees and recovering €137k in engineering time. It is not defensible if auto-coverage is below 50% and the customer is still running the same manual process alongside the plugin.

This means the **evidence coverage rate is not just a product metric, it is a pricing prerequisite**. We do not expand to a second plugin until the first one clears 70% auto-coverage. We do not charge renewal until the first audit passes.

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
