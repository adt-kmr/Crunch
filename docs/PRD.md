# PRD — Crunch
### AA-Timing Financial Wellness Service with Voice Accessibility Channel
Version 1.0 · Prepared for CDPG National Finals build (IIM Bangalore, 30 Sept 2026)

---

## 1. Product summary

Crunch is a financial wellness service for aspiring middle-class Indian households (~₹30,000/month) built on Account Aggregator (AA) data. It reads consented bank-account data **as a timing dataset**: it detects when income actually lands, reorders which recurring debits can be rescheduled so they fall after inflow, and sells the resulting punctuality as a warranty product. A voice/call channel makes the same insight accessible to users who won't or can't use a dashboard, and additionally surfaces government-scheme eligibility as a secondary intervention.

**One-line thesis:** *This household pays a tax on being illegible. Crunch reorders debts to land after income arrives, and turns proof of punctuality into a credit identity.*

This PRD covers the MVP scope for an 8-day build. It does not cover production hardening, fundraising, or full telephony (PSTN/SIP) — those are explicitly out of scope (see §4).

---

## 2. Goals

### 2.1 Product goals
- Demonstrate a working, live AA-data pipeline: consent → fetch → timing model → reschedule recommendation.
- Demonstrate a punctuality warranty with a defensible, model-gated loss ratio.
- Demonstrate a voice channel that explains the same insight and answers one class of follow-up (government scheme eligibility) without touching raw transaction data live.
- Demonstrate the full consent lifecycle (grant → fetch → pause → revoke → deletion attestation) on screen.

### 2.2 Non-goals (explicitly out of scope for this build)
- Personalized investment advice of any kind (see §8.3 — regulatory boundary).
- Telephony/PSTN/SIP/Exotel/toll-free/missed-call infrastructure.
- Aadhaar-based eKYC as a live integration.
- Voice or fingerprint biometric authentication.
- Blockchain/distributed-ledger consent tracking (a standard audit-logged table is sufficient and more defensible).
- Household Layer 2 as a working feature — it is wireframe-only.
- RAG/vector search/embeddings over financial data.
- Production-grade security certification, rate limiting, abuse prevention, multi-tenant scaling.

---

## 3. Users and context

**Primary persona:** a household earning ~₹30,000/month combined, with income arriving irregularly (wages, gig payouts, informal cash) against fixed-date obligations (EMIs, SIPs, utility mandates). Low-to-moderate digital literacy. May prefer voice/vernacular interaction over an app dashboard. Identifiable in AA data itself via `mode = CASH` transactions.

**Secondary actor (Layer 2, wireframe only):** a family member who, after ~3 months of clean Layer 1 service, grants a second AA consent to enable a consolidated household view.

**Judges/evaluators (for the finals context):** expect live demonstration of AA/API depth, explainability, and regulatory awareness — not just a polished UI. Build accordingly: prioritize depth and correctness of the core engine over UI polish.

---

## 4. Scope

### 4.1 In scope for this build (MVP)
| Area | What's built |
|---|---|
| Auth | Mobile OTP + PIN (2FA) login; step-up OTP for high-stakes actions |
| AA integration | Perfios/Anumati sandbox connection; consent object with purpose code 102/104, fetchType PERIODIC, frequency DAILY, VIEW-preferred consentMode, bounded dataLife/dataRange |
| Engine | Inflow-stream clustering, obligation ledger, funding-probability scoring from per-transaction `currentBalance`, greedy/exhaustive optimiser for due-date reassignment, Monte Carlo/bootstrap simulation for "what if" projection |
| Insight surface | In-app cards: before/after reschedule, warranty coverage status, explainability text per recommendation |
| Warranty | ₹49/month punctuality warranty; loss-ratio dial tied to the coverage-confidence threshold |
| Voice channel | LiveKit + Gemini Live agent; light call-auth (caller-ID + spoken/DTMF PIN); reads only pre-computed insight objects; separate fixed-dataset tool for government-scheme eligibility |
| Consent lifecycle | Grant, pause, revoke, deletion-attestation flows, all visible and logged |
| Governance | Audit-logged consent ledger (Postgres); regulatory boundary text embedded in-app (IRDAI, SEBI) |
| Household Layer 2 | Wireframe/mockup only — no working backend |
| Fundability Layer 3 | One static slide/screen — no working backend |

### 4.2 Explicitly out of scope
See §2.2. If a feature isn't listed in §4.1, assume it's out of scope unless this PRD is revised.

---

## 5. End-to-end user flow

