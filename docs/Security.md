# Security.md — Crunch

Companion to `TechSpecifications.md` §6 (which has the concrete config values — TTLs, hashing, secret handling) and `Rules.md` §1–2 (which has the hard constraints). This file is the threat model and reasoning behind those specs — what's being protected, from whom, and why the controls are shaped the way they are. It's also the doc to defend the 15%-weighted "Trust, privacy & regulatory design" rubric criterion from, alongside the regulatory one-pager.

---

## 1. Data classification

Not all data in this system carries the same risk — treat it accordingly rather than applying one blanket policy everywhere.

| Category | Examples | Sensitivity | Handling |
|---|---|---|---|
| Financial transaction data | raw AA fetch responses | High | Never persisted beyond session unless `consentMode: STORE` was explicitly granted; never reaches the voice worker (`Rules.md` §1 rule 6) |
| Derived insight data | `InsightObject` (Schema.md §3.3) | Medium | Computed, aggregated — no raw transaction lines. Readable by app and voice worker under session/service auth |
| Non-financial profile | household size, occupation, etc. | Medium | Self-reported, minimal fields, each mapped to a stated purpose (PRD §8.1) |
| Auth credentials | phone, PIN hash, OTP | High | PIN never stored in plaintext; OTP never logged; phone is the only stable identifier |
| Consent metadata | `ConsentRecord` | Medium | Audit-only, append-only, never mutated after write |
| Static reference data | `schemes.json` | Public | No special handling needed — this is sourced from public gov.in pages |

**Rule of thumb**: if a piece of data could identify an individual transaction (who paid whom, how much, when, exactly), it's High. If it's an aggregate, a probability, or a decision derived from High data, it's Medium. Only Medium-and-below data is allowed to leave the backend's trust boundary (see §3).

---

## 2. Threat model

### 2.1 Assets to protect
1. Raw AA transaction data (High, §1) — the household's actual financial life.
2. AA and LiveKit credentials (`AA_SANDBOX_CLIENT_SECRET`, `LIVEKIT_API_SECRET`) — compromise here means an attacker could impersonate the app to the AA/room infrastructure.
3. User session integrity — an attacker gaining a session should not be able to do more than the legitimate user could.
4. Consent record integrity — the audit trail is only useful if it can't be quietly altered.

### 2.2 Actors and their realistic capability, at this build's scale
- **A curious judge/user** poking at the demo — realistic, should never be able to see another user's data or bypass step-up auth by fiddling with the UI.
- **A network observer** on shared demo wifi — realistic, mitigated by TLS everywhere (§4).
- **A malicious caller** trying to authenticate as someone else on the voice channel — realistic enough to design against (§5), even though full telephony isn't in scope (`NonGoals.md` §4.1).
- **A sophisticated attacker targeting the AA sandbox credentials directly** — out of scope for this build's threat model in the sense that production-grade infrastructure hardening isn't being built (`NonGoals.md` §4.4), but the credential-isolation pattern in §3 still holds even at demo scale, because it costs nothing extra to do correctly from the start.

### 2.3 Top risks and mitigations

| Risk | Mitigation |
|---|---|
| Voice worker compromised or misused to pull raw transaction data | Structural: voice worker's service token is scoped to only `/insight/:userId` and `/schemes/match` at the API-gateway layer, not by convention (`TechSpecifications.md` §5, `Rules.md` §1 rule 6) |
| A caller impersonates the account holder on a call | Caller-ID match + PIN before any personalized data is spoken; no PIN-only fallback (`AppFlow.md` §2.2) |
| Session hijack via a leaked JWT | Short TTL (30 min idle, `TechSpecifications.md` §6), step-up OTP required again for high-stakes actions regardless of session validity |
| Consent audit log tampered with after the fact | Append-only table, no `UPDATE`/`DELETE` grants on `consent_audit` at the DB-role level — enforce this with actual DB permissions, not just application logic |
| Secrets leaked into frontend bundle | `AA_SANDBOX_CLIENT_SECRET`, `LIVEKIT_API_SECRET`, `JWT_SIGNING_SECRET` never referenced in any frontend or voice-worker code path — backend-only, verified by grep before each demo build (§7 checklist) |
| PIN/OTP appearing in logs | Explicit redaction before any logger call (`TechSpecifications.md` §6) — verify by searching log output during Phase 2/3 testing, not just trusting the code comment |
| Over-broad data collection in the profile form | Every `UserProfile` field maps to a stated purpose, shown inline at collection time (`AppFlow.md` §1.3) |

