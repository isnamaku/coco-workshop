---
name: prd-to-pipeline-plan
description: "Turn a PRD (product requirements document) into an implementation plan for a Snowflake Dynamic Table. Use when: onboarding new source systems into an existing pipeline, updating a Silver/Gold layer from business requirements, translating a requirements spreadsheet into DT changes. Triggers: PRD, business requirements, source onboarding, pipeline plan, requirements to pipeline, onboard new source, update pipeline from requirements."
---

# PRD to Pipeline Plan

Convert a PRD file into a structured implementation plan for a target Snowflake Dynamic Table. The skill reads requirements, inspects the current DT definition, diffs the two, and produces a plan that separates confirmed changes from open questions -- never guessing where the PRD is ambiguous.

## Inputs

The user must provide two things:

| Input | Example | Notes |
|-------|---------|-------|
| `prd_path` | `./requirements.xlsx` | Local file. Supported: XLSX (multiple sheets), CSV. The file may contain source onboarding requests, column mappings, and/or business rules -- in separate sheets/files or combined. |
| `target_dynamic_table` | `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES` | Fully qualified name of the existing Dynamic Table to update. |

If either input is missing, ask for it before proceeding.

## Workflow

### Step 1: Read and Parse the PRD

1. Read the file at `prd_path`.
   - **XLSX**: Read each sheet. Common sheet patterns: "Source Onboarding", "Column Mapping", "Business Rules". If sheet names differ, infer purpose from column headers.
   - **CSV**: If a single CSV, infer whether it contains source onboarding, column mappings, or business rules from its headers. The user may provide multiple CSVs -- read all of them.
2. Extract and categorize every requirement into one of three buckets:
   - **Source onboarding**: new systems, regions, contacts, go-live dates, data delivery mechanisms
   - **Column mapping**: source-to-target field mapping, type conversions, transforms
   - **Business rules**: normalization logic, dedup rules, filtering, alerting, refresh config

**⚠️ STOP**: Present a summary of what was parsed -- number of sources found, number of mapping rows, number of business rules. Confirm with user before proceeding.

### Step 2: Inspect the Target Dynamic Table

1. Get the current DT definition:
   ```sql
   SELECT GET_DDL('DYNAMIC_TABLE', '<target_dynamic_table>');
   ```
2. Get current column list and types:
   ```sql
   DESCRIBE DYNAMIC TABLE <target_dynamic_table>;
   ```
3. Get current state (refresh mode, target lag, scheduling state):
   ```sql
   SHOW DYNAMIC TABLES LIKE '<table_name>' IN SCHEMA <db>.<schema>;
   ```
4. Sample a few rows to understand current data patterns:
   ```sql
   SELECT * FROM <target_dynamic_table> LIMIT 5;
   ```

### Step 3: Diff PRD Against Current State

Compare every PRD requirement against the current DT and classify each into:

| Classification | Meaning | Action |
|----------------|---------|--------|
| **New** | Not in current DT at all | Requires new UNION ALL branch, new column, or new transform |
| **Changed** | Exists but PRD specifies a different behavior | Requires modifying existing logic |
| **Confirmed** | Already implemented correctly | No action needed |
| **Contradicted** | PRD conflicts with current implementation | Flag as open question |

### Step 4: Surface Assumptions and Open Questions

This is the most important step. For every requirement that is not unambiguously confirmed:

**Assumptions** -- things the plan assumes to be true based on the PRD but that are not explicitly stated. Always list these. Examples:
- "PRD does not mention change tracking on new source tables; plan assumes it must be enabled."
- "PRD says 'direct map' for BAN_AMOUNT; plan assumes the column arrives as NUMBER, not TEXT."

**Open questions** -- things the PRD explicitly marks as undecided, or where two rules contradict, or where the PRD is silent on something the DT needs. Never resolve these by guessing. Examples:
- "BR-005 says payment terms normalization needs a decision. Plan cannot proceed on this field without a confirmed rule."
- "Baan cost center format changed in Jan 2025. PRD does not specify whether to normalize old format."

