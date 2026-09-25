# Implementation Plan — Crunch
Companion to `PRD.md`. This document is written for whoever (or whichever agent) writes the code — it breaks the PRD's functional requirements into ordered, checkable engineering tasks, with file structure, data shapes, and API contracts. Follow phases in order; each phase has a "done when" checkpoint before moving on.

---

## 0. How to use this document

- Work phase by phase. Do not start Phase N+1 until Phase N's "done when" checkpoint is met.
- Every data shape in §3 is the source of truth — the engine, API, and voice worker all read/write these exact shapes, so changing one means updating all three.
- FR numbers in brackets (e.g. `[FR-6]`) reference `PRD.md` §6 — keep them in code comments on the corresponding function so the mapping stays traceable for the regulatory one-pager and for judge Q&A.
- If sandbox data doesn't match an assumption below (see PRD §12), stop and resolve it before continuing — don't silently work around a missing field.

---

## 1. Repository structure

```
crunch/
├── backend/
│   ├── src/
│   │   ├── server.ts               # API server entrypoint, session/auth
│   │   ├── auth/
│   │   │   ├── otp.ts               # OTP generation/verification [FR-10, FR-11]
│   │   │   └── session.ts           # JWT issuance/validation
│   │   ├── aa/
│   │   │   ├── consent.ts           # AA consent object creation [FR-14, FR-15]
│   │   │   ├── fetch.ts             # FI data fetch from sandbox
│   │   │   └── lifecycle.ts         # pause/revoke/deletion [FR-16]
│   │   ├── engine/
│   │   │   ├── clustering.ts        # inflow/obligation clustering [FR-1, FR-3]
│   │   │   ├── cdf.ts               # empirical arrival-day CDF [FR-2]
│   │   │   ├── balance.ts           # balance curve reconstruction [FR-4]
│   │   │   ├── funding.ts           # P(funded) scoring [FR-5]
│   │   │   ├── optimiser.ts         # due-date reassignment [FR-6, FR-7, FR-8]
│   │   │   ├── simulate.ts          # Monte Carlo what-if [FR-9]
│   │   │   └── explain.ts           # deterministic explanation generator (§9 NFR)
│   │   ├── warranty/
│   │   │   └── warranty.ts          # enrollment + loss-ratio tracking [FR-21, FR-22]
│   │   ├── audit/
│   │   │   └── log.ts               # append-only consent/warranty audit log [FR-23]
│   │   ├── insight/
│   │   │   └── insightStore.ts      # read-only insight object store, served to voice worker
│   │   ├── schemes/
│   │   │   └── schemes.json         # static, sourced government-scheme dataset
│   │   └── db/
│   │       └── schema.sql           # Postgres schema (see §3)
│   ├── Dockerfile
│   └── package.json
├── voice-worker/                    # forked/adapted from LiveKit agent-starter-node
│   ├── src/
│   │   └── agent.ts                 # persona, Gemini Live wiring, tool calls [FR-17–FR-20]
│   ├── Dockerfile
│   └── package.json
├── frontend/                        # adapted from LiveKit agent-starter-react + insight dashboard
│   ├── app/
│   │   ├── (app)/dashboard/         # insight cards, warranty, consent lifecycle UI
│   │   └── (app)/call/              # call UI, reuses LiveKit template components
│   └── package.json
└── docs/
    ├── PRD.md
    ├── ImplementationPlan.md        # this file
    └── architecture-diagram.*       # trust-boundary diagram deliverable
```

---

## 2. Phased build sequence

