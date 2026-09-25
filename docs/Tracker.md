# Tracker.md — Crunch Build Tracker

Living document. Update it daily — check items off as they're actually true in the running code/demo, not as "should be done by now." If a phase slips, move the date, don't hide the slip. Cross-references: `[FR-N]` → `PRD.md` §6; `[Phase N]` → `ImplementationPlan.md`; `[R-N]` → `Rules.md` §1 hard constraint number.

---

## 0. Status at a glance

| Field | Value |
|---|---|
| Today's date | _fill in_ |
| Current phase | _fill in_ |
| Days remaining to 30 Sept | _fill in_ |
| Biggest open risk right now | _fill in_ |
| Demo run-through last timed at | _fill in / "not yet run"_ |

---

## 1. Phase checklist (mirrors ImplementationPlan.md)

### Phase 0 — Setup (Mon)
- [ ] Backend scaffolded (plain Node/Fastify or Express)
- [ ] Voice worker scaffolded from `agent-starter-node`
- [ ] Frontend scaffolded from `agent-starter-react` + `/dashboard` route added
- [ ] Postgres running, `schema.sql` applied
- [ ] Sandbox link obtained, one real FI response fetched and saved as fixture
- [ ] Q1 answered: is `transactionTimestamp` real time-of-day or stubbed to `00:00:00`? → **Answer:** _______
- [ ] Q2 answered: is per-transaction `currentBalance` populated? → **Answer:** _______
- [ ] Q3 answered: how many months of history returned? → **Answer:** _______
- [ ] `docs/sandbox-findings.md` written

**Phase 0 done:** ☐ Yes ☐ No — if No, do not start Phase 1.

### Phase 1 — Engine core (Tue–Wed)
- [ ] `clustering.ts` — inflow streams + obligation ledger `[FR-1, FR-3]`
- [ ] `cdf.ts` — empirical arrival-day CDF per stream `[FR-2]`, `lowConfidence` flag wired
- [ ] `balance.ts` — balance curve reconstruction `[FR-4]` (interpolation fallback built regardless of Q2 answer, `degradedMode` flag wired)
- [ ] `funding.ts` — P(funded) scoring `[FR-5]`
- [ ] `optimiser.ts` — due-date reassignment + coverage gate `[FR-6, FR-7, FR-8]`
- [ ] `explain.ts` — deterministic explanation strings
- [ ] `simulate.ts` — Monte Carlo what-if `[FR-9]`
- [ ] Backtest harness built and run
- [ ] **Accuracy number recorded:** _______ (include failure rate, not just success rate)

**Phase 1 done:** ☐ Yes ☐ No

### Phase 2 — Auth, consent, API server (Wed–Thu)
- [ ] OTP + PIN login `[FR-10]`
- [ ] Step-up OTP middleware on consent/warranty routes `[FR-11]`
- [ ] AA consent object builder — purpose code, fetchType, frequency, consentMode, dataLife, dataRange `[FR-14]`
- [ ] Consent parameters rendered to frontend `[FR-15]`
- [ ] FI fetch wired to live sandbox `[FR-14]`
- [ ] Pause / revoke / deletion-attestation + audit log `[FR-16, FR-23]`
- [ ] Engine auto-runs post-fetch, insight objects persisted
- [ ] `schemes.json` populated with 5–15 real, sourced schemes `[FR-17b]`
- [ ] Every `SchemeRecord.sourceUrl` verified as a real gov.in/official link

**Phase 2 done:** ☐ Yes ☐ No — full consent→fetch→engine→insight flow tested against **live sandbox**, not just fixture.

### Phase 3 — Voice channel (Thu)
- [ ] Single persona/instruction source of truth written `[FR-19]`
- [ ] `getInsightSummary(userId)` tool wired `[FR-17a, FR-18]`
- [ ] `matchSchemes(profileFields)` tool wired `[FR-17b, FR-20]`
- [ ] Caller-ID + PIN call-auth wired `[FR-12]`
- [ ] **Structural check done:** confirmed voice worker's service token cannot reach any `/aa/*` endpoint or raw transaction data `[R-6]`
- [ ] Frontend `/call` route joins LiveKit room per template

**Phase 3 done:** ☐ Yes ☐ No — live call tested: auth → insight explanation → scheme follow-up, worker logs checked for zero AA/transaction calls.

### Phase 4 — Warranty, dashboard, governance surface (Fri)
- [ ] `warranty.ts` — enrollment gated on coverage gate `[FR-21]`
- [ ] Loss-ratio dial live-updating on dashboard `[FR-22]`
- [ ] Dashboard: insight cards, warranty status, refusal statement `[FR-24]`
- [ ] SEBI boundary copy in-app (not just slides) `[R-1]`
- [ ] IRDAI-safe warranty copy in-app (not just slides) `[R-8]`
- [ ] Response filter/checklist blocking instrument-specific advice built and tested `[R-1]`

