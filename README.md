# AML Triage Agent

An AI-powered anti-money laundering triage system built on the Anthropic Claude API. 
The agent evaluates counterparty profiles for sanctions exposure, jurisdictional risk, and AML red flags, returning structured risk assessments with reasoning grounded in FATF typologies.

Built by someone who spent two years doing this manually at PwC.

---

## The Problem

AML analysts spend significant time on first-pass triage - checking sanctions lists, 
assessing jurisdiction risk, and evaluating whether a counterparty's transaction 
pattern matches their stated business purpose. This work is rule-heavy, repetitive, 
and well-suited to automation. The judgment calls at the boundary — the genuinely 
ambiguous cases — are where human expertise adds irreplaceable value.

This project explores where that boundary sits when Claude handles the first pass.

---

## Architecture

The agent uses Claude's tool use (function calling) to run a multi-step agentic loop:

1. Receives a structured counterparty profile as input
2. Decides which tools to call based on the case (sanctions check, jurisdiction lookup)
3. Calls external tools mid-reasoning to retrieve live data
4. Synthesises tool results with case facts to produce a structured JSON risk assessment

Input Case → Claude (reasoning) → Tool Calls → Claude (synthesis) → Structured Output

↑                                    |

└────────── Tool Results ────────────┘

### Tools
- **`check_sanctions(entity_name)`** — Queries the OpenSanctions API for sanctions matches
- **`get_jurisdiction_risk(country)`** — Returns FATF risk rating from a curated lookup table

### Output Schema
Every case returns a deterministic JSON object:
```json
{
  "case_id": "string",
  "risk_score": 0-100,
  "risk_category": "low | medium | high",
  "red_flags": ["flag 1", "flag 2"],
  "reasoning": "2-3 sentence explanation",
  "requires_human_review": true/false
}
```

---

## Evaluation

Evaluated against a 20-case labelled dataset spanning five categories: obvious high 
risk, obvious low risk, ambiguous medium risk, adversarial cases, and edge cases. 
Ground truth labels were set by the author prior to running the eval, based on 
two years of AML/FCPA analysis experience at PwC.

> **Note:** Ground truth labels were set by the author before running the eval. 
> The medium/high boundary is subjective — see Key Finding below for a full discussion of what the Haiku vs Sonnet comparison actually shows.

### Results

| Metric | Claude Haiku | Claude Sonnet |
|--------|-------------|---------------|
| Accuracy | 100% | 90% |
| Precision | 100% | 71.4% |
| Recall | 100% | 100% |
| F1 Score | 100% | 83.3% |
| Avg Latency | 6.18s | 14.14s |
| Avg Input Tokens | 1,376 | 1,416 |
| Avg Tool Calls/Case | 2.7 | 2.9 |


### Key Finding

At first glance, Haiku appears to outperform Sonnet — 100% accuracy versus 90%. 
This framing is misleading and worth examining carefully.

Sonnet's two errors are both false positives on the two most ambiguous cases in 
the dataset: C003 (BVI shell company, undisclosed beneficial ownership, payments 
from firms with procurement irregularities) and C014 (freight forwarder routing 
funds through Russia and Iran). Both cases sit genuinely at the medium/high 
boundary. A more conservative analyst — or a different labelling decision — 
would have marked both as high risk, which would flip Sonnet's "errors" into 
correct predictions and give Sonnet 100% accuracy too.

The real finding is not that Haiku is more accurate. It is that Sonnet applies 
more conservative judgment on ambiguous cases, which in AML is arguably the 
correct behaviour — false negatives (missed risks) are more costly than false 
positives (unnecessary reviews). The dataset also has a known limitation: 
ground truth labels were set by the same person who designed the cases, 
introducing selection bias toward the boundary decisions one analyst would make.

A production-grade eval would require a larger dataset with independently 
verified ground truth labels. This eval is best understood as a proof of concept 
demonstrating the framework, not a definitive performance benchmark.

**Recommended architecture for a startup deployment:** Haiku for high-volume 
first-pass screening, Sonnet reserved for cases Haiku flags as borderline. 
This hybrid approach reduces per-case cost while concentrating Sonnet's deeper 
reasoning where it matters most.

### Limitations

- 20-case dataset designed by the same person who labelled the ground truth — 
  introduces selection bias toward cases the model handles well
- Medium/high boundary is subjective; a different analyst might relabel C003 
  and C014 as high, which would flip Sonnet's "errors" to correct predictions
