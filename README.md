# Digital Dormancy Analysis — Python + Oracle SQL (Sanitized)

This repository showcases an analytics workflow that measures **digital dormancy and engagement** for a retail-banking–style environment. It produces a normalized **metrics fact** table and a companion CSV that can be dropped into BI dashboards.

> **Why it exists**  
> Demonstrate end-to-end KPI engineering: windowed cohorts, last-action attribution across channels (Web/App), offer segmentation, and consolidated reporting into a single table.

---

## What this project does

- **Materializes a base population** of open card customers (`CUST_OPEN_CARD`) with a derived **offer** label from application campaign types (`OFFER_A`, `OFFER_B`, `OFFER_C`, `OTHER`).
- **Joins a “digital ladder”** (CSV) to classify customers into **segments**:
  - `Friction` 
  - `Onboarding` 
- **Attributes the last digital action** by stitching Web and App event streams:
  - Web : `DM_WEB_LOGON`, `DM_WEB_EVENTS`
  - App : `DM_APP_LOGON`, `DM_MOBILE_EVENTS`
- **Builds cohorts and behaviors**:
  - Contact-centre callers (CSR) in the last 90 days 
  - 90-day **spend-active** cardholders
  - **Balance-transfer-only** statement holders in the current month
  - **Dormancy buckets**: `90–120`, `121–180`, `180+` days since last digital action
- **Writes everything** to a single, dashboard-ready table: `METRIC_RESULTS`.

---

## Inputs 

**Database objects (schema placeholders):**
- `WAREHOUSE.DM_CARD`, `WAREHOUSE.DM_APPLICATION`
- `WAREHOUSE.DM_WEB_LOGON`, `WAREHOUSE.DM_WEB_EVENTS`
- `WAREHOUSE.DM_APP_LOGON`, `WAREHOUSE.DM_MOBILE_EVENTS`
- `WAREHOUSE.DM_CARD_TXN`, `WAREHOUSE.DM_CARD_CYCLE`

**Flat files (local demo paths):**
- `./data/digital_ladder_sample.csv`
- `./data/csr_calls_sample.xlsx`

> Replace with your own synthetic/demo data where appropriate.

---

## Time windows

- `START_OF_MONTH = ADD_MONTHS(TRUNC(SYSDATE, 'MONTH'), -1)`
- `END_OF_MONTH   = TRUNC(SYSDATE, 'MONTH') - 1`
- `NINETY_DAYS_AGO_MONTH_ANCHORED = TRUNC(SYSDATE, 'MONTH') - 91`

These anchors are used inside SQL to produce **month-to-date** and **rolling 90-day** cohorts with clear, auditable semantics.

---

## Intermediate tables (materialized during the run)

| Table                     | Purpose (high level)                                                               |
|---------------------------|-------------------------------------------------------------------------------------|
| `CUST_OPEN_CARD`          | Open card holders + derived `OFFER` label                                           |
| `LAST_WEB_LOGIN`          | Last known Web login timestamp per customer                                         |
| `LAST_APP_LOGIN`          | Last known App login timestamp per customer                                         |
| `LAST_DIGITAL_LOGIN`      | Unified last digital timestamp + channel (`WEB`/`APP`)                              |
| `LAST_APP_ACTION`         | App event at the last digital timestamp                                             |
| `LAST_WEB_DATE_TIME`      | Helper for last Web action (date + max time)                                       |
| `LAST_WEB_ACTION`         | Web event at the last digital timestamp                                             |
| `SPEND_ACTIVE_90D`        | Customers with spend transactions in last 90 days                                   |
| `CARD_BT_ONLY_INT`        | Monthly statement snapshot (for BT vs total balances)                               |
| `CARD_BT_ONLY`            | Customers with *only* BT balance on last statement                                  |
| `LAST_DIGITAL_PERIOD`     | Dormancy bucket per customer (90–120 / 121–180 / 180+ days)                         |

All of the above are **best-effort dropped** at the end of the script.

---

## Segments & offers

- **Segments** (from `digital_ladder_sample.csv`):
  - `Friction` and `Onboarding`
- **Offers** (derived from `APP_CAMPAIGN_TYPE`):
  - `OFFER_A`, `OFFER_B`, `OFFER_C`, `OTHER`

---

## Metrics catalog (what gets inserted)

For each **segment** × **offer** combination:

- **Size of cohort**
  - `Total customers`
- **Last digital action tallies**
  - Counts of `LAST_ACTION` from unified Web/App streams
- **Service behavior**
  - `Customers calling contact centre (90d)` (deduped by customer reference)
- **Product behavior**
  - `Spend-active customers (90d)` (transaction group contains “SPEND”)
  - `Only BT balance on last statement` (monthly cycle)
- **Dormancy**
  - `Dormancy: 90–120 days`
  - `Dormancy: 121–180 days`
  - `Dormancy: 180+ days`

---

## Output artifacts

### 1) Metrics table (in-DB)

**`METRIC_RESULTS`**

```sql
CATEGORY        -- varchar2(40)  : Segment label (e.g., 'Friction', 'Onboarding')
OFFER           -- varchar2(16)  : OFFER_A / OFFER_B / OFFER_C / OTHER
METRIC          -- varchar2(200) : Metric name (see catalog above)
FIGURES         -- number        : Count for that metric