1. **Login** — mobile number + OTP, PIN set at signup. Session token issued.
2. **Non-financial profile capture** — household size, dependents, occupation type, self-reported existing debt/insurance (small form, <10 fields).
3. **AA consent** — redirect to AA/FIU consent flow (Anumati/Setu sandbox). Four parameters shown and logged: purpose code, fetchType, frequency, consentMode, dataLife, dataRange.
4. **Data fetch** — FI data pulled per ReBIT Deposit FI schema v2.0.0. Stored only as needed for session/analysis (VIEW-preferred; STORE only where the sandbox requires it, with short dataLife).
5. **Engine run**:
   a. Cluster CREDIT transactions by narration fingerprint + mode + amount band → build empirical arrival-day CDF per stream.
   b. Cluster recurring DEBIT transactions, tag each as reschedulable or not.
   c. Reconstruct continuous balance curve from per-transaction `currentBalance`; compute P(funded) per obligation at its scheduled timestamp.
   d. Optimiser reassigns due dates across the reschedulable subset to maximise the minimum P(funded).
   e. Coverage gate: warrant only obligations scoring above the confidence threshold post-optimisation.
6. **Insight delivery (app)** — before/after obligation calendar, plain-language explanation per moved item, warranty enrollment offer.
7. **Voice channel (optional, parallel)**:
   a. User calls in or taps "call me."
   b. Light auth: caller-ID match + spoken/DTMF PIN.
   c. Agent explains the already-computed insight in natural language.
   d. Agent answers government-scheme eligibility questions via a fixed-tool call against a static, sourced JSON dataset, matched to the user's non-financial profile fields. No raw transaction data is queried live during the call.
   e. Call ends; session token invalidated. No re-verification needed to end.
8. **Consent lifecycle demo (separate flow, always available)** — grant → fetch → pause → revoke → deletion attestation, each step timestamped and visible on screen.

---

## 6. Functional requirements

