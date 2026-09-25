# TechSpecifications.md — Crunch

Companion to `PRD.md` (what/why), `ImplementationPlan.md` (task sequence), `AppFlow.md` (screens/states), and `Rules.md` (constraints). This document is the "exact how" — algorithms with formulas, request/response schemas at field level, config values, and non-functional specs. Where this document and `ImplementationPlan.md` disagree on a detail, this one wins; update the other to match.

---

## 1. Stack and versions

| Component | Choice | Notes |
|---|---|---|
| Backend runtime | Node.js 22 | matches LiveKit template Dockerfile baseline |
| Backend framework | Fastify (or Express if faster to scaffold) | plain REST, no GraphQL needed for this scope |
| Engine language | TypeScript, co-located in `backend/src/engine/` | no separate service unless profiling shows a real need |
| Voice worker | LiveKit Agents SDK (Node), forked from `agent-starter-node` | |
| Realtime model | Gemini Live native-audio — **verify exact current model string before building**; do not hardcode a model name from an old doc without checking Google's current model list | Temperature 0.8 as a starting point (from Caller Module doc); tune if latency/naturalness suffers |
| VAD | Silero VAD, preloaded at worker init | per LiveKit template default |
| Turn detection | LiveKit multilingual turn detector | Cloud-dependent feature — confirm availability if self-hosting |
| Frontend | Next.js (React), forked from `agent-starter-react` for `/call`, custom routes for `/dashboard` etc. | |
| Database | PostgreSQL 15+ | single instance sufficient for demo scale |
| Package manager | pnpm, frozen-lockfile installs | matches LiveKit template Dockerfiles |
| Containerization | Docker, separate images for backend, voice-worker, frontend | per `ImplementationPlan.md` §1 structure |

---

## 2. Engine specification

### 2.1 Inflow stream clustering `[FR-1]`
- **Input**: all `CREDIT` transactions from the FI response.
- **Fingerprint key**: `hash(normalized_narration) + mode + amount_band`, where:
  - `normalized_narration` = narration string lowercased, with transaction-specific reference numbers (e.g. UTR numbers, UPI transaction IDs) stripped via regex before hashing, so recurring counterparties cluster together despite unique per-transaction IDs.
  - `amount_band` = `floor(amount / 500) * 500` (₹500 bands; tune if bands are too coarse/fine on real data).
- **Output**: one `InflowStream` per distinct fingerprint with ≥2 observed occurrences. Single-occurrence credits are not treated as streams (nothing to build a CDF from).

