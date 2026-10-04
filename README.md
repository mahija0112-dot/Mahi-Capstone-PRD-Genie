# PRD Genie

**An agentic AI documentation assistant that turns raw meeting transcripts, product briefs, and stakeholder notes into standardized PRDs, Epics, User Stories, and Gap Analyses.**

| | |
|---|---|
| **Company** | NeuronForge Technologies |
| **Author** | Mahija Sarma, Senior Product Manager (AI & FinTech) |
| **Program** | Applied Agentic AI for PMs/TPMs, Capstone Project |
| **Date** | September 2026 |

---

## Why PRD Genie

PMs and TPMs lose hours turning discussions into formal artifacts. PRD Genie targets four bottlenecks:

| Manual pain point | Agentic fix |
|---|---|
| Reading 1–2 hour transcripts line by line (3–5 hrs per PRD) | **Requirement Extractor Agent** |
| Every PM formats PRDs differently | **PRD Generator Agent** (single template) |
| Breaking PRDs into Epics and Stories takes 4–6 hrs | **Story Breakdown Agent** |
| Ambiguities surface mid-sprint as blockers | **Gap Analyzer Agent** |

## Goals and KPIs

- **Productivity:** cut PRD and story drafting time from ~8 hours to under 2 hours (75% reduction)
- **Quality:** 0% hallucinated features in approved PRDs, verified via source-traceability logs
- **Standardization:** 100% of product documentation on the master template (`prd_template.md`)
- **Adoption:** 85%+ of PMs/TPMs within 60 days of release

## How It Works

**Pattern:** Sequential pipeline with parallel branching. Inputs must be parsed before a PRD exists, and a PRD must exist before it can be split into stories. The extended agents run in parallel after core generation to reduce latency.

```
 Input (transcript / brief / notes)
              │
              ▼
   ┌─────────────────────────┐
   │ Requirement Extractor   │  gpt-4o-mini
   └───────────┬─────────────┘
               ▼
   ┌─────────────────────────┐
   │ PRD Generator           │  gpt-4o
   └───────────┬─────────────┘
       ┌───────┴────────┐
       ▼                ▼
 ┌───────────┐   ┌───────────────┐
 │ Story     │   │ Gap Analyzer  │
 │ Breakdown │   │ (extended)    │
 │ gpt-4o-mini│  │ gpt-4o        │
 └─────┬─────┘   └───────┬───────┘
       └────────┬────────┘
                ▼
     Human-in-the-loop PM review
                ▼
   Markdown / Doc export · Langfuse tracing
```

### Agents

| Agent | Model | Input → Output |
|---|---|---|
| **Requirement Extractor** | gpt-4o-mini | Raw text → JSON (`stated_requirements`, `nfr_technical`, `stakeholders`, `deadlines_milestones`, `dependencies`, `ambiguous_incomplete_items`, `status`) |
| **PRD Generator** | gpt-4o | Requirements JSON → Markdown PRD (Overview, Goals & Non-Goals, Personas, Functional Reqs, NFRs, Dependencies & Risks, Open Questions) |
| **Story Breakdown** | gpt-4o-mini | PRD → Epics, Features, and User Stories with priority tags and verbatim acceptance criteria |
| **Gap Analyzer** *(extended)* | gpt-4o | Requirements JSON + PRD → conflicts, missing specs, prioritized clarification questions |
| **Scope Estimator** *(extended)* | n/a | Shown in the architecture; no prompt or test coverage in this document |

### Guardrails baked into the prompts

1. **Grounding:** extract only what is explicitly stated; never invent or extrapolate.
2. **Verbatim metrics:** numbers like `10,000 users` or `p95 < 200ms` are never rounded or reworded.
3. **Fallback:** if nothing extractable exists, return `status: INSUFFICIENT_DATA` and halt downstream generation.
4. **Persona separation:** stories are partitioned per persona, never merged.
5. **Human-in-the-loop:** output is a draft. A PM must review and sign off before engineering estimation.

## Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Workflow / agents | **LangFlow** | Custom multi-agent chains, per-node prompt control, structured JSON hand-offs (chosen over n8n) |
| LLMs | **OpenAI gpt-4o-mini / gpt-4o** | Mini for high-volume, low-cost steps; 4o for PRD synthesis and gap analysis |
| Observability | **Langfuse** | Per-node traces, token/cost attribution, latency, via environment variables |
| Ingestion / export | **Python parsers + Markdown exporter** | Reads `.txt` / `.md`; outputs Markdown for GitHub, Notion, or Jira |

## Evaluation

All **12 baseline tests (T1–T12)** from `eval_prdgenie_inputs.txt` passed.

