# PDD - No-PO Invoice Chaser

## Document History

| Date | Version | Author | Role | Comments |
|------|---------|--------|------|----------|
| 2026-09-23 | 0.1 | uipath-analyst | Analyst | Initial analysis from request-work/request-details.md (No-PO invoice finder v1.0, UiPath Cartographer, 16 Sep 2026) |

## 1. Document Control

| Field | Value |
|-------|-------|
| Document | PDD - No-PO Invoice Chaser |
| Story key | JACTIV-761 |
| Epic key | [SME REVIEW] |
| Source file | request-work/request-details.md |
| Branch | analysis-jactiv-761 |
| Author | uipath-analyst |
| Status | Draft - pending SME approval |
| Version | 0.1 |

## 2. Introduction

**Process name:** No-PO Invoice Chaser
**Process Full Name:** `NoPoInvoiceChaser`

**Business objective:** Replace the daily manual Coupa review with a fully automated weekday run that finds invoices with no properly linked purchase order, counts them, and sends one Slack message to the AP responsible — ensuring consistent enforcement of the no-PO-no-pay policy and eliminating visibility risk from timing gaps.

**Owning department:** Accounts Payable

| Role | Name / contact |
|------|---------------|
| SME / Process Owner | Irina Capatina — irina.capatina@uipath.com, Slack WLX9BD8FN |
| BA | uipath-analyst |
| Developer | [SME REVIEW] |

## 3. Process Overview

| Attribute | Value |
|-----------|-------|
| Process full name | NoPoInvoiceChaser |
| Function and department | Invoice compliance monitoring — Accounts Payable |
| Short description | Weekday automated run: queries Coupa for invoices with no linked PO (past 7 days, status draft/new, excluding credit notes), counts qualifying records, sends one Slack DM to the AP SME; sends nothing on a clean day |
| Required roles | Automation (unattended); SME receives notification only |
| Trigger and schedule | Time-based — 10:00 Romania time, Monday–Friday |
| Volume (items per day / peak) | ~194 qualifying invoices per run (live sample); peak [SME REVIEW] |
| Average handling time (manual → automated) | Manual: ~daily ad-hoc; Automated: target < 2 min per run [SME REVIEW] |
| FTE effort | [SME REVIEW] |
| Estimated exception rate | Low — data is structured and rules are deterministic; credit-note exclusions and description-only POs are the only known exception types |
| Input data | Coupa invoice list: invoice date, status, PO linkage field, invoice type |
| Output data | Slack Block Kit DM to WLX9BD8FN: invoice count, date window, filtered Coupa URL |

## 4. To-Be Process (High Level)

The automation is a short linear unattended sequence — read, filter, count, send — running Monday–Friday at 10:00 Romania time with no human steps in the normal path.

**What becomes automated:**
- Coupa session start and invoice list retrieval
- Date-window calculation (today minus 7 days)
- Filtering by status (draft/new), invoice type (exclude credit notes), and PO-linkage
- Counting qualifying invoices
- Composing the Block Kit Slack message with count, date window, and filtered Coupa URL
- Delivering the Slack DM to the SME

**What stays human:**
- Receiving the Slack notification and acting on it (raising/linking POs)
- Requester follow-up (out of scope)

**Manual steps that disappear:**
- Manual Coupa login and filter configuration
- Record-by-record review and field copying
- Message drafting and sending

The automation makes no changes to Coupa records and does not create purchase orders.

## 5. Detailed Process Steps

