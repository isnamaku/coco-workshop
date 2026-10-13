# SILVER_AP_INVOICES -- PRD-Driven Update Handoff

**Date:** 2026-10-13
**Target:** `COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES`
**Connection:** ZZ26719 (ACCOUNTADMIN)

---

## 1. PRD Input Files

Downloaded from the Cortex Code Foundations quickstart assets:

| File | Purpose |
|------|---------|
| `sample_business_requirements_source_onboarding.csv` | Baan IV and Workday onboarding requests, contacts, go-live dates, blockers |
| `sample_business_requirements_column_mapping.csv` | 32-row source-to-Silver column mapping for Baan (16 rows) and Workday (16 rows) |
| `sample_business_requirements_business_rules.csv` | 10 business rules (BR-001 through BR-010) governing status normalization, dedup, payment terms, GL handling, refresh config |

---

## 2. Business Rules Applied

| Rule | What It Does | Disposition |
|------|-------------|-------------|
| BR-001 | Status normalization across all sources | Implemented -- CASE on Oracle, Baan, Workday branches |
| BR-002 | No currency conversion at Silver | Already correct -- no change |
| BR-003 | Baan dedup on INVOICE_NUMBER | Implemented -- QUALIFY ROW_NUMBER() on BAN_INVOICE_REF |
| BR-004 | Flag invoices > $500K | Out of scope -- DMF, not Silver DT |
| BR-005 | Payment terms normalization | Deferred to Gold per default recommendation; Oracle REPLACE reverted |
| BR-006 | GL account stored as-is | Already correct -- no change |
| BR-007 | SOURCE_SYSTEM literal per branch | Extended with 'BAAN' and 'WORKDAY' |
| BR-008 | Drop system-specific columns | Implemented -- removed SOURCE_ENTITY_CODE, SOURCE_DOCUMENT_TYPE |
| BR-009 | TARGET_LAG = DOWNSTREAM | Deferred -- kept '1 hour' until Gold DT exists |
| BR-010 | Data retention | Out of scope -- architecture note only |

---

## 3. Schema Change Summary

**Before (2 sources, 18 columns):**
SAP + Oracle, with SOURCE_ENTITY_CODE and SOURCE_DOCUMENT_TYPE, Oracle payment terms normalized (N30→NET30), Oracle status passed through raw.

**After (4 sources, 16 columns):**
SAP + Oracle + Baan + Workday. SOURCE_ENTITY_CODE and SOURCE_DOCUMENT_TYPE dropped. Oracle payment terms reverted to raw. Status normalized across all branches. Baan dedup added.

---

## 4. Assumptions Requiring Engineering Review

1. **Payment terms as-is at Silver.** Downstream consumers filtering on `PAYMENT_TERMS = 'NET30'` will miss Oracle/Baan rows (now `N30`). Audit downstream queries.
2. **Dropped columns.** Any query selecting `SOURCE_ENTITY_CODE` or `SOURCE_DOCUMENT_TYPE` from Silver will break. PRD says join back to Bronze via `SOURCE_INVOICE_ID + SOURCE_SYSTEM`.
3. **TARGET_LAG = '1 hour'** instead of DOWNSTREAM. Switch when Gold DT is created.
4. **Baan timestamps assumed UTC.** PRD says CET→UTC conversion happens in extract. Validate with Karen van der Berg.
5. **Baan cost center format** (`BC-XX` vs `BC-XXX`) passed through without normalization. Both formats coexist.
6. **Workday branch included despite pending legal signoff** (DPA-2025-0041). Remove if legal rejects.

---

## 5. Open Questions (Resolved with Defaults)

| # | Question | Default Used | Owner |
|---|----------|-------------|-------|
| 1 | Normalize payment terms at Silver or Gold? | Gold (BR-005 recommendation) | Sarah Chen + David Kim |
| 2 | Drop SOURCE_ENTITY_CODE / SOURCE_DOCUMENT_TYPE? | Yes (BR-008) | David Kim |
| 3 | Change TARGET_LAG to DOWNSTREAM? | No, keep '1 hour' until Gold exists | Engineering |
| 4 | Normalize Baan cost center format? | No, pass through as-is | Karen van der Berg |
| 5 | Include Workday before legal signoff? | Yes, include in DDL | Jennifer Okafor |

---

## 6. Validation Queries

Run these after any future re-deploy to confirm correctness:

```sql
-- Row counts by source (expect SAP:15, Oracle:15, Baan:10, Workday:10 = 50)
SELECT SOURCE_SYSTEM, COUNT(*) AS ROW_COUNT,
       COUNT(DISTINCT INVOICE_NUMBER) AS UNIQUE_INVOICES
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM ORDER BY SOURCE_SYSTEM;

-- Status normalization: only APPROVED and PENDING, no raw source values
SELECT SOURCE_SYSTEM, APPROVAL_STATUS, COUNT(*) AS CNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM, APPROVAL_STATUS ORDER BY 1, 2;

-- Payment terms pass-through: each source retains native format
SELECT SOURCE_SYSTEM, PAYMENT_TERMS, COUNT(*) AS CNT
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
GROUP BY SOURCE_SYSTEM, PAYMENT_TERMS ORDER BY 1, 2;

-- Baan dedup: zero duplicates on INVOICE_NUMBER
SELECT INVOICE_NUMBER, COUNT(*) AS DUPES
FROM COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES
WHERE SOURCE_SYSTEM = 'BAAN'
GROUP BY INVOICE_NUMBER HAVING COUNT(*) > 1;

-- Schema check: 16 columns, no SOURCE_ENTITY_CODE or SOURCE_DOCUMENT_TYPE
SELECT COLUMN_NAME
FROM COCO_WORKSHOP.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'PIPELINE_LAB' AND TABLE_NAME = 'SILVER_AP_INVOICES'
ORDER BY ORDINAL_POSITION;
```

---

## 7. DDL Recovery

Retrieve the current DDL at any time:

```sql
SELECT GET_DDL('DYNAMIC_TABLE', 'COCO_WORKSHOP.PIPELINE_LAB.SILVER_AP_INVOICES');
```

---

## 8. Reusing the PRD Evaluator Skill

The project skill is at `.cortex/skills/prd-to-pipeline-plan/SKILL.md`. To run it:

```
$prd-to-pipeline-plan
```

It expects two inputs:
- **prd_path** -- local file (XLSX or CSV) containing source onboarding, column mappings, and/or business rules
- **target_dynamic_table** -- fully qualified name of the DT to update

The skill always returns 5 sections: New Sources, Column Mapping Changes, Business Rules Impact, Assumptions, Open Questions. It will not generate DDL until open questions are resolved and the user approves.

---

## 9. Prerequisites for New Source Onboarding

When onboarding the next source system, ensure:

1. Bronze table exists in `COCO_WORKSHOP.SOURCE_DATA` with change tracking enabled
2. PRD file covers column mapping (source column → Silver column + transform) and any status/payment terms normalization rules
3. No dedup logic is assumed -- PRD must state whether the source has known duplicate issues