**Rules for surfacing vs. assuming:**
- If the PRD says "NEEDS DECISION", "TBD", "open question", or similar -- it is an **open question**. Never default it.
- If the PRD is silent on a topic that affects the DT (e.g., dedup strategy for a new source) -- it is an **open question**.
- If two rules in the PRD contradict each other or contradict the current DT -- it is an **open question**.
- If the PRD clearly states a rule with an owner and a "confirmed" status -- it is a **confirmed change**. Proceed.
- If you must make a reasonable inference to connect a PRD requirement to a DT change (e.g., enabling change tracking) -- it is an **assumption**. State it.

### Step 5: Present the Plan

Always return exactly these five sections in this order:

#### Section 1: New Source Systems
For each new source:
- Source system name and ERP platform
- Data delivery mechanism and frequency
- Region and go-live target
- Any blockers (legal, technical, access)

#### Section 2: Column Mapping Changes
A table with columns: `Silver Column | Current Sources | New Source(s) | Transform | Status`

Where Status is one of: `Confirmed`, `Changed`, `New Column`, `Dropped`, `Open Question`

#### Section 3: Business Rules Impact
For each rule that affects the DT:
- Rule ID and category
- What changes in the DT (specific SQL-level change)
- Whether it applies to existing branches, new branches, or both
- Status: `Implement`, `Already Implemented`, `Out of Scope (not Silver)`, `Open Question`

#### Section 4: Assumptions
Numbered list. Each assumption includes:
- What is assumed
- Why (what PRD evidence supports it)
- Risk if the assumption is wrong

#### Section 5: Open Questions
Numbered list. Each open question includes:
- The question
- Why it blocks implementation
- Who owns the decision (if the PRD says)
- What the current recommendation is (if the PRD says)

**⚠️ STOP**: Present the full plan. Do not generate DDL until the user reviews the plan, resolves open questions, and explicitly approves.

### Step 6: Generate Implementation SQL (only after approval)

Only proceed here when the user says to. Generate:
1. Any `ALTER TABLE ... SET CHANGE_TRACKING = TRUE` statements needed
2. A `CREATE OR ALTER DYNAMIC TABLE` statement with the updated query
3. A verification query to confirm row counts by source system

Use `CREATE OR ALTER` (not `CREATE OR REPLACE`) to avoid cascading reinit to downstream DTs.

**⚠️ STOP**: Present the SQL for final approval before executing.

## Best Practices

- **Never guess on open questions.** A plan that says "I assumed X" when the PRD says "TBD" is a defect. Surface it and wait.
- **Preserve existing behavior.** If the PRD doesn't mention a current DT feature, don't remove it. Flag it as an assumption: "Plan preserves existing X; PRD is silent on this."
- **Show your mapping.** For every Silver column, trace it back to a specific PRD row or rule. If no PRD row covers a column, note the gap.
- **Separate Silver concerns from Gold concerns.** If the PRD mentions a rule that belongs in Gold (e.g., currency conversion, unified chart of accounts), classify it as "Out of Scope (not Silver)" and say why.
- **Respect the PRD's own status fields.** If a mapping row says "Open -- needs decision", that is not "Confirmed". Carry it through as an open question even if you think the answer is obvious.

## Stopping Points

1. After parsing the PRD (Step 1)
2. After presenting the full plan (Step 5)
3. After generating SQL, before execution (Step 6)

## Output

A structured plan with exactly five sections (New Sources, Column Mapping, Business Rules, Assumptions, Open Questions) plus optional implementation SQL after approval. The plan is the primary artifact -- SQL generation is a follow-on step, not the default.

## Example Usage

```
User: I have a new PRD at ./requirements.xlsx. Use it to plan updates to
      COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES.

CoCo: [Reads XLSX, inspects current DT, produces 5-section plan]
      "The PRD introduces 2 new sources (Baan IV, Workday), adds status
       normalization (BR-001) across all branches, and requires Baan dedup
       (BR-003). I found 3 open questions that need resolution before I can
       generate DDL: payment terms normalization scope (BR-005), Baan cost
       center format handling, and whether to drop SOURCE_ENTITY_CODE per
       BR-008 which contradicts the current DT."

User: [Resolves open questions]

CoCo: [Generates CREATE OR ALTER DYNAMIC TABLE with all 4 sources]
```