- Production validation would require a larger, independently labelled dataset

---

## Cost Analysis

| Model | Avg Input Tokens | Approximate Cost/Case* |
|-------|-----------------|----------------------|
| Claude Haiku | 1,376 | ~$0.0002 |
| Claude Sonnet | 1,416 | ~$0.002 |

*Based on Anthropic's published API pricing. Sonnet costs approximately 10x 
more per case than Haiku on this workload.

At 10,000 cases/day:
- Haiku: ~$2/day
- Sonnet: ~$20/day
- Hybrid (Haiku first-pass, Sonnet on 10% flagged cases): ~$3.80/day

---

## What I'd Build Next

The current tool set — sanctions screening and jurisdictional risk — covers the 
two highest-signal, lowest-latency checks appropriate for a first-pass triage 
layer. These are the questions with binary answers: is this entity sanctioned? 
Is this jurisdiction flagged? The harder, more valuable layer is the one that 
catches risk before it crystallises into a formal sanction.

**Adverse media monitoring** is the most impactful next addition. Sanctions 
lists are backwards-looking — they flag people who have already been caught. 
A news API integration (GDELT, NewsAPI, or a commercial provider like 
Refinitiv) scanning for negative coverage on entities, beneficial owners, and 
associated counterparties would surface emerging risk in real time. In my PwC 
work, adverse media was often the first signal that a client relationship 
needed re-evaluation — weeks or months before any formal regulatory action.

**Regulatory enforcement search** would add structured lookups against public 
enforcement databases: FCA register, Central Bank of Ireland, SEC enforcement 
actions, DOJ FCPA settlements. These are publicly accessible APIs that surface 
prior regulatory history on both companies and individuals. Given that this 
project was built by someone who spent two years tracking FCPA violations, 
this tool would be the most personally familiar to implement — and the most 
immediately useful for a compliance-focused startup customer.

**Legal record lookup** via jurisdiction-specific court record APIs — PACER 
in the US, Companies House in the UK — would add a third verification layer 
for entity and individual litigation history. Coverage is fragmented by 
jurisdiction and comprehensive sources are typically commercial, but targeted 
lookups on high-risk flagged cases would meaningfully reduce false negatives.

Together, these additions would implement a genuinely tiered architecture: 
fast automated screening first, deeper investigation tools triggered only on 
flagged cases, with human analysts reserved for the genuinely ambiguous 
boundary cases where judgment — not pattern matching — is what's needed. 
That boundary is exactly where this eval showed the most interesting failures.

---

## Stack

- **LLM:** Anthropic Claude API (Haiku for dev/eval, Sonnet for production comparison)
- **Tool use:** Native Anthropic function calling (no LangChain)
- **Sanctions data:** OpenSanctions API (free tier)
- **Jurisdiction risk:** FATF grey/blacklist lookup table
- **Evaluation:** Custom eval harness with precision, recall, F1, confusion matrix
- **Language:** Python 3.12

---


## Background

This project was built as a demonstration for Anthropic's Applied AI Architect 
(Startups) role in Dublin — not to pad a portfolio, but because building it 
was the most honest way to close the gap between what the role requires and 
what a resume can show.

The domain knowledge comes from two years as a Risk Consultant in Financial 
Risk Analytics at PwC, where I built Python and SQL pipelines to detect AML 
and FCPA violations across 1M+ financial records for clients across investment 
banking, retail, and infrastructure. The cases in this eval are not synthetic 
exercises — they are based on real typologies I triaged manually, which is 
precisely why I built the tool to automate the first pass.

The commercial instinct comes from co-founding and scaling Jyoti's Residences 
to 200+ customers across 10 Indian states — doing founder-led sales personally 
for the first 18 months, building the operational and pricing systems from 
scratch, and making autonomous decisions with no safety net. That experience 
is what the Applied AI Architect role describes as the qualifying background 
of a "technical founder who has done founder-led sales."

The project is scoped to demonstrate the four capabilities the role centres on: 
context engineering (system prompt design that produces reliable structured 
outputs), tool use architecture (agentic loop with live external tool calls), 
evaluation framework design (labelled dataset, precision/recall/F1, failure 
analysis), and cost-performance reasoning (Haiku vs Sonnet comparison with a 
concrete hybrid deployment recommendation).

Anthropic's bet — that the most important AI work happens at the seam of 
capability, safety, and real customer problems — is the bet I want to spend 
the next decade making with the startups building on Claude.