---

## 3. Trust boundaries (summary — full diagram lives in the architecture deliverable per `ImplementationPlan.md` Phase 5)

```
Browser/Caller  →  Frontend (no secrets)  →  Backend (holds AA + LiveKit + JWT secrets)  →  AA sandbox
                                                     │
                                                     ├─► Postgres (Medium-classified data only, see §1)
                                                     │
                                                     └─► Voice worker (holds GOOGLE_API_KEY + scoped service token only)
```
The backend is the only component that can talk to the AA sandbox. The voice worker cannot reach the AA sandbox at all — not "shouldn't," structurally cannot, because it has no AA credentials and its service token is scoped away from any `/aa/*` route. This is the specific claim worth stating plainly if a judge asks how the voice channel stays safe: it isn't a policy, it's an absence of capability.

---

## 4. Transport and storage encryption

- All API traffic over HTTPS/TLS — no plaintext HTTP anywhere, including local dev if it's reasonably easy to avoid.
- LiveKit call media encrypted via WebRTC's built-in transport security (DTLS-SRTP) — no additional work needed here, just don't disable it.
- Postgres: encryption at rest if the hosting environment supports it by default (most managed Postgres does); not a custom build item for this timeline, but don't actively disable it either.
- PIN hashing: bcrypt or argon2, never plaintext, never reversible.

---

## 5. Voice-channel-specific security notes

- Call-auth (caller-ID + PIN) happens before either tool (`getInsightSummary`, `matchSchemes`) is available to the agent — the agent should have no tool access at all in an unauthenticated call state, not just a prompt instruction to "wait for auth."
- The agent's persona instructions should include an explicit refusal behavior for any question that would require data outside its two tools — see `TechSpecifications.md` §5 for the exact prompt guidance.
- No call recording/transcript persistence beyond the session in this build unless explicitly added with its own consent flow — don't silently start logging call transcripts to the database as a debugging convenience and leave it running.

---

## 6. Secure coding checklist (run against every PR touching auth, consent, or the voice worker)

- [ ] No secret is referenced in any file under `frontend/` or `voice-worker/` except the specific, scoped tokens listed in `TechSpecifications.md` §6.
- [ ] Every new backend route has explicit auth middleware — no route is auth-optional by omission.
- [ ] Every high-stakes action route (§`Rules.md` list in §1) requires step-up OTP, not just a valid session.
- [ ] No raw transaction data is returned from any endpoint the frontend or voice worker calls — only `InsightObject` or `SchemeRecord` shapes.
- [ ] No `console.log`/logger call includes `pin`, `otp`, or raw transaction fields — grep for these before merging.
- [ ] Any new DB table or column is checked against §1's data classification — High-classified data doesn't get a new persistent home without a deliberate decision.
- [ ] `consent_audit` remains append-only — no code path issues an `UPDATE` or `DELETE` against it.

---

## 7. Pre-demo security checklist (run once, day of/before)

- [ ] `git grep` for `AA_SANDBOX_CLIENT_SECRET`, `LIVEKIT_API_SECRET`, `JWT_SIGNING_SECRET`, `GOOGLE_API_KEY` across `frontend/` and `voice-worker/` — must return nothing.
- [ ] Attempt to call an `/aa/*` route using the `VOICE_WORKER_SERVICE_TOKEN` — must fail with 401/403 (this is the structural claim you'll make live; confirm it, don't assume it).
- [ ] Confirm session JWTs expire as configured (log in, wait past TTL, confirm re-auth is required).
- [ ] Confirm a step-up-gated action (e.g. warranty enrollment) actually blocks without the step-up OTP, even with a valid session.
- [ ] Confirm the consent audit trail shows every action from a full grant→pause→revoke→deletion cycle, timestamped, in order.

---

## 8. What this build does not claim (see `NonGoals.md` §4.4 for full reasoning)

This is a demo-scale prototype's security posture, not a production one. It does not include: penetration testing, formal security certification, rate limiting/DDoS protection, intrusion detection, or multi-tenant isolation hardening. The trust-boundary and data-minimization design in §3 would hold at scale, but the operational hardening around it is explicitly next-phase work — say this plainly if asked, rather than implying more maturity than exists.
