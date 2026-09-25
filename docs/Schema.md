# Schema.md — Crunch

This is the single canonical reference for every data shape in the system. `PRD.md`, `ImplementationPlan.md`, `AppFlow.md`, and `TechSpecifications.md` all reference these shapes — if any of them shows a shape that disagrees with this file, **this file wins** and the other should be updated to match. This file exists because `Rules.md` §2 says data shapes are shared contracts across services; this is where that contract actually lives.

---

## 0. Conventions

- All timestamps: ISO 8601, UTC (`2026-09-26T10:00:00Z`).
- All durations (e.g. `dataLife`): ISO 8601 duration format (`P6M` = 6 months).
- All monetary amounts: `number`, in rupees, no paise subdivision unless a field explicitly says otherwise.
- All IDs: UUID v4 strings unless noted.
- Enum-like string fields are written as TypeScript union types below — treat them as closed sets; adding a new value is a schema change, not a free-text field.
- "Consumed by" in each table below tells you which services read/write that shape — check it before assuming a shape is private to one service.

---

## 1. External schema — ReBIT Deposit FI (v2.0.0), fields Crunch actually consumes

Crunch does not consume the full ReBIT schema — only these fields, per transaction:

| Field | Type | Used by | Notes |
|---|---|---|---|
| `transactionTimestamp` | `xs:dateTime` | `clustering.ts`, `balance.ts` | Phase 0 must confirm this carries real time-of-day, not `00:00:00` stub |
| `valueDate` | `xs:date` | `balance.ts` (fallback ordering if timestamp is stubbed) | |
| `type` | `CREDIT \| DEBIT` | `clustering.ts` | splits inflow streams from obligation ledger |
| `mode` | `CASH \| ATM \| CARD \| UPI \| FT \| OTHERS` | `clustering.ts`, profile inference | `CASH` is the informal-income signal referenced in `PRD.md` §1 |
| `amount` | `xs:float` | all engine modules | |
| `currentBalance` | `xs:string` (parse to number) | `balance.ts` | primary path requires this per-transaction; see §5 fallback |
| `narration` | `xs:string` | `clustering.ts` | stripped of per-transaction reference numbers before fingerprinting — see `TechSpecifications.md` §2.1 |

No other ReBIT fields are read by Crunch in this build. If a future phase needs more (e.g. account-level summary fields for the Household layer), add them here first.

---

## 2. AA consent object schema (Setu-style)

```ts
interface ConsentRequest {
  purposeCode: "102" | "104";
  fetchType: "PERIODIC" | "ONETIME";
  frequency: "DAILY" | string;        // DAILY is the only value this build should send
  consentMode: "VIEW" | "STORE";      // VIEW preferred; STORE only if sandbox requires it
  dataLife: string;                   // ISO 8601 duration, always finite — never "INF"
  dataRange: { from: string; to: string }; // ISO 8601 dates, bounded
}
```
**Consumed by**: `backend/src/aa/consent.ts` (builds it), `frontend` Consent intro screen (renders it before redirect, per `AppFlow.md` §1.4), `consent_audit` table (logs it on every lifecycle action).

---

## 3. Internal engine shapes

### 3.1 `InflowStream`
```ts
interface InflowStream {
  streamId: string;
  narrationFingerprint: string;
  mode: "CASH" | "ATM" | "CARD" | "UPI" | "FT" | "OTHERS";
  amountBand: [number, number];
  arrivalDayCDF: { day: number; cumulativeProbability: number }[];
  cyclesObserved: number;
  lowConfidence: boolean;             // true if cyclesObserved < 6
}
```
**Consumed by**: `funding.ts`, `optimiser.ts`, `simulate.ts`. **Not exposed directly** via any API endpoint — it's an internal engine artifact; the dashboard and voice worker only ever see the derived `InsightObject` (§3.3).

### 3.2 `Obligation`
```ts
interface Obligation {
  obligationId: string;
  type: "loan_emi" | "sip" | "utility_mandate" | "other";
  scheduledDay: number;
  amount: number;
  reschedulable: boolean;
  originalPFunded: number;            // 0–1
  optimisedDay: number | null;        // null if not moved
  optimisedPFunded: number | null;
  aboveCoverageGate: boolean;
  explanation: string;                // deterministic, from explain.ts
}
```
**Consumed by**: `InsightObject.obligations` (§3.3), `/dashboard/obligation/:id` (`AppFlow.md` §1.6), `warranty.ts` (enrollment eligibility check).

### 3.3 `InsightObject` — the boundary shape
```ts
interface InsightObject {
  userId: string;
  generatedAt: string;                // ISO timestamp
  obligations: Obligation[];
  degradedMode: boolean;               // true if balance interpolation fallback was used
  summaryText: string;                 // short, pre-generated plain-language summary
}
```
**This is the only shape the voice worker is ever allowed to read** (`GET /insight/:userId`, per `TechSpecifications.md` §3.4 and `Rules.md` §1 rule 6). No field here contains a raw transaction record. If a future change adds a field to this shape, it must be checked against that constraint before merging — anything derived from a single raw transaction rather than an aggregate/computed value does not belong here.

**Consumed by**: `frontend` `/dashboard`, `voice-worker` `getInsightSummary` tool, `insight_objects` table.