| ID | Scenario | What it verifies |
|---|---|---|
| T1 | Detailed transcript | 100% grounded extraction |
| T2 | Vague brief | `INSUFFICIENT_DATA`, 0 invented requirements |
| T3 | Contradictory requirements | 5s auto-refresh vs. "minimize API calls" flagged |
| T4 | Acceptance-criteria heavy | ACs preserved verbatim |
| T5 | Incomplete notes | `INSUFFICIENT_DATA`, no assumptions on TBDs |
| T6 | Multi-stakeholder tension | All viewpoints captured, timeline risk flagged |
| T7 | Technical NFRs | Metrics preserved exactly |
| T8 | Persona-heavy | 3 separate stories (Admin, End User, Auditor) |
| T9 | Empty input | Pipeline halts, no blank or fabricated PRD |
| T10 | Dependency and risk | High schedule risk flagged for unknown ETA |
| T11 | Full PRD generation | Output matches `prd_template.md` |
| T12 | Story breakdown | Epic and story structure with priority tag |

**Method:** deterministic, rules-based verification (string/token grounding checks, exact-match metric checks, schema validation). It is **not** LLM-as-a-judge. The document includes an optional Evaluator Agent prompt and sample `pytest`-style checks if you want to add semantic grading later.

**Key finding from tracing:** failures happened at the *inter-agent boundary*. Loose JSON from the Extractor caused the PRD Generator to infer context. Adding the `INSUFFICIENT_DATA` fallback and schema validation at extraction removed the drift.

## Observability and Cost

Sample Langfuse trace (T1): **3.42s** total, **2,150 tokens**, **$0.00041**.

| Metric | Value |
|---|---|
| Cost per pipeline run | ~$0.00041 |
| Cost per PM per day (10 runs) | ~$0.0041 |
| Cost per PM per month (22 working days) | ~$0.09 |
| 50 PMs, monthly | ~$4.51 |
| Langfuse | Free tier (up to 50k observations/month) |

### Production monitoring targets

- **Extraction completeness:** ≥ 98% of explicit requirements captured
- **Hallucination rate:** 0.0% of PRD items untraceable to source
- **PM time-saved ratio:** ≥ 75% reduction vs. the 8-hour baseline

## Rollout Plan (8 weeks)

| Weeks | Phase |
|---|---|
| 1–2 | Core pipeline architecture (LangFlow + gpt-4o-mini setup) |
| 3–4 | Verification and observability (Langfuse, T1–T12 testing) |
| 5–6 | Extended capabilities (Gap Analyzer, Scope Estimator) |
| 7–8 | Pilot with 5 PM teams and production hardening |

## Scope

**In scope:** ingestion of transcripts, notes, and briefs; explicit vs. ambiguous requirement separation; structured PRDs; Epics and User Stories; gap analysis; Langfuse observability.

**Out of scope (Phase 1):** calendar booking, direct code generation, autonomous sprint planning without human sign-off.

## Governance

| Role | Owner |
|---|---|
| Sponsor | VP of Product Management |
| Product Lead | Senior Product Manager (AI & FinTech) |
| Engineering Lead | AI Platform Lead (LangFlow / Infrastructure) |
| Primary users | PMs, TPMs, Business Analysts, Engineering Leads |

## Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Hallucinated requirements | Grounding prompts, `INSUFFICIENT_DATA` fallback, traceability logs |
| Silent scope drift (vague input becomes a firm feature) | Gap Analyzer, HITL sign-off |
| PMs over-relying on generated output | Mandatory human review before engineering estimation |
| False-alarm gaps | Confidence thresholding on gap identification |

## Roadmap Ideas

- **Hybrid RAG:** a Pinecone vector store of past PRDs and architecture guidelines
- **Fine-tuning:** train a gpt-4o-mini checkpoint on ~500 annotated transcript/requirement pairs to reduce reliance on gpt-4o
- **Integrations:** Jira / Linear sync

## Repository Contents

| Item | Description |
|---|---|
| `PRD_Genie_Design_and_other_documentation.docx` | Full capstone write-up: charter, architecture, prompts, test results, cost model, reflection |
| LangFlow workflow export (JSON) | Importable pipeline definition |
| `prd_template.md` | Master PRD template the generator must follow |
| `eval_prdgenie_inputs.txt` | Baseline dataset (T1–T12) |

## Getting Started

1. Import the LangFlow workflow JSON into your LangFlow instance.
2. Set your OpenAI API key and Langfuse environment variables (`LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`).
3. Run an input file (`.txt` / `.md`) through the pipeline.
4. Review the PRD, backlog, and gap report as a PM, then approve before handing off to engineering.

## Author

**Mahija Sarma**, Senior Product Manager (AI & FinTech)