**Phase 4 done:** ☐ Yes ☐ No

### Phase 5 — Household/fundability placeholders, architecture diagram (Fri–Sat)
- [ ] Household wireframe screens built (static/non-functional), labeled as preview
- [ ] Fundability static screen built
- [ ] Architecture/trust-boundary diagram produced, matches actual code (not aspirational)

**Phase 5 done:** ☐ Yes ☐ No

### Phase 6 — Full run-through and hardening (Sun–Mon)
- [ ] Full demo run timed end to end — **result:** _______ minutes (target ≤20)
- [ ] Cut list decided if over time (what got cut): _______
- [ ] Backup screen-capture recording made
- [ ] Adversarial Q&A rehearsal run — who played hostile examiner: _______
- [ ] Issues found in adversarial pass and fixed: _______

**Phase 6 done:** ☐ Yes ☐ No

---

## 2. Recommended deliverables (from the competition brief — separate from code phases)

| # | Deliverable | Status | Notes |
|---|---|---|---|
| 1 | Architecture diagram (data flows, trust boundaries) | ☐ Not started ☐ Draft ☐ Done | |
| 2 | Live sandbox/API demonstration | ☐ Not started ☐ Draft ☐ Done | |
| 3 | Customer journey and consent flow (walkthrough) | ☐ Not started ☐ Draft ☐ Done | can reuse `AppFlow.md` §1.4/§1.8 |
| 4 | One-page regulatory/privacy/risk note | ☐ Not started ☐ Draft ☐ Done | must include SEBI + IRDAI paragraphs, `PRD.md` §8.3/§8.4 |
| 5 | One-page business model note | ☐ Not started ☐ Draft ☐ Done | warranty arithmetic + honest gap (no named lender yet) |
| 6 | Final deck | ☐ Not started ☐ Draft ☐ Done | submitted before finals |

---

## 3. Open decisions (from PRD §12) — resolve, don't let these linger

| Item | Status | Decision / Answer |
|---|---|---|
| Current Gemini Live model name/availability confirmed | ☐ Open ☐ Resolved | |
| Sandbox `currentBalance` depth confirmed (Phase 0) | ☐ Open ☐ Resolved | |
| Full web app vs. clickable prototype for household-facing UI | ☐ Open ☐ Resolved | |
| Named lender/buyer for proof-export line (20 emails sent, per Study Brief) | ☐ Open ☐ Resolved | Emails sent: ___ / 20. Replies: ___ |

---

## 4. Rules.md compliance self-audit (run this before submission, not just once)

Check each — if any box is unchecked at submission time, that's a stop-ship issue, not a nice-to-have:

- [ ] No specific-instrument investment advice anywhere in UI copy, agent output, or deck `[R-1]`
- [ ] No blockchain/DLT language anywhere `[R-2]`
- [ ] No biometric auth anywhere `[R-3]`
- [ ] No claimed live Aadhaar eKYC `[R-4]`
- [ ] No quantum computing language anywhere `[R-5]`
- [ ] Voice worker structurally cannot reach raw transaction data `[R-6]`
- [ ] Low-confidence outputs visibly flagged, not smoothed over `[R-7]`
- [ ] Warranty copy is service-level-remedy framed, not insurance framed `[R-8]`
- [ ] Degraded-mode flag (if active) is visible in the UI `[R-9]`
- [ ] No out-of-scope features silently added without a PRD update `[R-10]`

---

## 5. Blockers log

| Date | Blocker | Owner | Resolved? | Resolution |
|---|---|---|---|---|
| | | | ☐ | |
| | | | ☐ | |
| | | | ☐ | |

---

## 6. Demo timing breakdown (fill in during Phase 6, compare against AppFlow/PRD target)

| Beat | Target | Actual (last run) |
|---|---|---|
| Household clip | 0–2 min | |
| Problem framing | 2–4 min | |
| Live: consent → fetch → Rhythm engine output | 4–9 min | |
| Live: call demo (insight + scheme lookup) | 9–11 min | |
| Consent lifecycle theatrics | 11–13 min | |
| Business model | 13–16 min | |
| Trust/regulatory slide | 16–18 min | |
| Buffer / Q&A transition | 18–20 min | |
| **Total** | **20 min** | |

If actual total exceeds 20 minutes after two run-throughs, cut the call-demo beat down to one scheme example first (per `PRD.md` §10, Sun task) before cutting anything else.

---

## 7. Daily log (append, don't overwrite)

```
Date:
What got done:
What didn't, and why:
Tomorrow's one non-negotiable task:
```