### Phase 0 — Setup (Mon, before anything else)
- [ ] Clone/scaffold `backend` from a plain Node/Express or Fastify starter (not the LiveKit template — that's for `voice-worker`/`frontend` call UI only).
- [ ] Scaffold `voice-worker` from LiveKit's `agent-starter-node`.
- [ ] Scaffold `frontend` from LiveKit's `agent-starter-react`, then add a `/dashboard` route for insight cards outside the call flow.
- [ ] Set up Postgres locally (or a hosted dev instance); apply `schema.sql` (§3.6).
- [ ] Pull the Perfios/Anumati sandbox link. Fetch one FI response and answer the three open questions from PRD §12:
  1. Is `transactionTimestamp` real time-of-day or stubbed to `00:00:00`?
  2. Is per-transaction `currentBalance` populated?
  3. How many months of history come back?
- [ ] Record the answers in `docs/sandbox-findings.md` — this determines whether `balance.ts` needs an interpolation fallback (see Phase 1 note).

**Done when:** one real FI response is saved locally as a fixture (`backend/src/engine/__fixtures__/sample-fi-response.json`) and the three questions above are answered in writing.

### Phase 1 — Engine core (Tue–Wed)
Build against the fixture from Phase 0, not live sandbox calls yet — that keeps iteration fast.

- [ ] `clustering.ts`: group CREDIT transactions by narration fingerprint + `mode` + amount band into inflow streams `[FR-1]`. Group recurring DEBITs into an obligation ledger with a reschedulable flag `[FR-3]`.
- [ ] `cdf.ts`: for each inflow stream, build the empirical arrival-day distribution across observed cycles `[FR-2]`. If fewer than ~6 cycles observed, tag the stream `low_confidence: true` (§9 NFR — no fabricated precision).
- [ ] `balance.ts`: reconstruct a continuous balance curve from per-transaction `currentBalance` `[FR-4]`. **If Phase 0 found `currentBalance` is not populated per-transaction**, implement the interpolation fallback here and log a `degraded_mode: true` flag on the output — this must be visible in the UI, not hidden.
- [ ] `funding.ts`: score P(funded) per obligation at its scheduled timestamp using the balance curve `[FR-5]`.
- [ ] `optimiser.ts`: greedy or exhaustive search reassigning due dates within the reschedulable subset to maximise minimum P(funded) `[FR-6]`. Apply the confidence threshold as the coverage gate `[FR-7]`, expose it as a parameter `[FR-8]`.
- [ ] `explain.ts`: for every optimiser decision, generate the explanation string deterministically from the same inputs used in the decision — no separate free-text call.
- [ ] Write a backtest harness: hold out the most recent cycle, run the pipeline on the rest, compare predicted vs. actual funding outcome. Produce one honest accuracy number (e.g., median arrival-day error, % of covered obligations funded) — this feeds the regulatory/demo one-pagers, not just the code.

**Done when:** running the pipeline on the Phase 0 fixture produces a before/after obligation calendar with explanations, and the backtest harness prints an accuracy number (including its failure rate).

### Phase 2 — Auth, consent, API server (Wed–Thu)
- [ ] `otp.ts` + `session.ts`: mobile OTP + PIN login, JWT session issuance `[FR-10]`. Step-up OTP middleware for consent grant/revoke and warranty enrollment routes `[FR-11]`.
- [ ] `consent.ts`: build the AA consent request object with purpose code 102/104, fetchType PERIODIC, frequency DAILY, consentMode VIEW-preferred, finite dataLife, bounded dataRange `[FR-14]`. Render all parameters back to the frontend for display `[FR-15]`.
- [ ] `fetch.ts`: call the sandbox's FI data endpoint using the granted consent, store the response only as needed (respect VIEW vs STORE) `[FR-14]`.
- [ ] `lifecycle.ts` + `log.ts`: pause, revoke, deletion-attestation actions, each writing a timestamped row to the audit log `[FR-16, FR-23]`.
- [ ] Wire Phase 1's engine to run automatically after a successful fetch; persist the resulting insight objects (§3.3) to `insightStore.ts`.
- [ ] `schemes.json`: populate with a small number (5–15) of real, sourced government schemes, each tagged with matching criteria (income band, occupation, family size, etc.) `[FR-17, part 2]`.

**Done when:** a full consent → fetch → engine → insight-object flow runs end-to-end against the live sandbox (not just the fixture), and pause/revoke/deletion each produce a visible audit-log entry.

### Phase 3 — Voice channel (Thu)
- [ ] `agent.ts`: define a single persona/instruction source of truth (no drift between Gemini system prompt and any separate agent config) `[FR-19]`.
- [ ] Register two tools only:
  - `getInsightSummary(userId)` → reads from `insightStore.ts` (read-only, no AA credentials in this process) `[FR-17a, FR-18]`.
  - `matchSchemes(profileFields)` → queries `schemes.json` `[FR-17b, FR-20]`.
- [ ] Implement call-time light auth: caller-ID match + spoken/DTMF PIN before either tool is invoked `[FR-12]`.
- [ ] Confirm in code review (not just by testing) that no code path in `voice-worker` can reach raw transaction data or AA credentials — this is the claim you'll defend live, so it needs to be structurally true, not just usually true.
- [ ] Wire the frontend `/call` route to LiveKit room join per the standard template flow.

**Done when:** a live call authenticates via PIN, explains a real insight object in natural language, and answers a scheme-eligibility follow-up via `matchSchemes` — confirmed by checking worker logs show zero calls to any AA/transaction endpoint during the session.

### Phase 4 — Warranty, dashboard, governance surface (Fri)
- [ ] `warranty.ts`: enrollment gated on coverage-gate pass `[FR-21]`; track revenue/claim-cost/loss-ratio and expose as an adjustable-threshold view for the demo `[FR-22]`.
- [ ] Frontend `/dashboard`: insight cards (before/after, explanations), warranty status, consent lifecycle controls, and the persistent refusal statement `[FR-24]`.
- [ ] Add the regulatory boundary copy (SEBI §8.3, IRDAI §8.4 language from PRD) as static, reviewed text in both the app and the written one-pagers — do not let the voice agent free-generate this language.
- [ ] If time allows: implement a response filter/checklist on agent output that blocks anything resembling specific-instrument investment advice (§8.3) — a simple keyword/pattern check plus system-prompt constraint is enough for this build; it doesn't need to be sophisticated, it needs to be demonstrably present.

**Done when:** the dashboard shows a working warranty view with a movable confidence-threshold slider that visibly changes the loss ratio, and the refusal + regulatory boundary text is present in-app, not only in slides.

### Phase 5 — Household/fundability placeholders, architecture diagram (Fri–Sat)
- [ ] Household Layer 2: static wireframe screens only (Figma or a non-functional React route) — do not connect a second AA consent or build real household-arbitrage logic.
- [ ] Fundability Layer 3: one static screen/slide showing the "exportable consented proof" concept — no export logic needs to work.
- [ ] Produce the architecture/trust-boundary diagram (docs/architecture-diagram.*) reflecting §7 of the PRD, specifically showing that the voice worker has no path to AA credentials or raw transactions.

**Done when:** both wireframes exist and are presentable, and the diagram accurately reflects the actual code (not an aspirational version of it).

### Phase 6 — Full run-through and hardening (Sun–Mon)
- [ ] Run the entire demo flow start to finish with a stopwatch against the 20-minute structure in PRD §6/demo doc. Cut anything that doesn't fit — the PRD's rule stands: build one, wireframe one, say one.
- [ ] Record a full backup screen-capture of the live demo in case the live version fails on the day.
- [ ] Run an adversarial Q&A pass: have someone try to break the "voice channel never touches raw data" claim, the SEBI boundary, and the loss-ratio arithmetic. Fix anything that doesn't hold up under a direct question.

**Done when:** the timed run-through fits in 20 minutes, the backup recording exists, and every claim in §11 of the PRD ("Success metrics for the build") can be pointed at in the running app.

---

## 3. Data shapes (source of truth — keep engine/API/voice worker in sync)

### 3.1 InflowStream
```ts
{
  streamId: string;
  narrationFingerprint: string;
  mode: "CASH" | "ATM" | "CARD" | "UPI" | "FT" | "OTHERS";
  amountBand: [number, number];
  arrivalDayCDF: { day: number; cumulativeProbability: number }[];
  cyclesObserved: number;
  lowConfidence: boolean;   // true if cyclesObserved < 6
}
```

### 3.2 Obligation
```ts
{
  obligationId: string;
  type: "loan_emi" | "sip" | "utility_mandate" | "other";
  scheduledDay: number;
  amount: number;
  reschedulable: boolean;
  originalPFunded: number;
  optimisedDay: number | null;   // null if not moved
  optimisedPFunded: number | null;
  aboveCoverageGate: boolean;
  explanation: string;           // deterministic, from explain.ts
}
```

### 3.3 InsightObject (what the voice worker reads)
```ts
{
  userId: string;
  generatedAt: string;          // ISO timestamp
  obligations: Obligation[];
  degradedMode: boolean;        // true if balance interpolation fallback was used
  summaryText: string;          // short, pre-generated plain-language summary
}
```

### 3.4 ConsentRecord (audit ledger row)
```ts
{
  consentId: string;
  userId: string;
  action: "grant" | "fetch" | "pause" | "revoke" | "deletion_attested";
  purposeCode: "102" | "104";
  fetchType: "PERIODIC" | "ONETIME";
  frequency: string;
  consentMode: "VIEW" | "STORE";
  dataLife: string;
  dataRange: { from: string; to: string };
  timestamp: string;
}
```

### 3.5 SchemeRecord (static dataset)
```ts
{
  schemeId: string;
  name: string;
  sourceUrl: string;             // must be a real gov.in or official source
  eligibility: {
    incomeBandMax?: number;
    occupation?: string[];
    familySizeMin?: number;
    // add fields as needed, keep minimal
  };
  nextSteps: string;              // what the user does after eligibility match
}
```

### 3.6 Postgres schema (minimum viable)
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY,
  phone TEXT UNIQUE NOT NULL,
  pin_hash TEXT NOT NULL,
  profile JSONB NOT NULL DEFAULT '{}'
);