### 2.2 Empirical arrival-day CDF `[FR-2]`
- For each stream, collect the day-of-cycle (day-of-month, or day-offset-from-previous-occurrence if the cycle isn't monthly) for every observed occurrence.
- Build the empirical CDF directly from these observations — **no parametric fitting** (no assumed normal/lognormal distribution). This is a deliberate simplicity choice: it's explainable by construction and doesn't risk misrepresenting a small sample as well-characterized.
- `cyclesObserved` = count of occurrences. `lowConfidence = cyclesObserved < 6`.

### 2.3 Obligation ledger `[FR-3]`
- **Input**: all `DEBIT` transactions, clustered the same way as §2.1.
- **Reschedulable flag**, by `type`:
  - `loan_emi`: `reschedulable = true`, but capped — see §2.5 optimiser constraint (typically one date-change permitted).
  - `sip`: `reschedulable = true`.
  - `utility_mandate`: `reschedulable = true` if the narration matches a known utility-provider pattern with a published date-change mechanism; otherwise `false` (default to `false` when uncertain — do not assume rescheduling is possible without evidence).
  - `other`: `reschedulable = false` by default.

### 2.4 Balance curve reconstruction `[FR-4]`
- **Primary path**: use `currentBalance` from each transaction directly, sorted by `transactionTimestamp` (or `valueDate` if timestamp is stubbed — see Phase 0 finding), to build a step function of balance over time.
- **Fallback path** (if `currentBalance` is not populated per-transaction): reconstruct from month-end/period summary balances plus transaction amounts, interpolating linearly between known balance points. Set `degradedMode: true` on any `InsightObject` computed this way.
- **Funding probability** `[FR-5]`: for obligation `o` scheduled at time `t`, `P(funded) = P(balance(t) >= o.amount)`, estimated from the reconstructed balance curve's historical behavior around comparable points in prior cycles (not a single-point balance check — the point is to capture the *distribution* of balance at that time-of-cycle, matching the inflow-CDF philosophy in §2.2).

### 2.5 Optimiser `[FR-6, FR-7, FR-8]`
- **Objective**: maximise `min(P(funded))` across all obligations, by reassigning `optimisedDay` for each reschedulable obligation.
- **Search space per obligation**: a small, bounded set of candidate days (e.g. ±5 days from original, or constrained further by known lender rules — one EMI date-change, SIP date flexibility per AMC rules).
- **Method**: exhaustive search if the combined search space is small (a handful of reschedulable obligations × small day-range is tractable); fall back to greedy (reassign the worst-P(funded) obligation first, holding others fixed, iterate) if the space grows too large. No ML — this must stay explainable by construction.
- **Coverage gate**: `aboveCoverageGate = optimisedPFunded >= confidenceThreshold`. `confidenceThreshold` is a tunable parameter (default suggestion: 0.85 — tune against backtest results, don't hardcode a number that hasn't been validated against real data).

### 2.6 Monte Carlo simulation `[FR-9]`
- Resample `N` future cycles (default `N = 1000` simulation runs, tune for demo-time performance) by drawing from each stream's empirical arrival-day distribution (§2.2) and each obligation's amount distribution.
- For each simulated cycle, compute whether each obligation would have been funded under (a) the original schedule and (b) the optimised schedule.
- Output: a distribution of outcomes ("funded in X% of simulated cycles") for the before/after comparison — this is the basis of the "what if" demo beat, not a single deterministic number.

### 2.7 Explanation generation `[FR-24 support, NFR]`
- `explain.ts` takes the same inputs used in §2.4–2.5 and renders a template string, e.g.:
  > "Your {obligation.type} of ₹{amount} was scheduled for day {originalDay}, when funds were available only {originalPFunded*100}% of the time. Moving it to day {optimisedDay} — after your typical {stream} arrival — raises that to {optimisedPFunded*100}%."
- No LLM call in this path. Keep it deterministic so the number in the explanation always matches the number the optimiser actually used.

---

## 3. API specification (field-level)

### 3.1 `POST /auth/otp/request`
```json
// request
{ "phone": "+91XXXXXXXXXX" }
// response 200
{ "otpSent": true, "expiresInSeconds": 300 }
// response 429 (too many requests)
{ "error": "rate_limited", "retryAfterSeconds": 60 }
```

### 3.2 `POST /auth/otp/verify`
```json
// request
{ "phone": "+91XXXXXXXXXX", "otp": "123456", "pin": "XXXX" }
// response 200
{ "sessionToken": "jwt...", "isNewUser": true }
// response 401
{ "error": "invalid_otp", "attemptsRemaining": 2 }
```

### 3.3 `POST /consent/grant`
```json
// request
{ "purposeCode": "102", "fetchType": "PERIODIC", "frequency": "DAILY",
  "consentMode": "VIEW", "dataLife": "P6M", "dataRange": { "from": "2026-01-01", "to": "2026-09-26" } }
// response 200
{ "consentId": "uuid", "redirectUrl": "https://aa-sandbox.../consent/uuid" }
```
`dataLife` uses ISO 8601 duration format (e.g. `P6M` = 6 months) — keep this finite, never `INF`, per PRD §8.2.

### 3.4 `GET /insight/:userId` (called by frontend with session auth, or voice worker with service token)
```json
// response 200
{
  "userId": "uuid",
  "generatedAt": "2026-09-26T10:00:00Z",
  "obligations": [
    {
      "obligationId": "uuid",
      "type": "sip",
      "scheduledDay": 3,
      "amount": 2000,
      "reschedulable": true,
      "originalPFunded": 0.42,
      "optimisedDay": 6,
      "optimisedPFunded": 0.91,
      "aboveCoverageGate": true,
      "explanation": "..."
    }
  ],
  "degradedMode": false,
  "summaryText": "We moved 1 of 4 payments this cycle to reduce bounce risk."
}
```
**This endpoint never returns raw transaction records — only the `InsightObject` shape.** This is the field-level enforcement point for `Rules.md` §1 rule 6.

### 3.5 `POST /schemes/match` (voice worker only, service token)
```json
// request
{ "incomeBand": "0-30000", "occupation": "gig_worker", "familySize": 4 }
// response 200
{ "matches": [ { "schemeId": "...", "name": "...", "nextSteps": "...", "sourceUrl": "..." } ] }
```

### 3.6 Error format (all endpoints)
```json
{ "error": "error_code_string", "message": "human-readable, safe to display", "retryable": true }
```
No stack traces or internal identifiers in any error response body, per `AppFlow.md` §1.10.

---

## 4. Database specification

Extends the schema in `ImplementationPlan.md` §3.6 with indices and constraints:

```sql
CREATE INDEX idx_consent_audit_user ON consent_audit(user_id, created_at DESC);
CREATE INDEX idx_insight_objects_user ON insight_objects(user_id, generated_at DESC);
CREATE INDEX idx_warranty_user ON warranty_enrollments(user_id, status);

ALTER TABLE users ADD CONSTRAINT phone_format CHECK (phone ~ '^\+91[0-9]{10}$');
ALTER TABLE consent_audit ADD CONSTRAINT valid_action
  CHECK (action IN ('grant','fetch','pause','revoke','deletion_attested'));
ALTER TABLE consent_audit ADD CONSTRAINT valid_consent_mode
  CHECK (consent_mode IN ('VIEW','STORE'));
```
Raw transaction payloads, if temporarily held during a fetch for engine processing, must not be persisted to any table beyond the request lifecycle unless `consentMode = STORE` was explicitly granted — and even then, only for the duration of `dataLife`. Add a scheduled job to purge any such data past `dataLife` if `STORE` mode is ever used.

---

## 5. Voice worker configuration

```ts
// indicative agent session config — verify field names against current LiveKit Agents SDK docs at build time
const session = new voice.AgentSession({
  model: "<verify current Gemini Live native-audio model string>",
  voice: "Charon",
  temperature: 0.8,
  vad: sileroVAD,
  turnDetector: multilingualTurnDetector,
});
session.registerTool("getInsightSummary", getInsightSummaryHandler);
session.registerTool("matchSchemes", matchSchemesHandler);
```
- `getInsightSummaryHandler` and `matchSchemesHandler` call the backend's `/insight/:userId` and `/schemes/match` endpoints respectively, authenticated with `VOICE_WORKER_SERVICE_TOKEN` — no other backend routes are registered as callable tools.
- Persona instructions (single source of truth per `Rules.md` §2) should explicitly state, in the system prompt: the agent discusses money and scheduling only, never health/relationships/other personal-life inferences (PRD §8.5); the agent never names specific investment instruments (PRD §8.3); the agent states uncertainty rather than guessing when a question falls outside `getInsightSummary`/`matchSchemes` results.

---

## 6. Security specifications

| Item | Spec |
|---|---|
| Session JWT TTL | 30 minutes idle timeout, sliding on activity |
| Step-up OTP validity | 5 minutes, single-use |
| OTP length/format | 6-digit numeric, 5-minute expiry, max 3 attempts before 60-second lockout |
| PIN | 4–6 digit, hashed (bcrypt or argon2) before storage, never logged |
| `VOICE_WORKER_SERVICE_TOKEN` | scoped token, distinct from user session JWTs, restricted at API-gateway middleware to `/insight/*` and `/schemes/match` only |
| Transport | HTTPS/TLS for all API traffic; LiveKit's own WebRTC encryption for call media |
| Secrets | `AA_SANDBOX_CLIENT_SECRET`, `LIVEKIT_API_SECRET`, `JWT_SIGNING_SECRET`, `GOOGLE_API_KEY` — environment variables only, never committed, never sent to frontend |
| PIN/OTP in logs | Explicitly excluded from all logging (redact before any `console.log`/logger call) |

---

## 7. Performance and scale notes (demo-scale, not production-scale)

- Engine run (clustering → optimiser) should complete in low single-digit seconds on one household's data (a few months of transactions) — if it doesn't, profile `optimiser.ts`'s search space before optimizing elsewhere.
- Monte Carlo `N = 1000` runs should complete well under a second for a handful of obligations — increase `N` only if demo timing allows, since it doesn't change the explanation, only the confidence of the displayed range.
- No load-testing or multi-tenant scale target is in scope for this build (per `PRD.md` §2.2 non-goals) — a single concurrent demo user is the actual requirement.

---

## 8. Testing specification

- **Unit tests**: `clustering.ts`, `cdf.ts`, `balance.ts`, `funding.ts`, `optimiser.ts` each need direct tests against the Phase 0 fixture, independent of the live sandbox.
- **Backtest**: hold out the most recent observed cycle per stream, run the full pipeline on the remainder, compare predicted `P(funded)` against actual outcome. Report both the accuracy number and the failure cases explicitly (per `ImplementationPlan.md` Phase 1 and `Rules.md` §5).
- **Integration test**: one full run of consent-grant → fetch → engine → insight-object-served, against the live sandbox, not just the fixture.
- **Voice worker structural test**: automated check (or, at minimum, a documented manual check) that `VOICE_WORKER_SERVICE_TOKEN` cannot successfully call any `/aa/*` route — this should fail loudly (401/403) if attempted, and that failure should itself be tested.