| Step | Action | Application | Expected Result | Remarks |
|------|--------|-------------|-----------------|---------|
| 1.1 | Schedule triggers at 10:00 Romania time on a weekday | Scheduler | Run context initialised | Monday–Friday only; Romania = EET/EEST (BR-04 trigger) |
| 1.2 | Calculate date window: window_end = today; window_start = today − 7 days | Automation | window_start, window_end variables set | Used in Coupa filter and in message footer |
| 2.1 | Authenticate to Coupa | Coupa | Active session established | Credential source [SME REVIEW]; read-only access |
| 2.2 | Query Coupa invoice list filtered by invoice_date ≥ window_start AND invoice_date ≤ window_end | Coupa API/UI | Raw invoice record set returned | Interface type [SME REVIEW] — see section 6 |
| 2.3 | Filter records: retain only status = draft OR status = new | Automation | Subset with qualifying status | Applies BR-04 |
| 2.4 | Filter records: exclude invoice type = credit note | Automation | Credit notes removed from set | Applies BR-03 |
| 2.5 | Filter records: exclude invoices where PO linkage field is properly populated | Automation | Records with valid PO link removed | A PO number in the description field only does NOT satisfy this check (BR-02); only a properly linked PO field passes |
| 2.6 | Count remaining qualifying invoices → invoice_count | Automation | Integer count | Applies BR-05 |
| 3.1 | Decision: invoice_count = 0? | Automation | Branch: zero → step 3.2; non-zero → step 4.1 | Applies BR-07 |
| 3.2 | [Zero path] Send no Slack message; end run successfully | Automation | Run completes; Slack is silent | BR-07: clean day produces no output |
| 4.1 | Build filtered Coupa URL using window_start and window_end | Automation | coupa_url = https://uipath-test.coupahost.com/invoices?q%5Binvoice_date_gteq%5D={{window_start}}&q%5Binvoice_date_lteq%5D={{window_end}}&q%5Bstatus_eq%5D=draft | URL template from source; note PO-linkage filter cannot be expressed in URL (BR-05 note) |
| 4.2 | Compose Slack Block Kit message payload with invoice_count, coupa_url, window_start, window_end, run_date | Automation | JSON Block Kit payload ready | Title: ":receipt: {{invoice_count}} invoices need a purchase order"; body 3 icon-led lines; primary button "Open the list in Coupa"; footer with dates and run_date |
| 4.3 | Send Slack DM to recipient WLX9BD8FN via Slack HTTP Request | Slack | Message delivered; HTTP 200 response | Applies BR-06; DM by Slack member ID, not email; Block Kit not plain text |
| 5.1 | Log run result (count, window, outcome) | Automation | Run log entry written | [DEFAULT] standard logging; no audit-retention design in scope |
| 5.2 | End run | Automation | Run closed | Normal exit |

## 6. Applications and Systems

| Application | Interface type | Access method | Login method | Credential handling | Comments |
|-------------|---------------|---------------|--------------|--------------------|-|
| Coupa | [SME REVIEW] — web UI or REST API | [SME REVIEW] | [SME REVIEW] | Credentials in UiPath Orchestrator Asset [DEFAULT] | Source states read-only access; instance hostname uipath-test.coupahost.com |
| Slack | API (HTTP) | Slack HTTP Request activity / Webhook | Bot token or Webhook URL | Token stored in Orchestrator Asset [DEFAULT] | Block Kit payload; DM to member ID WLX9BD8FN; protocol: Slack Web API or Incoming Webhook [SME REVIEW] |
| UiPath Orchestrator | Scheduler + Asset store | Orchestrator API | Robot service account | [DEFAULT] | Hosts schedule (10:00 EET/EEST weekdays) and credential assets |

## 7. Business Rules

| ID | Rule | Source | Applies at step |
|----|------|--------|----------------|
| BR-01 | Invoices without a properly linked purchase order are subject to the no-PO-no-pay policy and must be identified | BR-001 | 2.5 |
| BR-02 | A PO number typed into the invoice description but not properly linked does not satisfy the PO requirement | BR-002 | 2.5 |
| BR-03 | Credit notes must be excluded from the notification population | BR-003 | 2.4 |
| BR-04 | Include only invoices with status draft or new and an invoice date within the past seven days | BR-004 | 2.2, 2.3 |
| BR-05 | Count the qualifying invoices; report the count in the message; do not list individual invoices | BR-005 | 2.6, 4.2 |
| BR-06 | Send the count to SME Irina Capatina (Slack member ID WLX9BD8FN) by Slack DM, with a no-PO-no-pay explanation, a request to link POs, and a filtered Coupa URL | BR-006 | 4.2, 4.3 |
| BR-07 | Send nothing when a successful query returns no qualifying invoices | BR-007 | 3.1, 3.2 |
| BR-08 | No retry, fallback or recovery behaviour is required; a run that cannot complete is a failed run | BR-008 | 9.1–9.5 |
| BR-09 | A run that cannot complete produces no notification; no second message path exists | BR-009 | 9.1–9.5 |
| BR-10 | The automation must not create or modify purchase orders, approve invoices, change Coupa records, or track requester completion | BR-010 | All steps |