CREATE TABLE consent_audit (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  action TEXT NOT NULL,
  purpose_code TEXT,
  fetch_type TEXT,
  frequency TEXT,
  consent_mode TEXT,
  data_life TEXT,
  data_range_from DATE,
  data_range_to DATE,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE insight_objects (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  payload JSONB NOT NULL,
  generated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE warranty_enrollments (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  obligation_id TEXT NOT NULL,
  enrolled_at TIMESTAMPTZ DEFAULT now(),
  status TEXT NOT NULL DEFAULT 'active'
);
```

---

## 4. API contract (backend, consumed by frontend and voice worker)

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/auth/otp/request` | none | send OTP to phone |
| POST | `/auth/otp/verify` | none | verify OTP, return session JWT |
| POST | `/auth/step-up` | session | re-verify via OTP for high-stakes actions |
| POST | `/consent/grant` | session + step-up | create AA consent, redirect to AA flow |
| POST | `/consent/pause` | session + step-up | pause active consent |
| POST | `/consent/revoke` | session + step-up | revoke active consent |
| POST | `/consent/delete` | session + step-up | request deletion, log attestation |
| POST | `/aa/fetch` | session | trigger FI fetch under active consent |
| GET | `/insight/:userId` | session (frontend) OR internal service token (voice worker) | fetch latest InsightObject — **read-only, no raw transactions returned** |
| POST | `/warranty/enroll` | session + step-up | enroll an eligible obligation |
| GET | `/warranty/status` | session | current loss-ratio, coverage, threshold |
| POST | `/schemes/match` | internal service token (voice worker) | match static scheme dataset against profile fields |

**Critical constraint:** the voice worker's service token must only grant access to `/insight/:userId` and `/schemes/match`. It must not have credentials for `/aa/*` or any endpoint that returns raw transactions. Enforce this at the API-gateway/middleware level, not just by convention — this is the structural claim Phase 3 needs to hold under a hostile question.

---

## 5. Environment / config checklist

- `AA_SANDBOX_CLIENT_ID`, `AA_SANDBOX_CLIENT_SECRET` — backend only, never exposed to frontend or voice worker.
- `GOOGLE_API_KEY` — voice worker only, for Gemini Live.
- `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` — backend (token minting) only; browser never receives these.
- `JWT_SIGNING_SECRET` — backend only.
- `DATABASE_URL` — backend only.
- `VOICE_WORKER_SERVICE_TOKEN` — scoped as described in §4, issued by backend, used by voice worker to call `/insight` and `/schemes/match` only.

---

## 6. Definition of done (whole project)

- [ ] All FR items in PRD §6 are implemented or explicitly deferred with a written reason.
- [ ] Backtest accuracy number exists and is stated honestly, including failure rate.
- [ ] Consent lifecycle fully demonstrable live: grant → fetch → pause → revoke → deletion attestation.
- [ ] Voice call demonstrably cannot reach raw transaction data (verified structurally, not just by testing).
- [ ] Warranty loss-ratio dial works and is tied to the actual coverage-gate threshold, not a hardcoded number.
- [ ] Regulatory boundary text (SEBI, IRDAI) is in-app, not just in slides.
- [ ] Refusal statement is in-app.
- [ ] 20-minute run-through timed and fits, with a recorded backup.
