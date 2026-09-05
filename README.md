<div align="center">

# DarGratia Church Suite

**A hierarchical, multi-tenant financial platform for churches and their national offices — recurring tithes and offerings, regulated remittance math, dual-signature monthly reports, and consolidated annual reporting for government and lender filings.**

![Laravel](https://img.shields.io/badge/Laravel-FF2D20?style=flat-square&logo=laravel&logoColor=white)
![PHP](https://img.shields.io/badge/PHP_8.3-777BB4?style=flat-square&logo=php&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white)
![i18n](https://img.shields.io/badge/EN_·_ES_·_PT-0A66C2?style=flat-square)
![Status](https://img.shields.io/badge/status-onboarding%20first%20org-success?style=flat-square)

</div>

> **Documentation-only repository.** Production source and customer data are private. Published here: the domain model, the accounting rules that drove it, and the design decisions that make a regulated-money product defensible.

---

## 1. The business problem

Church treasurers do bookkeeping in spreadsheets, then hand-calculate a monthly remittance owed to their national office, then print and sign it. The arithmetic is subtle, the deadlines are fixed, and the money is other people's donations. Errors are common, uncomfortable to discover, and awkward to correct.

Meanwhile the national office receives dozens of inconsistently-formatted reports by email, portal, and paper — and has no consolidated picture of what came in, who submitted on time, or which congregations are current.

**Two customers, one system:** the local church needs bookkeeping and a correct remittance; the national office needs intake, consolidation, and its own ledger.

---

## 2. Domain model — where most of the difficulty lives

This product is not hard because of its stack. It is hard because the accounting rules are non-obvious and getting them wrong is a financial and reputational problem for the customer.

### 2.1 Remittance is computed on gross, never net

> **Rule:** the amount owed to the national office is calculated on **gross income received**, not on the balance remaining after expenses.
>
> A congregation that receives $10,000 and spends $4,000 still remits against $10,000.

This is the single most common manual error, and it is exactly the kind of rule a naive implementation gets wrong — because "net" is the intuitive default in every other accounting context.

### 2.2 Two independent remittance lines

| Line | Base | Rate |
|---|---|---|
| Tithe of tithes | Total tithes received in the period | 10% |
| Offering remittance | Total offerings received in the period | 10% |

These are computed and displayed **separately**, never summed into a single figure, because the office reconciles them as distinct categories.

### 2.3 Designated funds are excluded from the base

Missions contributions are forwarded to their destination in full through a separate channel. Including them in the offering base would cause the congregation to remit against money it never kept. The offering base therefore excludes designated pass-through funds by category, not by manual adjustment.

### 2.4 Computed, but not coercive

The remittance fields **auto-fill from the computed figure**. A treasurer can override them — because sometimes what was actually sent differs from what was owed, and a system that refuses to record reality gets abandoned for a spreadsheet.

When an override occurs, the report **flags a variance**. The record stays truthful, the discrepancy stays visible, and nobody is forced to lie to the software.

> This is a general principle I apply to any compliance feature: **compute the right answer, make it the default, allow the truth to be recorded, and surface the delta.** Hard enforcement produces workarounds; workarounds produce silent data corruption.

### 2.5 Scope discipline

Deployed **going-forward only** — no retroactive recalculation or audit of prior periods. Retro-auditing a congregation's historical under-remittance would have been technically straightforward and commercially fatal. Knowing which correct feature *not* to build is part of the job.

---

## 3. Hierarchy and tenancy

```
┌────────────────────────────────────────────────────────────┐
│  ORGANIZATION  (national / worldwide office)                │
│  · own books: income and expenditure                        │
│  · consolidated monthly + annual reports                    │
│  · role-managed council (president, VP, treasurer,          │
│    secretary, assistant, auditor, member)                   │
│  · own branding — child churches are onboarded under it     │
└──────────────────┬─────────────────────────────────────────┘
                   │  flat hierarchy — no districts or regions
   ┌───────────────┼───────────────┬──────────────────┐
   ▼               ▼               ▼                  ▼
┌──────┐      ┌──────┐        ┌──────┐           ┌──────┐
│Church│      │Church│        │Church│    ...    │Church│
└──────┘      └──────┘        └──────┘           └──────┘
  Fully isolated entities. No cross-referencing between churches
  anywhere in the system — not in search, not in reports, not in exports.
```

| Decision | Rationale |
|---|---|
| **Flat hierarchy** | Congregations report directly to the national office. Districts were explicitly out of scope; modelling them speculatively would have added a join to every query for no customer |
| **Absolute church isolation** | A treasurer at one congregation must never see another's numbers, in any surface. Enforced at the scope level, same fail-closed pattern as the CRM |
| **Organization as a real ledger** | The office pays rent, love offerings, and assistance outward, and receives income outside the church reports. It needs full double-sided books, not just an inbox |
| **Branded onboarding** | A church invited by an organization receives welcome and onboarding mail branded as the organization, not as the software vendor. The platform is infrastructure, not the relationship |

### Role-based finance visibility

Roles are fixed council positions rather than freely composed permission sets — church councils have standard offices, and inventing a permission builder for a non-technical treasurer is a usability failure.

Finance visibility defaults **on** for the secretary (they need the numbers to do the job) and is **toggleable off per role** by the president. Sensible default, explicit override.

---

## 4. Reporting and signatures

### Monthly report flow

```
  Church closes the period
        │
        ▼
  Report generated ── gross-basis math, two remittance lines,
        │             variance flags if overridden
        ▼
  Dual signature ──── autopen authorized: treasurer may apply the
        │             pastor's signature and vice versa; neither
        │             blocks on the other
        ▼
  Payment method recorded ── check + check number, or Zelle +
        │                    confirmation number
        ▼
  Submitted ── portal · email · printed PDF
        │
        ▼
  Office auto-acknowledges by email on receipt
        │
        ▼
  After the deadline: president's monthly report —
  what came in, who submitted, who is current
```

**Why autopen is a feature, not a shortcut:** the pastor and the treasurer are frequently not in the same place before a filing deadline. A strict "each principal signs personally" model would have made the deadline unmeetable. The customer explicitly authorizes mutual application, and the system records *who applied* each signature, so the audit trail is stronger than a paper original.

### Annual consolidation

One-button annual report aggregating everything received from all congregations, archived immutably. These filings go to government agencies, banks for loan applications, and charity registrars — so the archive has to be reproducible years later, which means reports are stored as rendered artifacts, not regenerated from live data that may have since changed.

---

## 5. Data portability

Two requirements that are easy to underestimate:

**Member import.** Congregations arrive with a CSV or spreadsheet in whatever shape they've kept it. The importer runs **preview-first**: it parses, maps, and reports counts — created, matched, ambiguous — and writes **nothing** until the operator confirms and chooses skip-or-update for matches. An import that silently overwrites a members table is unrecoverable for a customer with no backups.

**Full account export.** Everything — members, funds, categories, transactions, reports, settings, logo, team — exportable and re-importable into a new account, so a customer can be migrated without retyping. Portability is also a trust argument during the sale: the customer's data is theirs, and they can leave.

**Member disambiguation.** Congregations routinely have several people sharing a first and last name. The member card carries a middle initial as a first-class field, because the same full name appearing three times in a donation list is a support ticket every single month.

```
[PLACEHOLDER — paste ~20 lines of sanitized remittance-calculation code here:
 the gross-basis aggregation, the designated-fund exclusion, the two-line
 split, and the variance detection. Redact: real fund/category names.]
```

---

## 6. Code Review Notes — regulated-money defect classes

| Defect | Why it survives review | The check |
|---|---|---|
| **Net-basis remittance** | Reads naturally; matches every generic accounting example a model has seen | Assert against a fixture with material expenses. A test where expenses are zero cannot distinguish gross from net |
| **Designated funds folded into the base** | The aggregation looks tidy and complete | Every fund category must be explicitly classified as included or excluded. No default |
| **Float currency** | Passes on round numbers | Exact decimal or integer minor units, asserted for exact equality |
| **Hard enforcement with no override** | Looks more "correct" and more secure | Ask what the user does when reality disagrees. If the answer is "leaves the system," the constraint is wrong |
| **Report regenerated from live data** | Simpler, less storage | An archived filing must render identically in five years. Store the artifact |

---

## 7. Platform notes

- Trilingual throughout — **English, Spanish, Portuguese** with full key parity; the customer base is Spanish-first and the product is sold across the Americas
- Terminology was deliberately generalized to **Organization / Organización** so the product is not bound to one denomination's vocabulary
- Address and contact completeness is prompted at first login for both organizations and churches — incomplete addresses break official filings, so the block is placed at onboarding rather than at deadline
- Billing: a monthly base for the office plus a per-additional-church rate, card on file. **A declined card triggers outreach, not a lockout** — cutting off a congregation's books over a payment failure is the wrong response to the wrong problem

```
[PLACEHOLDER — screenshot: monthly report with the two remittance lines
 and a variance flag. Use the demo church. Blur any real names.]
```

```
[PLACEHOLDER — screenshot: organization dashboard with churches as rows,
 submission status per period. Demo data only.]
```

---

<div align="center">

**Architecture and write-up by [Ginedy McIntosh](https://github.com/ginedy-mcintosh) · Houston, TX**

</div>