### 3.4 `ConsentRecord` (audit ledger row)
```ts
interface ConsentRecord {
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
**Consumed by**: `backend/src/audit/log.ts` (writes), `/consent/manage` audit trail view (`AppFlow.md` §1.8, reads), `consent_audit` table.

### 3.5 `SchemeRecord` (static dataset)
```ts
interface SchemeRecord {
  schemeId: string;
  name: string;
  sourceUrl: string;                  // must be a real gov.in or official source — verified per record
  eligibility: {
    incomeBandMax?: number;
    occupation?: string[];
    familySizeMin?: number;
  };
  nextSteps: string;
}
```
**Consumed by**: `backend/src/schemes/schemes.json` (source), `POST /schemes/match` (queries it), voice worker's `matchSchemes` tool.

### 3.6 `UserProfile` (non-financial, self-reported)
```ts
interface UserProfile {
  householdSize: number;
  dependents: number;
  occupation: string;
  hasExistingDebt: boolean;
  hasExistingInsurance: boolean;
}
```
**Consumed by**: `users.profile` JSONB column, `POST /schemes/match` request body (mapped to `incomeBand`/`occupation`/`familySize`), Profile setup screen (`AppFlow.md` §1.3). **Note**: no income figure is self-reported directly in this build — if `incomeBand` is needed for scheme matching and isn't otherwise captured, add an explicit self-reported income-band field here rather than inferring it from AA data for this purpose (inferred income should stay in the engine layer, not cross into the profile shape used for a different purpose — keep purpose limitation clean per `PRD.md` §8.1).

### 3.7 `WarrantyEnrollment`
```ts
interface WarrantyEnrollment {
  enrollmentId: string;
  userId: string;
  obligationId: string;
  enrolledAt: string;
  status: "active" | "cancelled";
}
```
**Consumed by**: `warranty.ts`, `warranty_enrollments` table, `/warranty` screen.

---

## 4. Database DDL (canonical — consolidates `ImplementationPlan.md` §3.6 and `TechSpecifications.md` §4)

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone TEXT UNIQUE NOT NULL,
  pin_hash TEXT NOT NULL,
  profile JSONB NOT NULL DEFAULT '{}',   -- UserProfile shape, §3.6
  created_at TIMESTAMPTZ DEFAULT now()
);
ALTER TABLE users ADD CONSTRAINT phone_format CHECK (phone ~ '^\+91[0-9]{10}$');

CREATE TABLE consent_audit (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
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
ALTER TABLE consent_audit ADD CONSTRAINT valid_action
  CHECK (action IN ('grant','fetch','pause','revoke','deletion_attested'));
ALTER TABLE consent_audit ADD CONSTRAINT valid_consent_mode
  CHECK (consent_mode IN ('VIEW','STORE'));
CREATE INDEX idx_consent_audit_user ON consent_audit(user_id, created_at DESC);

CREATE TABLE insight_objects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  payload JSONB NOT NULL,               -- InsightObject shape, §3.3
  generated_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_insight_objects_user ON insight_objects(user_id, generated_at DESC);

CREATE TABLE warranty_enrollments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  obligation_id TEXT NOT NULL,
  enrolled_at TIMESTAMPTZ DEFAULT now(),
  status TEXT NOT NULL DEFAULT 'active'
);
CREATE INDEX idx_warranty_user ON warranty_enrollments(user_id, status);
```

**Explicitly not a table in this build**: raw transactions. If `consentMode = STORE` is ever exercised because the sandbox requires it, store the minimum needed, tag it with the granting consent's `dataLife`, and add a purge job — do not create a permanent `transactions` table by default.

---

## 5. Degraded-mode and low-confidence propagation

Two flags originate in the engine and must be threaded through every shape and screen that touches their data, not recomputed independently downstream:

- `degradedMode` (on `InsightObject`): set once in `balance.ts` when the `currentBalance`-per-transaction fallback path is used. Propagates unchanged into the dashboard and any voice `summaryText`.
- `lowConfidence` (on `InflowStream`): set once in `cdf.ts`. Any `Obligation` depending on a low-confidence stream should carry this through in its `explanation` text, even though `Obligation` itself has no dedicated `lowConfidence` field — the explanation string is the mechanism (see `explain.ts` spec in `TechSpecifications.md` §2.7).

---

## 6. API request/response schemas (index — full detail in `TechSpecifications.md` §3)

| Endpoint | Request shape | Response shape |
|---|---|---|
| `POST /auth/otp/request` | `{ phone }` | `{ otpSent, expiresInSeconds }` |
| `POST /auth/otp/verify` | `{ phone, otp, pin }` | `{ sessionToken, isNewUser }` |
| `POST /consent/grant` | `ConsentRequest` (§2) | `{ consentId, redirectUrl }` |
| `GET /insight/:userId` | — | `InsightObject` (§3.3) |
| `POST /schemes/match` | `{ incomeBand, occupation, familySize }` | `{ matches: SchemeRecord[] }` |
| `POST /warranty/enroll` | `{ obligationId }` | `WarrantyEnrollment` (§3.7) |

This table is an index only — `TechSpecifications.md` §3 has full field-level request/response examples. If the two ever disagree on a field name or type, this file's interface definitions in §1–§3 are canonical; fix `TechSpecifications.md` to match.

---

## 7. Change policy

Any change to a shape in §1–§3 requires, in the same commit/PR:
1. Update to this file.
2. Update to every "Consumed by" service listed for that shape.
3. Update to any DDL in §4 that stores the shape.
4. A note in `Tracker.md`'s daily log if the change happened mid-build, so it's traceable later.

This is the practical form of `Rules.md` §2's "data shapes are shared contracts" rule — this file is where that contract is checked.
