# AI Event Instrumentation Copilot

## Stakeholder Plan — Product Analytics, Inito

**Version 1.0 | May 2026**

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Solution Overview](#3-solution-overview)
4. [Technical Architecture](#4-technical-architecture)
5. [Foundation Work Completed](#5-foundation-work-completed)
6. [End-to-End Workflow](#6-end-to-end-workflow)
7. [Output Deliverables](#7-output-deliverables)
8. [Implementation Roadmap](#8-implementation-roadmap)
9. [Success Metrics](#9-success-metrics)
10. [Risks &amp; Mitigations](#10-risks--mitigations)
11. [Team &amp; Resource Requirements](#11-team--resource-requirements)
12. [Appendix](#12-appendix)

---

## 1. Executive Summary

Inito tracks user behaviour across its fertility tracking app using Amplitude analytics. Every new product feature requires a detailed **event instrumentation document** — a specification that maps every user action on every screen to a named analytics event, with properties, triggers, and quality checks. Today, this is done entirely by hand.

This document presents a plan to **automate event instrumentation** using a multi-agent AI pipeline powered by Claude. A product manager provides a PRD (Product Requirements Document); the system reads it, understands the feature, and produces a complete, review-ready instrumentation spec in minutes — a task that currently takes 2–4 hours per feature.

Beyond generation, the same system includes a **Cleanup Audit Tool** that continuously monitors the existing event taxonomy for duplicates, naming violations, and inconsistencies — preventing technical debt from accumulating in the analytics layer.

**The net outcome:** faster feature launches, higher data quality, and a scalable analytics infrastructure that grows with the product without growing the team's manual workload.

---

## 2. Problem Statement

### 2.1 The Current Process

Every time Inito ships a new feature, the product analytics team must produce an instrumentation spec from scratch. This involves:

1. Reading the PRD in detail to understand all screens, CTAs, and user flows
2. Identifying every trackable event (page loads, user actions, business completions)
3. Naming each event following strict PascalCase conventions (max 40 chars, CTA uppercase, etc.)
4. Checking 952 existing events to avoid creating duplicates
5. Assigning properties to each event (Source, Journey, boolean flags, etc.)
6. Validating against a 13-point checklist
7. Writing everything into a structured Excel handoff document for engineering

### 2.2 Pain Points

| Pain Point                                      | Impact                                                                                                                                                                           |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Time-intensive**                        | 2–4 hours per feature. At 3–5 features per sprint, this consumes 6–20 hours of analytics work per sprint just on documentation                                                |
| **Requires deep institutional knowledge** | The analyst must memorise 952 existing events, 1,043 properties, and a complex naming rulebook to avoid errors                                                                   |
| **Error-prone by nature**                 | Manual checks miss near-duplicate events (e.g.`BottomSheetCTACLicked` vs `BottomSheetCTAClicked` — a single-letter typo that now permanently splits user data in Amplitude) |
| **Bottleneck on feature launches**        | Engineers cannot start instrumentation until the spec is ready. Delays in analytics docs translate to delayed feature releases                                                   |
| **Taxonomy degradation over time**        | Without automated enforcement, the event taxonomy accumulates naming violations, inconsistencies, and semantic duplicates that make dashboards increasingly unreliable           |

### 2.3 The Scale of the Problem Today

An audit of Inito's current Amplitude taxonomy (952 events) reveals:

| Issue                                          | Count            |
| ---------------------------------------------- | ---------------- |
| Exact duplicate event pairs                    | 4 pairs          |
| Near-typo pairs (1–2 char difference)         | 14 pairs         |
| Events using dot-notation (old iOS SDK format) | 332              |
| Events with spaces (violate PascalCase)        | 317              |
| Events with underscores                        | 120              |
| Events exceeding 40-character limit            | 159              |
| Incorrect CTA capitalisation                   | 5                |
| Load suffix inconsistencies                    | 81               |
| Potential semantic duplicates                  | 14 pairs         |
| **Total issues identified**              | **1,063+** |

These aren't just cosmetic issues — duplicate events split user data across two event names, making funnel analysis unreliable. Naming violations make event discovery harder, slowing down dashboard and experiment setup.

---

## 3. Solution Overview

### 3.1 The AI Instrumentation Copilot

The AI Instrumentation Copilot is a **Python-based CLI tool** that takes a PRD as input and produces a complete, validated instrumentation spec as output — in under 5 minutes.

```
Input:  PRD text file (.txt / .md)
        Optional: design notes, existing events KB, existing properties KB

Output: Excel workbook (.xlsx) with 6 structured tabs
        Ready for engineering handoff and Amplitude setup
```

Under the hood, it runs a **6-agent pipeline** where each agent is powered by Claude Sonnet (claude-sonnet-4-6) and handles a specific stage of the instrumentation process.

### 3.2 The Cleanup Audit Tool

A companion tool that reads the existing event taxonomy and produces a colour-coded **cleanup report** — identifying duplicates, naming violations, semantic inconsistencies, and a full migration map with suggested fixes.

```
Input:  Events CSV (the full taxonomy)

Output: Excel workbook with 6 audit tabs
        Prioritised fix list + old → new name migration map
```

### 3.3 Core Capabilities

| Capability                      | Description                                                                                   |
| ------------------------------- | --------------------------------------------------------------------------------------------- |
| **Feature Understanding** | Parses a PRD to extract every screen, CTA, user flow, error state, and business KPI           |
| **Event Generation**      | Generates correctly-named candidate events for every trackable user action                    |
| **Duplicate Prevention**  | Checks all 952 existing events before creating new ones — reuses or extends where possible   |
| **Property Assignment**   | Assigns the right event and user properties to each event, reusing the 1,043-property library |
| **Validation**            | Runs a 13-point quality checklist, flags issues, and surfaces PM questions                    |
| **Dashboard Suggestions** | Recommends Amplitude charts and funnels aligned to the feature's KPIs                         |
| **Taxonomy Audit**        | Detects duplicates, naming violations, and semantic inconsistencies across the full taxonomy  |

---

## 4. Technical Architecture

### 4.1 The 6-Agent Pipeline

The copilot runs six specialised AI agents in sequence. Each agent has a single responsibility, receives the output of the previous agent, and passes its result forward.

```
PRD Text
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  AGENT A — Feature Understanding                            │
│  Parses PRD → extracts screens, CTAs, flows, KPIs           │
│  Output: feature_atoms JSON                                  │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  AGENT B — Candidate Event Generation                       │
│  Generates all candidate events following naming rules      │
│  Output: candidate_events list (over-generated is fine)     │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  AGENT C — Existing Event Mapping                           │
│  Compares candidates against 952 existing events            │
│  Decision: REUSE / EXTEND / NEW for each candidate          │
│  Output: mapped_events list                                  │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  AGENT D — Property Resolution                              │
│  Assigns properties from 1,043-property library             │
│  Identifies net-new properties needed                        │
│  Output: property_mapped_events list                         │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  AGENT E — Validation & Gap Detection                        │
│  Runs 13-point checklist on all events                       │
│  Flags naming violations, missing KPIs, PM questions         │
│  Output: validation_result with review_notes                 │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  AGENT F — Output Formatting                                │
│  Structures everything for the Excel handoff document        │
│  Adds dashboard suggestions aligned to feature KPIs         │
│  Output: formatted dict → 6-tab Excel file                  │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
                     Excel Output (.xlsx)
```

### 4.2 Agent Responsibilities

| Agent                          | Role                   | Key Input                          | Key Output                               |
| ------------------------------ | ---------------------- | ---------------------------------- | ---------------------------------------- |
| **A: Feature Parser**    | Understand the feature | PRD text                           | Screens, CTAs, flows, KPIs, error states |
| **B: Event Generator**   | Propose all events     | Feature atoms                      | Candidate event list with triggers       |
| **C: Event Mapper**      | Prevent duplication    | Candidates + existing 952 events   | REUSE / EXTEND / NEW decisions           |
| **D: Property Resolver** | Assign properties      | Events + existing 1,043 properties | Full event→property mapping             |
| **E: Validator**         | Quality gate           | Mapped events + KPIs               | Issues, flags, PM questions              |
| **F: Formatter**         | Package output         | All above                          | Structured Excel-ready data              |

### 4.3 The Knowledge Base

The Knowledge Base is the system's memory — it tells the AI what already exists so it doesn't create redundant events or properties.

| KB Component                                    | Contents                                                              | Format     |
| ----------------------------------------------- | --------------------------------------------------------------------- | ---------- |
| **Events KB** (`kb_events.csv`)         | 952 events with event_type, feature_group, trigger, properties        | CSV / JSON |
| **Properties KB** (`kb_properties.csv`) | 1,043 unique properties with type, enum values, which events use them | CSV / JSON |

The KBs are passed as optional inputs to the pipeline:

```bash
python main.py --prd feature.txt \
  --events kb_events.csv \
  --properties kb_properties.csv
```

Without the KBs, the pipeline still works but cannot check for existing duplicates.

### 4.4 Technology Stack

| Component     | Technology                                          |
| ------------- | --------------------------------------------------- |
| AI Model      | Claude Sonnet (claude-sonnet-4-6) via Anthropic API |
| Language      | Python 3.10+                                        |
| Output Format | Excel (openpyxl)                                    |
| Input Formats | Plain text PRD (.txt, .md), CSV/JSON for KBs        |
| CLI Interface | argparse (standard Python)                          |
| Dependencies  | `anthropic`, `openpyxl`                         |

### 4.5 Inito-Specific Naming Rules (Hardcoded as Constraints)

All Inito's naming conventions are **hardcoded into the system** via `config.py` — agents cannot violate them:

| Rule               | Specification                                                         |
| ------------------ | --------------------------------------------------------------------- |
| Case               | PascalCase only                                                       |
| Length             | Maximum 40 characters                                                 |
| CTA                | Must be uppercase (`CTA`, never `Cta` or `cta`)                 |
| Event types        | Exactly 3: PageLoad, UserAction, ProcessCompletion                    |
| PageLoad suffix    | `PageLoaded` (not `Loaded` or `ScreenLoaded`)                   |
| Boolean properties | Must start with `Is` (e.g. `IsFirstTest`, `IsDismissible`)      |
| Source property    | Mandatory on all standalone UserAction/PageLoad events                |
| Journey property   | Mandatory on all events inside a multi-step flow                      |
| Forbidden generics | `BottomSheetLoaded` alone (must specify which), `EmailCTAClicked` |
| Numeric values     | Integer/Float type — never stored as string                          |

---

## 5. Foundation Work Completed

Before the copilot can run at full quality, it needs a rich, accurate Knowledge Base. The following foundational work has been completed:

### 5.1 Event Taxonomy Extraction

**What we did:** Used Amplitude's MCP API to fetch properties for all 952 events in Inito's taxonomy. This was done in parallel batches of 40 events to handle rate limits and session timeouts.

**Results:**

| Metric                                         | Value      |
| ---------------------------------------------- | ---------- |
| Total events in taxonomy                       | 952        |
| Events with properties extracted               | 952 (100%) |
| Events that have properties defined            | 731        |
| Events with no taxonomy properties (Amplitude) | 221        |
| Unique properties discovered                   | 1,043      |

**Output files produced:**

| File                                  | Purpose                                                           |
| ------------------------------------- | ----------------------------------------------------------------- |
| `kb_events.csv`                     | Complete events KB — drop-in input for `--events` flag         |
| `kb_properties.csv`                 | Complete properties KB — drop-in input for `--properties` flag |
| `Inito Events Total - Updated.xlsx` | Updated master Excel with properties column filled                |

### 5.2 Taxonomy Cleanup Audit

**What we did:** Built and ran a cleanup audit tool that analyses the full taxonomy for quality issues — both with fast rule-based checks and with LLM-powered semantic duplicate detection.

**Findings from `cleanup_report.xlsx`:**

| Issue Category                       | Count            | Severity    | Action                            |
| ------------------------------------ | ---------------- | ----------- | --------------------------------- |
| Exact duplicate event pairs          | 4 pairs          | 🔴 Critical | Merge immediately                 |
| Near-typo pairs (edit distance ≤ 2) | 14 pairs         | 🔴 Critical | Review and merge                  |
| Dot-notation events (old iOS SDK)    | 332              | 🟠 High     | Disable in Amplitude SDK settings |
| Events with spaces                   | 317              | 🟠 High     | Rename or disable                 |
| ALL_CAPS legacy events               | 20               | 🟠 High     | Rename to PascalCase              |
| Events with underscores              | 100              | 🟠 High     | Rename to PascalCase              |
| Events starting with lowercase       | 31               | 🟠 High     | Rename                            |
| Incorrect CTA capitalisation         | 5                | 🟡 Medium   | Rename                            |
| Events exceeding 40-char limit       | 159              | 🟡 Medium   | Shorten                           |
| Load suffix inconsistencies          | 81               | 🟢 Low      | Standardise to PageLoaded         |
| Semantic duplicate pairs (LLM)       | 14 pairs         | 🟡 Medium   | PM review required                |
| **Total**                      | **1,063+** |             |                                   |

**Key semantic duplicates identified (High confidence — requires immediate review):**

| Event A                                | Event B                           | Issue                                       |
| -------------------------------------- | --------------------------------- | ------------------------------------------- |
| `EditPeriodLoaded`                   | `EditPeriodPageLoaded`          | Same screen, different suffix — data split |
| `OnboardingCompletedLoaded`          | `OnboardingCompletedPageLoaded` | Same screen, different suffix — data split |
| `BBTMeauringInstructionCTAClicked`   | `BBTMeasuringScreenCTAClicked`  | Typo + semantic overlap                     |
| `BBTMeauringInstructionScreenLoaded` | `BBTMeasuringScreenLoaded`      | Typo + semantic overlap                     |
| `NGDevicePageLoaded`                 | `NGDeviceOnPageLoaded`          | Ambiguous "On" — likely same event         |
| `NGDeviceCTAClicked`                 | `NGDeviceOnCTAClicked`          | Ambiguous "On" — likely same event         |

The `cleanup_report.xlsx` contains a **MigrationMap tab** with 589 rows — a complete `old_name → suggested_new_name` table ready for the team to review and implement.

---

## 6. End-to-End Workflow

### 6.1 New Feature Instrumentation (Copilot Flow)

```
PM writes PRD
      │
      ▼
Analyst runs: python main.py --prd feature.txt
                              --events kb_events.csv
                              --properties kb_properties.csv
      │
      ▼ (~3-5 minutes)
Excel output: feature_instrumentation.xlsx
      │
      ├── Tab 1: EventInstrumentation  ← review event names & triggers
      ├── Tab 2: EventProperties       ← review property assignments
      ├── Tab 3: UserProperties        ← review user attributes
      ├── Tab 4: NetNewProperties      ← approve any new properties before adding
      ├── Tab 5: ReviewNotes           ← address PM questions & flagged issues
      └── Tab 6: SuggestedDashboards   ← optional: create Amplitude charts
      │
      ▼
Analyst reviews & edits (30–60 minutes vs. 2–4 hours today)
      │
      ▼
Engineering handoff → Amplitude taxonomy setup
```

### 6.2 Taxonomy Maintenance (Cleanup Flow)

```
Periodic run (per sprint / per release):
      │
python cleanup_audit.py --events kb_events.csv
      │
      ▼ (~30 seconds)
cleanup_report.xlsx
      │
      ├── ExactDuplicates    ← fix before next release
      ├── NamingViolations   ← backlog for taxonomy cleanup sprint
      ├── SemanticDuplicates ← PM review → merge if confirmed
      └── MigrationMap       ← implementation reference for engineers
```

### 6.3 How the Copilot Handles Edge Cases

| Scenario                                                      | How the System Handles It                                                                  |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| New feature introduces a screen similar to an existing one    | Agent C detects EXTEND — adds a differentiating property rather than creating a duplicate |
| PRD is ambiguous about a trigger                              | Agent E flags it as `AmbiguousTrigger` in ReviewNotes, surfacing it for PM decision      |
| A CTA appears on multiple screens                             | Agent D assigns `ScreenDetail` property to disambiguate                                  |
| Feature has no design notes                                   | Agent A infers screen structure from PRD alone                                             |
| PRD success metric has no corresponding event                 | Agent E flags `MissingKPI` — the PM must decide how to instrument it                    |
| High-frequency action (scroll, date picker tap) is in the PRD | Agent B skips it and notes it as `HighFrequencyRisk`                                     |
| Same flow, different product variant (NG vs OG reader)        | Agent D adds a `ReaderType` property rather than creating a new event                    |

---

## 7. Output Deliverables

### 7.1 Instrumentation Spec (per feature)

The main output of the copilot — a 6-tab Excel file designed for direct engineering handoff.

| Tab                            | Purpose                                  | Key Columns                                                                         |
| ------------------------------ | ---------------------------------------- | ----------------------------------------------------------------------------------- |
| **EventInstrumentation** | Master event list                        | Event Name, Type, Category, Trigger, Screen, Status (New/Reuse/Extend), Related KPI |
| **EventProperties**      | Per-event property mapping               | Event Name, Property, Type, Enum Values, Existing?, Mandatory?                      |
| **UserProperties**       | User-level attribute references          | Event Name, User Property, Existing?                                                |
| **NetNewProperties**     | New properties needing taxonomy addition | Property Name, Type, Description, First Used In                                     |
| **ReviewNotes**          | Quality flags and PM questions           | Type, Event, Note, Action Required                                                  |
| **SuggestedDashboards**  | Amplitude chart recommendations          | Dashboard Name, Chart Type, Events Used, Purpose                                    |

**Colour coding:** New events = green, Reused events = blue, Extended events = yellow, Flagged = red.

### 7.2 Cleanup Audit Report (taxonomy-wide)

| Tab                             | Purpose                                              |
| ------------------------------- | ---------------------------------------------------- |
| **Summary**               | Issue counts by severity — quick health snapshot    |
| **ExactDuplicates**       | Must-fix pairs — merge these to stop data splitting |
| **NamingViolations**      | All structural violations with suggested renames     |
| **SuffixInconsistencies** | Standardise to `PageLoaded`                        |
| **SemanticDuplicates**    | LLM-identified pairs for PM review                   |
| **MigrationMap**          | Complete old → new mapping for implementation       |

### 7.3 Knowledge Base Files (reusable across all features)

| File                  | Description                                                  |
| --------------------- | ------------------------------------------------------------ |
| `kb_events.csv`     | 952 events with properties — used by `--events` flag      |
| `kb_properties.csv` | 1,043 properties with usage — used by `--properties` flag |

---

## 8. Implementation Roadmap

### Phase 1 — Foundation (✅ Completed)

**Goal:** Build the infrastructure needed for the copilot to function at production quality.

| Task                                                                    | Status  |
| ----------------------------------------------------------------------- | ------- |
| Build the 6-agent pipeline (Agents A–F)                                | ✅ Done |
| Build Excel output writer (6 tabs)                                      | ✅ Done |
| Extract all 952 events from Amplitude taxonomy via MCP                  | ✅ Done |
| Build Knowledge Base loaders (CSV + JSON)                               | ✅ Done |
| Generate `kb_events.csv` and `kb_properties.csv` (1,043 properties) | ✅ Done |
| Build the Cleanup Audit Tool                                            | ✅ Done |
| Run taxonomy audit — identify 1,063+ issues                            | ✅ Done |
| Identify 14 semantic duplicate pairs with LLM                           | ✅ Done |
| Produce `cleanup_report.xlsx` (589-row migration map)                 | ✅ Done |

**Deliverable:** A fully functional copilot + a complete, accurate Knowledge Base.

---

### Phase 2 — Taxonomy Cleanup (🔜 Next: 2–3 Weeks)

**Goal:** Resolve the critical issues identified in the audit report before the copilot goes into regular use. A clean taxonomy means higher quality copilot output.

**Priority 1 — Critical (must fix before copilot goes live):**

| Action                              | Scope                                                              | Effort  |
| ----------------------------------- | ------------------------------------------------------------------ | ------- |
| Merge 4 exact duplicate event pairs | 4 events to deprecate                                              | 1 hour  |
| Merge/resolve 14 near-typo pairs    | Review each pair, keep canonical                                   | 2 hours |
| Fix 5 CTA capitalisation errors     | Rename `EditSymptomCtaClicked` → `EditSymptomCTAClicked` etc. | 30 min  |

**Priority 2 — High (complete within 2 weeks):**

| Action                                                        | Scope                                                        | Effort     |
| ------------------------------------------------------------- | ------------------------------------------------------------ | ---------- |
| Review and confirm 6 high-confidence semantic duplicate pairs | PM decision required                                         | 2 hours    |
| Disable or rename dot-notation events (332)                   | Mostly old iOS SDK auto-events — disable in SDK settings    | 1 day      |
| Standardise 81 suffix inconsistencies to `PageLoaded`       | Rename `EditPeriodLoaded` → `EditPeriodPageLoaded` etc. | 3–4 hours |

**Priority 3 — Medium (cleanup sprint):**

| Action                                     | Scope                                                        | Effort         |
| ------------------------------------------ | ------------------------------------------------------------ | -------------- |
| Rename 317 space-containing events         | Many are Amplitude SDK defaults (disable)                    | 2–3 days      |
| Rename 120 underscore events to PascalCase | Use MigrationMap tab for suggested names                     | 1 day          |
| Shorten 159 events over 40 chars           | Focus on the 332 dot-notation events (many are too long too) | Included above |

**Tooling:** All of this work is driven by `cleanup_report.xlsx` — specifically the `MigrationMap` tab which has `old_name → suggested_new_name` for every issue.

**Deliverable:** A clean, consistent taxonomy; updated `kb_events.csv` reflecting the clean state.

---

### Phase 3 — Pilot Run (🔜 Weeks 3–4)

**Goal:** Run the copilot on 2–3 real upcoming features. Validate output quality. Measure time savings.

| Task                                          | Owner     | Notes                                                                                      |
| --------------------------------------------- | --------- | ------------------------------------------------------------------------------------------ |
| Select 2–3 upcoming PRDs as pilot features   | PM        | Choose features with clear screens and KPIs                                                |
| Run copilot on each PRD                       | Analytics | `python main.py --prd feature.txt --events kb_events.csv --properties kb_properties.csv` |
| Compare AI output vs manual spec              | Analytics | Measure: accuracy, completeness, time taken                                                |
| Identify gaps in AI output                    | Analytics | Feed back into agent prompts and config                                                    |
| Refine system prompts based on pilot feedback | Analytics | Iterate on Agent A and B prompts especially                                                |

**Success criteria for pilot:**

- Copilot output requires ≤ 60 minutes of analyst review/editing (vs. 2–4 hours manually)
- All events in output are correctly named (PascalCase, ≤ 40 chars, CTA uppercase)
- No existing event is recreated as a new event when REUSE/EXTEND was appropriate
- All PRD KPIs have at least one corresponding ProcessCompletion event

**Deliverable:** Validated copilot output on 2–3 real features; refined agent prompts.

---

### Phase 4 — Integration & Process Change (Weeks 5–8)

**Goal:** Make the copilot a standard part of the feature delivery process.

| Task                                   | Description                                                                                                                                         |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Process documentation**        | Write the new instrumentation workflow — when to run the copilot, how to review output, how to handle ReviewNotes flags                            |
| **KB maintenance SOP**           | Define how `kb_events.csv` and `kb_properties.csv` are updated when new events are shipped (after each sprint)                                  |
| **Engineering handoff template** | Standardise how the copilot's Excel output is shared with engineering — e.g. as a Notion page, Jira attachment, or direct Amplitude taxonomy entry |
| **Amplitude taxonomy sync**      | Define a process for keeping the KB in sync with actual Amplitude state (quarterly re-extraction, or on-demand after major releases)                |
| **Periodic cleanup cadence**     | Schedule a quarterly taxonomy cleanup run using `cleanup_audit.py` — review MigrationMap, address Critical issues before they accumulate         |

**Deliverable:** Documented, repeatable process. Copilot is standard tooling for every feature from this point forward.

---

### Phase 5 — Enhancements (Month 3+)

These are future improvements to increase automation depth:

| Enhancement                                | Description                                                                                                                         | Value                                                 |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| **Direct Amplitude taxonomy writes** | Instead of producing an Excel spec for manual entry, use Amplitude's API to directly create events and properties in a draft branch | Eliminates the manual data-entry step entirely        |
| **PRD auto-detection**               | Watch a Notion/Jira/Google Drive folder; auto-run the copilot when a new PRD is created                                             | Removes the manual trigger step                       |
| **Dashboard auto-creation**          | Use Amplitude MCP to auto-create the charts from the `SuggestedDashboards` tab                                                    | Analytics dashboards ready the moment a feature ships |
| **Post-launch validation**           | After a feature goes live, automatically check whether the expected events are firing in Amplitude                                  | Closes the loop from spec to actual data              |
| **Slack/Notion integration**         | Post the ReviewNotes flags to a Slack channel or Notion page for PM review, instead of opening an Excel file                        | Faster review turnaround                              |

---

## 9. Success Metrics

### 9.1 Efficiency Metrics

| Metric                                     | Baseline (Today) | Target (Phase 3) | Target (Phase 4+) |
| ------------------------------------------ | ---------------- | ---------------- | ----------------- |
| Time to produce instrumentation spec       | 2–4 hours       | ≤ 90 minutes    | ≤ 60 minutes     |
| Analytics analyst hours per feature        | 2–4 hours       | 1–1.5 hours     | 45–60 minutes    |
| Time from PRD finalisation to spec handoff | 1–2 days        | Same day         | Same day          |
| Features instrumented per sprint           | 3–5             | 6–8             | 8–12             |

### 9.2 Quality Metrics

| Metric                                           | Baseline (Today)        | Target              |
| ------------------------------------------------ | ----------------------- | ------------------- |
| Events passing 13-point validation on first pass | ~60% (estimated)        | ≥ 90%              |
| Duplicate events created per quarter             | Unknown (no monitoring) | 0                   |
| Events over 40 chars added per quarter           | ~5–10                  | 0                   |
| CTA naming violations per quarter                | ~2–3                   | 0                   |
| Taxonomy issues (audit)                          | 1,063 (current state)   | < 100 after Phase 2 |

### 9.3 Coverage Metrics

| Metric                                | Current                                                       | Target                             |
| ------------------------------------- | ------------------------------------------------------------- | ---------------------------------- |
| Events in KB with properties defined  | 731 / 952 (77%)                                               | 95%+                               |
| Unique properties in KB               | 1,043                                                         | Kept current (updated each sprint) |
| Feature categories covered by copilot | All (Testing, Shop, Onboarding, Medication, Charts, Settings) | All                                |

---

## 10. Risks & Mitigations

| Risk                                                   | Likelihood             | Impact | Mitigation                                                                                                                                                       |
| ------------------------------------------------------ | ---------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **AI generates incorrect event names**           | Medium                 | Medium | Agent E's 13-point validator + analyst review step before handoff. System is an accelerator, not a replacement for review.                                       |
| **KB becomes stale**                             | High (without process) | High   | Define a KB update SOP (Phase 4). Run monthly re-extraction. Stale KB means Agent C misses reuse opportunities.                                                  |
| **PRD quality varies**                           | High                   | Medium | Agent A is designed to infer from incomplete PRDs. Agent E surfaces ambiguities as PM questions in ReviewNotes. Poor PRD = more review flags, not broken output. |
| **Amplitude API rate limits**                    | Medium                 | Low    | Already handled via batched extraction with delay. Only affects KB refresh, not the copilot itself.                                                              |
| **Engineering doesn't use the spec**             | Low                    | High   | Standardise handoff format in Phase 4 process docs. Make it easy to consume (clear Excel format, Notion/Jira integration).                                       |
| **Over-reliance on AI — reduced analyst skill** | Low                    | Medium | Keep analyst in the loop as the mandatory reviewer. The copilot drafts; the analyst decides.                                                                     |
| **Taxonomy cleanup breaks existing charts**      | Medium                 | High   | Use Amplitude's branch feature to test renames before applying. Clean up in batches with engineering review.                                                     |

---

## 11. Team & Resource Requirements

### 11.1 People

| Role                                  | Involvement                                                     | Phase              |
| ------------------------------------- | --------------------------------------------------------------- | ------------------ |
| **Product Analytics (Lead)**    | Tool owner, runs the copilot, reviews output, maintains KB      | All phases         |
| **Product Manager**             | Provides PRDs, reviews ReviewNotes flags, approves PM questions | Phase 3+ (ongoing) |
| **Engineering (iOS/Android)**   | Receives spec, implements events, confirms fires in Amplitude   | Phase 3+           |
| **Data Engineering (optional)** | Helps with Amplitude taxonomy API integration in Phase 5        | Phase 5 only       |

### 11.2 Infrastructure

| Requirement                 | Notes                                                                                               |
| --------------------------- | --------------------------------------------------------------------------------------------------- |
| **Anthropic API key** | Required to run the copilot. Claude Sonnet (claude-sonnet-4-6) model.                               |
| **Python 3.10+**      | Standard requirement. Runs locally or on any server.                                                |
| **Amplitude account** | Existing Inito account (appId 407712). Used for KB extraction and optional Phase 5 taxonomy writes. |
| **File storage**      | KB files (`kb_events.csv`, `kb_properties.csv`) stored in project directory. ~200KB.            |

### 11.3 Estimated Costs (Anthropic API)

The copilot makes 6 Claude API calls per feature (one per agent). Each call is ~2,000–8,000 tokens.

| Per Feature                       | Estimate               |
| --------------------------------- | ---------------------- |
| Input tokens per pipeline run     | ~30,000–50,000 tokens |
| Output tokens per pipeline run    | ~5,000–10,000 tokens  |
| Cost per run (claude-sonnet-4-6)  | ~$0.10–$0.20 USD      |
| Monthly cost at 20 features/month | ~$2–4 USD             |

The cleanup audit tool runs once per sprint: ~$0.50–$1.00 per run.

**Total estimated monthly API cost: < $10 USD.**

---

## 12. Appendix

### 12.1 CLI Reference

**Instrumentation Copilot:**

```bash
# Basic usage
python main.py --prd feature_name.txt

# Full usage with Knowledge Base
python main.py \
  --prd feature_name.txt \
  --design design_notes.txt \
  --events kb_events.csv \
  --properties kb_properties.csv \
  --output feature_instrumentation.xlsx

# Debug mode (saves intermediate JSON)
python main.py --prd feature_name.txt --debug
```

**Cleanup Audit Tool:**

```bash
# Fast mode (rule-based only, no API key needed, ~30 seconds)
python cleanup_audit.py --events kb_events.csv --skip-semantic

# Full mode (includes LLM semantic duplicate detection, ~5 minutes)
python cleanup_audit.py --events kb_events.csv --output cleanup_report.xlsx
```

### 12.2 File Structure

```
Event Instrumentation/
├── main.py                          ← Copilot CLI entry point
├── cleanup_audit.py                 ← Cleanup audit CLI entry point
├── kb_events.csv                    ← 952 events Knowledge Base (use with --events)
├── kb_properties.csv                ← 1,043 properties Knowledge Base (use with --properties)
├── cleanup_report.xlsx              ← Latest taxonomy audit report
├── Inito Events Total - Updated.xlsx ← Master event registry (updated with properties)
│
├── instrumentation_copilot/
│   ├── config.py                    ← Naming rules, event types, model config
│   ├── orchestrator.py              ← Pipeline runner (A→F)
│   ├── agents/
│   │   ├── a_feature_parser.py      ← Agent A: PRD understanding
│   │   ├── b_event_generator.py     ← Agent B: Event generation
│   │   ├── c_event_mapper.py        ← Agent C: REUSE/EXTEND/NEW
│   │   ├── d_property_resolver.py   ← Agent D: Property assignment
│   │   ├── e_validator.py           ← Agent E: 13-point validation
│   │   └── f_formatter.py           ← Agent F: Excel formatting
│   ├── cleanup/
│   │   ├── rule_auditor.py          ← 10-check rule-based auditor
│   │   ├── semantic_auditor.py      ← LLM semantic duplicate detector
│   │   └── report_writer.py         ← 6-tab cleanup Excel writer
│   ├── knowledge/
│   │   └── kb.py                    ← KnowledgeBase class (load, search, retrieve)
│   └── output/
│       └── excel_writer.py          ← Instrumentation Excel writer
│
└── sample_inputs/
    ├── sample_prd.txt               ← Example PRD (Medication Reminder v2)
    ├── sample_existing_events.csv   ← Example events KB format
    └── sample_existing_properties.csv ← Example properties KB format
```

### 12.3 The 13-Point Validation Checklist

Every event generated by the copilot is validated against these rules (Agent E):

1. PascalCase enforced throughout
2. Event name ≤ 40 characters
3. `CTA` is uppercase in event name (not `Cta` or `cta`)
4. Descriptive name — no generic suffixes like `BottomSheetLoaded` alone
5. Trigger condition written and unambiguous
6. `Source` property present on all standalone UserAction/PageLoad events
7. `Journey` property present on all events inside a multi-step flow
8. Boolean properties start with `Is` prefix
9. No high-frequency action events (date pickers, scroll, repetitive taps)
10. Each ProcessCompletion event is tied to a specific PRD KPI
11. No duplicate events in the generated batch
12. Error states are covered (as UserAction or ProcessCompletion with appropriate properties)
13. KPI coverage verified — all PRD success metrics have at least one corresponding event

### 12.4 ReviewNotes Flag Types

When the validator detects an issue it cannot resolve automatically, it creates a ReviewNote:

| Flag Type             | Meaning                                                | Who Resolves            |
| --------------------- | ------------------------------------------------------ | ----------------------- |
| `NamingViolation`   | Event name breaks a naming rule                        | Analytics               |
| `DuplicateRisk`     | Event may overlap with an existing one                 | Analytics               |
| `MissingKPI`        | A PRD success metric has no corresponding event        | PM + Analytics          |
| `MissingSource`     | A standalone event is missing the `Source` property  | Analytics               |
| `AmbiguousTrigger`  | The trigger condition is unclear from the PRD          | PM                      |
| `HighFrequencyRisk` | An event may fire too frequently (scroll, date picker) | Engineering + Analytics |
| `PMQuestion`        | A business decision is needed to complete the spec     | PM                      |

---

*Document prepared by: Inito Product Analytics*
*Contact: product.analytics@inito.com*