### 6.1 Engine (core differentiator — build this first and most carefully)
- **FR-1**: System shall cluster CREDIT transactions into inflow streams using narration fingerprint, `mode`, and amount band.
- **FR-2**: System shall compute an empirical CDF of arrival day per inflow stream across all observed cycles (not just a mean).
- **FR-3**: System shall cluster recurring DEBIT transactions into an obligation ledger, with a reschedulable/non-reschedulable flag per obligation type (loan EMIs: typically one date-change allowed; gig payouts: non-reschedulable by definition, they're inflows not obligations — clarify in code comments).
- **FR-4**: System shall reconstruct a continuous balance curve using per-transaction `currentBalance` (not month-end snapshots).
- **FR-5**: System shall compute P(funded) for each obligation at its currently scheduled timestamp, using the balance curve.
- **FR-6**: System shall run a due-date optimiser (greedy or exhaustive search acceptable — no ML required) that reassigns dates within the reschedulable subset to maximise the minimum P(funded) across all obligations.
- **FR-7**: System shall apply a confidence threshold (the "coverage gate") — only obligations scoring above threshold post-optimisation are eligible for warranty coverage.
- **FR-8**: System shall expose the confidence threshold as an adjustable parameter (for the demo's "loss-ratio dial" beat).
- **FR-9**: System shall support a Monte Carlo/bootstrap "what if" simulation: resample from observed inflow/obligation distributions to project outcome ranges over N future cycles, before and after a given user action.

### 6.2 Auth
- **FR-10**: Login requires mobile number + OTP + PIN.
- **FR-11**: Step-up OTP required before: granting/revoking AA consent, enrolling in the warranty, any household-layer action (even in wireframe, gate it the same way for consistency).
- **FR-12**: Voice-call sessions require caller-ID match against the account's registered number, plus a spoken or DTMF PIN, before any personalized data is surfaced.
- **FR-13**: No additional verification required on logout — session token invalidation is sufficient.

### 6.3 AA / consent
- **FR-14**: Consent request shall set purpose code 102 (default) or 104 (ongoing monitoring, if PERIODIC fetch selected), fetchType PERIODIC, frequency DAILY, consentMode VIEW (fallback to STORE only if the sandbox requires it), a finite dataLife, and a bounded dataRange.
- **FR-15**: All four consent parameters (purpose, fetchType/frequency, consentMode, dataLife/dataRange) shall be rendered visibly in the UI at consent time.
- **FR-16**: System shall support pause, revoke, and deletion-attestation actions on an active consent, each producing a timestamped audit log entry.

### 6.4 Voice channel
- **FR-17**: The voice agent shall only query two data sources: (a) pre-computed insight objects generated by the engine in §6.1, and (b) a static, sourced JSON dataset of government schemes matched against non-financial profile fields.
- **FR-18**: The voice agent shall never query raw AA transaction data directly during a call.
- **FR-19**: The voice agent's persona, tool access, and refusal boundaries shall be defined in a single source-of-truth instruction set (avoid drift between the Gemini system prompt and any separate agent configuration — see Caller Module doc §3.3(c)).
- **FR-20**: Follow-up questions on scheme eligibility shall be handled by re-invoking the same fixed-dataset tool with narrower parameters, not by open-ended retrieval.

### 6.5 Warranty
- **FR-21**: Warranty enrollment is only available for obligations that passed the coverage gate (FR-7).
- **FR-22**: System shall track and display: revenue per subscriber (₹49 × 12), claim cost per covered bounce (~₹295 incl. GST), and current loss ratio given the active confidence threshold.

### 6.6 Governance/audit
- **FR-23**: Every consent action (grant, fetch, pause, revoke, deletion) shall be written to an append-only audit log with timestamp, action type, and consent parameters at time of action.
- **FR-24**: The app shall display, at least once in the primary user flow, a plain-language statement of what is and isn't inferred from the data (the "refusal" statement — see §8.4).

---

## 7. System architecture

```
┌─────────────┐      ┌──────────────────┐      ┌────────────────────┐
│   Frontend   │◄────►│  App/API server   │◄────►│  AA sandbox          │
│ (web/app)    │      │ (auth, consent,   │      │ (Perfios/Anumati)    │
└─────────────┘      │  session mgmt)    │      └────────────────────┘
                      │        │          │
                      │        ▼          │
                      │  ┌────────────┐   │
                      │  │  Engine     │   │
                      │  │ (clustering,│   │
                      │  │ optimiser,  │   │
                      │  │ simulation) │   │
                      │  └────────────┘   │
                      │        │          │
                      │        ▼          │
                      │  ┌────────────┐   │
                      │  │ Insight     │   │
                      │  │ objects DB  │   │
                      │  └────────────┘   │
                      └──────────────────┘
                              ▲
                              │ (reads insight objects + scheme dataset only)
                      ┌──────────────────┐
                      │  Voice worker     │
                      │ (LiveKit + Gemini │
                      │  Live, per Caller │
                      │  Module doc)      │
                      └──────────────────┘
                              ▲
                      ┌──────────────────┐
                      │  Caller (phone /  │
                      │  browser call UI) │
                      └──────────────────┘
```

**Trust boundary note (for the architecture diagram deliverable):** the App/API server is the only component holding the AA client secret. The Voice worker holds no AA credentials and no direct database access to raw transactions — it only calls an internal read-only endpoint that serves pre-computed insight objects and the static scheme dataset.

### 7.1 Suggested stack
- **Frontend**: Next.js/React (reuse LiveKit's `agent-starter-react` template for the call UI portion per the Caller Module doc; a separate lightweight dashboard for insight cards).
- **App/API server**: Node.js, holds AA credentials, issues JWTs, manages consent lifecycle, exposes the read-only insight endpoint for the voice worker.
- **Engine**: Python or Node — clustering, CDF construction, optimiser, Monte Carlo simulation. No ML model required for v1 (explicit non-goal — keep it explainable-by-construction).
- **Voice worker**: LiveKit Agents (Node), Gemini Live native-audio model (`gemini-2.5-flash-native-audio-preview-12-2025` or current equivalent — verify against Google's current model list before building, as names change), Silero VAD, LiveKit multilingual turn detector.
- **Data store**: Postgres — insight objects, consent audit ledger, minimal user profile. No raw transaction persistence beyond session unless the sandbox requires STORE mode.
- **Deployment**: Docker for frontend and backend, per existing LiveKit template Dockerfiles. Self-hosted LiveKit + local Node processes is sufficient for the demo; LiveKit Cloud only if multilingual turn detection/noise cancellation are needed and time permits.

---

## 8. Regulatory and governance requirements

### 8.1 Data protection
- DPDP Act 2023: purpose limitation and data minimisation apply to every data category collected. Each field collected in the non-financial profile (§5 step 2) must map to a stated purpose (household size → scheme eligibility matching; occupation → same).
- Consent spans two regimes: AA consent for financial data, DPDP-style consent for the non-financial profile and call recording/transcription (if any). Do not conflate them in the UI — show them as separate, clearly-labeled consents.

### 8.2 AA-specific
- Purpose code 102 ("customer spending patterns, budget or other reportings") is explicitly not a lending code — do not use Crunch's output to make or imply a lending decision without switching to an appropriate purpose code and disclosure.
- `consentMode: VIEW` is preferred over `STORE`; use `STORE` only where the sandbox mandates it, and set the shortest workable `dataLife`.

### 8.3 Financial advice boundary (SEBI)
- The app and voice agent must never recommend specific securities, mutual funds, or specific allocations. Permitted: descriptive analytics ("you are paying more in EMI interest than your relative earns on savings"), generic education (what an intra-family loan or promissory note is), and referral to a licensed distributor/registered investment adviser for anything resembling a recommendation. This boundary should be enforced in code (a checklist/filter on generated agent responses, not just a prompt instruction) as well as in writing.

### 8.4 Warranty product boundary (IRDAI)
- The ₹49/month product must be framed and implemented as a **service-level remedy** (crediting fees, reimbursing the levied charge) — not risk pooling or an indemnity contract, which would trigger insurance regulation. Any copy referring to the warranty ("we pay the charge") must be reviewed against this framing before it ships.

### 8.5 Refusal statement
- The product must state, at least once in the primary flow, that it models the household's money and not its life — e.g., it does not infer health, relationship status, or anything beyond financial timing/obligations from the transaction data. Implement this as a persistent, visible statement (not just a slide).

---

## 9. Non-functional requirements

- **Explainability**: every automated recommendation (a moved due date, a warranty coverage decision, a scheme match) must have an accompanying plain-language reason, generated deterministically from the same inputs the engine used — not a separate free-text generation step that could drift from the actual computation.
- **Auditability**: every consent state change and every warranty coverage decision must be logged with enough detail to reconstruct why it happened.
- **Latency**: voice responses should feel conversational (this is inherent to using Gemini Live's native audio path rather than a chained STT→LLM→TTS pipeline — do not introduce an intermediate pipeline).
- **Resilience**: engine should degrade gracefully with thin data (Caller Module doc's honest-limits precedent — e.g., if fewer than ~6 inflow cycles are observed, surface the CDF as low-confidence rather than fabricating precision).

---

## 10. Build plan (8 days)

| Day | Deliverable |
|---|---|
| Mon | Pull sandbox link, fetch one FI response, confirm: is `transactionTimestamp` real time-of-day or stubbed; is per-transaction `currentBalance` populated; how many months of history are returned. |
| Tue | Optimiser v1 on real sandbox data; first before/after on one account. |
| Wed | Backtest harness + accuracy number. Stub the fixed government-scheme dataset and its tool-call interface (~100 lines). Send lender-signal outreach emails. |
| Thu | Consent lifecycle end-to-end (grant/pause/revoke/deletion). Wire voice-call light auth (caller-ID + PIN). Confirm voice worker only reads pre-computed insight objects. Architecture/trust-boundary diagram. |
| Fri | Record real-household clip. Write regulatory/risk one-pager (include §8.3 and §8.4 boundary paragraphs) and business-model one-pager. |
| Sat | Deck spine. Household Layer 2 wireframe only. Fundability Layer 3: one slide. |
| Sun | Full 20-minute run-through with a stopwatch. Cut anything that doesn't fit — if the voice-channel beat doesn't land clean, reduce it to one scheme-eligibility example. Record backup demo video. |
| Mon | Adversarial Q&A rehearsal. Submit deck. |

---

## 11. Success metrics for the build (not the business)

- Live, on-sandbox-data demonstration of consent → fetch → engine output, with a real before/after reschedule.
- A stated, honest accuracy number (e.g., median arrival-day error, % of covered obligations funded) — including where it fails.
- Full consent lifecycle demonstrated live, all four consent parameters visible on screen.
- Voice call completed live (or via backup recording) demonstrating both insight explanation and one scheme-eligibility lookup, without any raw-transaction query visible in logs during the call.
- Regulatory one-pager contains written SEBI and IRDAI boundary paragraphs, not just verbal answers.

---

## 12. Open items requiring a decision before/during build

- Confirm current Gemini Live model name/availability (model identifiers in the Caller Module doc are illustrative and should be re-verified against Google's live documentation at build time).
- Confirm the Perfios/Anumati sandbox's actual data depth (per Mon's task) — if `currentBalance` is not populated per-transaction, FR-4/FR-5 need a fallback (e.g., interpolate from summary balances, and disclose this as a limitation).
- Decide whether the household-facing app is a full web app or a slide-deck-quality clickable prototype — given the 8-day window, a clickable prototype is acceptable per the brief's scope ("working prototype or demonstrable service flow in wireframes/figma flows").

---

## Appendix: Key sources and constraints referenced in this PRD
- ReBIT Deposit FI schema, v2.0.0 (23 Jan 2025) — transaction/summary field definitions.
- Setu AA consent object spec — purpose codes, fetchType, frequency, dataLife, consentMode.
- RBI penal charges circular — penal charges not penal interest, no capitalisation, effective 1 April 2024.
- DPDP Act 2023 — purpose limitation, data minimisation, storage limitation.
- SEBI (Investment Advisers) Regulations, 2013 — registration requirement for personalized investment advice.
- Sahamati guardrails: https://sahamati.org.in/
- CDPG Annual Conclave 2026 Stage II brief (rubric, deliverables, scope table).
- Internal Study Brief and Caller Module docs (architecture and engine detail this PRD consolidates).
