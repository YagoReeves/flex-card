# Flex Card — Project Context

Standing context for all Flex Agent routines. Use this to:
- bucket signal into the canonical workstreams (§5)
- weigh findings against the critical-path risks (§4)
- avoid attributing work to Card squad that they don't own (§6)

If a routine needs deeper detail than what's here, the canonical sources are linked in §8 — fetch from Notion only when needed.

---

## 1. Product summary

Flex Card is a single Mastercard-branded card issued by **WebBank**, processed by **Marqeta**, that combines two consumer credit products on one credential via Mastercard One Credential (M1C):

- **Primary credential:** Dynamic Secured Non-Revolving Credit (Reg Z open-end). Always-on default for all card spend.
- **Secondary credential:** BNPL Pay-in-4 instalment loans (Reg Z closed-end + Reg E for repayment rails). Serviced by **Peach**. User-initiated via "Pay Later" mode, one purchase per activation.

**Three-account ledger (secured credit only, ledgered by Marqeta):**
- **Wallet Account** — consumer's "debit" hub (direct deposits, AFTs, payroll). Balance dynamically sets the credit limit.
- **Secured Deposit Account (SDA)** — restricted collateral. Funds move from wallet → SDA on every approved credit auth, ensuring 1:1 collateral coverage.
- **Credit Account** — tied to the card's primary PAN. All secured credit transactions post here and appear on the Reg Z statement.

BNPL transactions are recorded in Marqeta's transaction history tagged BNPL-funded but do **not** impact wallet, SDA, or credit account.

**MVP gating:** Builder subscribers only ($14.99/month or $38.25/quarter).

---

## 2. Partner stack

| Role | Partner |
|---|---|
| Sponsor bank | **WebBank** |
| Network | **Mastercard** (M1C-enabled) |
| Processor | **Marqeta** (replacing legacy I2C) |
| Physical card fulfilment | **Idemia** (via IC Payments) |
| BNPL loan servicing | **Peach Finance** |
| Instant transfers (AFT/OCT) | **TabaPay** |
| Direct deposit data | **Pinwheel** |

Bureau reporting (Metro 2) will be built **in-house** — no external vendor. I2C is the **legacy** processor being deprecated; do not treat as active.

---

## 3. Top milestones (T2-26 Card squad plan)

| Milestone | Target |
|---|---|
| Critical path approval + implementation (incl. production key exchange) | **Aug 2026** |
| UAT start (test cases agreed, testers recruited, scripts written) | **Sep 2026** |
| UAT completion + WebBank approval | **Nov 2026** |

UAT readiness by end of September 2026 is the squad's anchor commitment.

---

## 4. Critical-path risks

Every Brief should weigh signal against these. If a finding maps to one of these risks, surface it explicitly.

1. **Product complexity** — ACH solution between Marqeta and WebBank still being locked in. Marqeta agreement gated on ACH resolution. Mastercard CIS / BIN assignment gated on Marqeta agreement. Idemia plastics has a 3-month lead time post-BIN.
2. **Partner approval delays** — Flex is partner-heavy by design (5-way: WebBank + Mastercard + Marqeta + Idemia + TabaPay). Each of 19 MVP features needs approvals from one or more. Approval cycles are outside Cleo's direct control.

---

## 5. Workstream spine

This is the canonical 10-workstream taxonomy used in the Flex Hub. **Bucket all routine output by these.**

| # | Workstream | Owner(s) | Scope |
|---|---|---|---|
| 1 | **Programme** | Jago | Overall governance, dependency tracking, cadence, timeline, cross-partner coordination, UAT/MVT milestone management |
| 2 | **Card Product** | Edu | Onboarding UX, Spend tab refactor, card issuance & activation, card management (PIN/freeze/details), purchase experience (auth, clearing, ATM), disputes (build), offboarding (build), statements |
| 3 | **BNPL Product** | Mariana | Pre/post-purchase BNPL, plan management, BNPL eligibility, Peach loan servicing integration, multi-use card design, audience expansion |
| 4 | **Bank & Compliance** | Jago + Legal | WebBank relationship, **WebBank implementation checklist** (Mirror DB), credit policy, regulatory posture (Reg E/Z), offboarding policy approval, disclosures, bank sign-off on risk posture |
| 5 | **Platform Foundations** | Edu + Banking Ops | Marqeta integration + **Mastercard BIN setup** (see sub-workstreams below) |
| 6 | **Money Movement** | Finance Ops + Payments | AFT/top-ups, EWA flows, DD routing (WebBank direct), OCT/offloads, ATM, ACH withdrawal, BillPay, reconciliation/settlement, TabaPay integration, Pinwheel re-engagement |
| 7 | **Servicing & Operations** | Customer Ops | Disputes operational readiness (CS training, workflow, evidence handling), offboarding ops, live agent support, CS tooling, runbooks |
| 8 | **Card Issuance & Fulfilment** | PMM + Banking Ops | **Physical card / plastics** (design, inventory, activation, reissuance — partners: Idemia, IC Payments, Mastercard), contactless, push provisioning to Apple/Google Wallet, virtual card issuance |
| 9 | **Credit Reporting** | unowned | Bureau reporting approach + ops, Metro 2 file generation/furnishing, vendor decision (Bloom vs in-house) |
| 10 | **Fraud & Risk** | Fraud Infra | Fraud controls on AFT/OCT, ATM monitoring, velocity rules, SEON migration, chargeback management, ATO controls |

### Sub-workstreams (currently active priorities under Platform Foundations)

- **Marqeta migration / implementation** — codebase refactor, sandbox + production environments, JIT integration, API work, migration of existing CBC cardholders from I2C to Marqeta. Currently the largest live build.
- **Mastercard BIN setup** — CIS project initiation and BIN assignment with Mastercard. Gates Idemia plastics and Push Provisioning. Currently a top blocker.

Treat these two as workstreams in their own right when surfacing signal — they're large enough that lumping them into "Platform Foundations" loses resolution.

---

## 6. What Card squad does NOT own

People often misattribute these — don't bucket signal here against Card:
- **Builder upsells** → Growth
- **Builder conversion flow** → Growth
- **Cash advance boosts** → EWA Core
- **Card infrastructure** (new squad being hired)
- **CBC fraud & disputes** (new squad being hired)

---

## 7. Source docs (fetch only if needed)

- **Flex Hub:** https://www.notion.so/Flex-Hub-2ef5c63b874580d4a6dfcdc14c2edbc2 — central project hub, includes workstreams, team structure, timeline DB, decision log
- **Card T2-26 Planning Doc:** https://www.notion.so/Card-T2-26-Planning-Doc-32e5c63b874580ac855cdab0b25b3c3e — milestones, OKRs, dependencies, staffing
- **Product Construct V2 (Web):** https://www.notion.so/Product-construct-V2-for-Web-2fe5c63b8745803ba54ae8a1ec83a369 — full feature-level construct (very long; sections cover Awareness/Onboarding, Card Issuance, Money Movement, Purchase Experience, Card/Account Management, Card Operations, Open Questions)
- **Abridged construct doc (for WebBank):** https://www.notion.so/Abridged-doc-for-Web-3425c63b874580cd9ef7e92447c82d4b — compact version, source for §1 above
- **WebBank Mirror DB** (agent-managed) — checklist tracker, lives in Flex Hub
- **Action Items DB** (agent-managed) — Proposed/Confirmed action items, lives in Flex Hub