## 8. Business Exceptions

| ID | Name | Trigger step | Trigger condition | Action |
|----|------|-------------|-------------------|--------|
| B1 | Credit note encountered | 2.4 | Invoice type = credit note | Exclude record from result set; continue processing remaining records |
| B2 | Description-only PO | 2.5 | PO text present in description field only; PO linkage field empty or invalid | Treat as missing PO; include record in qualifying set if other rules pass (BR-02) |
| B3 | No qualifying invoices (clean day) | 3.1 | invoice_count = 0 after all filters applied | Send no Slack message; end run successfully (BR-07) |

## 9. System Errors

| ID | Name | Trigger condition | Severity | Retry policy | Action |
|----|------|-------------------|----------|--------------|--------|
| S1 | Coupa application unresponsive | Coupa login or query request times out or returns HTTP 5xx | High | None (BR-08) | Log error; terminate run; report as failed run |
| S2 | Coupa element / field not found | Expected invoice list element or field absent during query/filter | High | None (BR-08) | Log error; terminate run; report as failed run |
| S3 | Slack message delivery failure | Slack HTTP request returns non-200 / webhook error | High | None (BR-08) | Log error; terminate run; report as failed run; no retry DM sent (BR-09) |
| S4 | Network timeout | HTTP request to Coupa or Slack times out | Medium | None (BR-08) | Log error; terminate run; report as failed run |
| S5 | Credential expiry | Orchestrator Asset unavailable or credential rejected at login | High | None (BR-08) | Log error; terminate run; alert Orchestrator operator [DEFAULT] |
| S6 | Unhandled exception | Any uncaught runtime error | High | None (BR-08) | Log full exception; terminate run; report as failed run [DEFAULT] |

## 10. Assumptions, Dependencies and Open Questions

1. **OQ-01 - Coupa interface type.** [SME REVIEW] Is the Coupa invoice list queried via REST API or browser UI automation?
2. **OQ-02 - Coupa authentication method.** [SME REVIEW] What credential type does the robot use to authenticate to Coupa (API key, OAuth, username/password)?
3. **OQ-03 - Slack delivery mechanism.** [SME REVIEW] Is the Slack notification sent via the Web API (bot token) or an Incoming Webhook URL?
4. **OQ-04 - Coupa hostname environment.** Source URL is uipath-test.coupahost.com; confirm production hostname before go-live.
5. **OQ-05 - Reminder wording and date format.** [SME REVIEW] Confirm final wording and readable date format for window_start/window_end in message footer (Next Steps item 3).
6. **OQ-06 - Failed-run alerting channel.** [DEFAULT] Orchestrator job failure alert assumed; confirm whether an additional human notification is wanted on run failure.
7. **OQ-07 - No-qualifying-invoices message.** Source section 7 states a congratulatory Slack message is sent on a clean day; BR-007 and section 2.1 scope say send nothing. [SME REVIEW] Resolve conflict — this PDD implements BR-07 (no message on clean day) pending SME decision.
8. **OQ-08 - Volume peak.** Live sample is 194; peak volume and any pagination requirement for Coupa query are unknown. [SME REVIEW]
9. **Coupa access is read-only.** Confirmed by scope boundary; no write operations are performed.
10. **Block Kit payload.** Source states exact JSON is in architectural considerations section 4; that section is not included in the input document. Developer must obtain the verbatim payload from the SME.

## 11. Success Criteria

1. A weekday test run executes at 10:00 Romania time and completes without error.
2. Only invoices with status draft or new and invoice date within the past seven days are evaluated; credit notes are excluded.
3. An invoice with a PO number in the description field only is counted as missing a linked PO.
4. The Slack DM to WLX9BD8FN reports the correct total count and includes a working filtered Coupa URL.
5. A successful run with zero qualifying invoices delivers no Slack message.
6. A run that fails is recorded as a failed run in Orchestrator and delivers no Slack message.
