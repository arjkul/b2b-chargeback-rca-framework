# b2b-chargeback-rca-framework

> A structured 12-step root cause analysis framework for B2B EDI chargeback investigations — built from a real multi-system investigation at ShipMonk.

---

## Overview

EDI chargebacks are one of the most operationally complex problems in B2B wholesale fulfillment. Retailers like REI, Target, and Walmart issue chargebacks for compliance violations — but the root cause typically spans multiple systems: the WMS, the EDI notification layer, the order management system, and the carrier.

This framework documents the investigation methodology I developed and used to run chargeback RCAs at ShipMonk — tracing violations from the retailer's deduction back through Snowflake, PostgreSQL, and EDI event logs to a definitive root cause.

---

## The Problem Space

### Common EDI Chargeback Violation Codes

| Code | Violation | Typical Root Cause |
|---|---|---|
| NC06 | Missing ASN (Advance Ship Notice) | ASN not sent, sent late, or sent with wrong data |
| NC08 | Manual Receipt of Carton | Retailer received without ASN — manual processing triggered |
| NC01 | Routing Guide Violation | Wrong carrier or service level used |
| NC02 | Labeling Error | GS1-128 label missing or malformed |
| NC04 | PO Not Acknowledged | 855 transaction not sent within SLA |

---

## The 12-Step RCA Framework

### Phase 1: Scope the Violation
**Step 1 — Identify all affected purchase orders**
Pull the chargeback document and map each deduction to a PO number, violation code, and dollar amount.

**Step 2 — Confirm PO existence in the WMS**
Query the order management system to confirm the POs exist, their fulfillment dates, and assigned warehouse.

```sql
-- Snowflake: confirm PO and fulfillment window
SELECT
    po_number,
    merchant_id,
    warehouse_id,
    created_at,
    fulfilled_at,
    DATEDIFF('day', created_at, fulfilled_at) AS fulfillment_days
FROM wholesale_orders
WHERE po_number IN ('PO-001', 'PO-002', 'PO-003')
```

---

### Phase 2: Reconstruct the Fulfillment Timeline
**Step 3 — Pull shipment events from the binlog**
The binlog captures every state change on an order. Reconstruct the fulfillment timeline end-to-end.

**Step 4 — Check for SLA breaches**
Compare actual ship dates to the retailer's routing guide SLA. Late fulfillment is often the trigger for downstream EDI failures.

**Step 5 — Identify carton and label records**
Verify that carton labels were generated, and that GS1-128 SSCCs were assigned correctly.

---

### Phase 3: Investigate the EDI Layer
**Step 6 — Query ASN send events**
Pull EDI notification records to determine whether an 856 ASN was generated and transmitted.

```sql
-- PostgreSQL (ACCI): ASN notification history
SELECT
    id,
    order_references->>'poNumber' AS po_number,
    transaction_type,
    status,
    created_at,
    error_message
FROM edi_notifications
WHERE
    order_references->>'poNumber' IN ('PO-001', 'PO-002', 'PO-003')
    AND transaction_type = '856'
ORDER BY created_at
```

**Step 7 — Check for acknowledgment errors**
Look for "Acknowledgment already exists" or duplicate transaction errors that may have blocked retransmission.

**Step 8 — Verify ASN timing against ship event**
The ASN must be sent within the retailer's SLA window after the ship event. Late ASNs trigger NC06/NC08 regardless of whether the order shipped on time.

---

### Phase 4: Determine Root Cause
**Step 9 — Map the failure chain**
Document the exact sequence of events that led to the violation. Common chains:

```
Late fulfillment (11–12 days)
    → Ship event fired outside retailer SLA window
        → ASN generated but retailer's system rejected as late
            → Retailer manually received carton
                → NC06 + NC08 violations issued
```

**Step 10 — Distinguish systemic vs. one-time failures**
Determine whether this was a process failure (late pick), a system failure (ASN not generated), or a configuration failure (wrong retailer EDI setup).

---

### Phase 5: Document and Remediate
**Step 11 — Write the RCA summary**
Document findings in a structured format: timeline, root cause, contributing factors, impacted POs, total chargeback value.

**Step 12 — Define corrective actions**
Map each root cause to a specific corrective action with an owner and due date.

| Root Cause | Corrective Action | Owner | Due Date |
|---|---|---|---|
| Late fulfillment | Add SLA breach alert in Metabase for wholesale orders > 7 days | Data/Ops | TBD |
| ASN not sent during fulfillment | Audit ASN trigger logic for orders fulfilled during system outage window | EDI Engineering | TBD |
| Duplicate ACK errors blocking retry | Fix retry logic in ACCI to handle duplicate acknowledgment gracefully | Engineering | TBD |

---

## Data Systems Used

| System | Purpose |
|---|---|
| Snowflake / MARTS |  Order data, fulfillment events, merchant config |
| ACCI (PostgreSQL) |  EDI notification history, ASN send events |
| SHIPMONK_BINLOG |  Raw event log for order state changes |

---

## Case Study: 

**Merchant**: Wholesale apparel brand (anonymized)
**Retailer**: Major outdoor retailer
**POs affected**: 3 purchase orders
**Violation codes**: NC06 (Missing ASN), NC08 (Manual Receipt of Carton)

**Key findings:**
- All 3 POs fulfilled 11–12 days after order creation — significantly past the retailer's 5-day SLA
- No ASN notifications were sent during the January 2026 fulfillment events
- December 2025 ACCI logs showed "Acknowledgment already exists" errors that blocked retry logic
- Root cause was a combination of late fulfillment and a silent ASN transmission failure

**Outcome**: Full timeline documented, engineering ticket filed for ACCI retry fix, Metabase alert created to flag wholesale orders approaching 7-day age threshold.

---

## Related Repos

- [`warehouse-ops-intelligence`](https://github.com/arjkul/warehouse-ops-intelligence) — Ops metrics BI system
- [`metabase-sql-playbook`](https://github.com/arjkul/metabase-sql-playbook) — SQL patterns including EDI and billing queries
