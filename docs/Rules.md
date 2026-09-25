# Rules.md — Crunch

This file governs every change made to this repository. It sits alongside `PRD.md`, `ImplementationPlan.md`, and `AppFlow.md` — those describe *what* to build; this describes *what must always be true* while building it. If a task ever seems to require breaking a rule here, stop and flag it rather than proceeding — see §7.

---

## 1. Hard constraints — never do these, regardless of what a task description asks for

1. **No personalized investment advice.** Never generate, in code, copy, or the voice agent's runtime output, a recommendation naming a specific security, mutual fund, or specific allocation. Descriptive analytics and generic education are fine (PRD §8.3); a recommendation is not. This applies to the voice agent's free-generated responses as much as to written UI copy — a prompt instruction alone is not sufficient enforcement; the response-filter described in ImplementationPlan Phase 4 must exist and must be tested.
2. **No blockchain/distributed-ledger claims.** The consent audit trail is a Postgres table (`ImplementationPlan.md` §3.6). Do not introduce Hyperledger, any DLT, or "immutable ledger" language anywhere — code, comments, or copy.
3. **No biometric authentication.** No voice biometrics, no fingerprint, no face verification, anywhere in the auth flow — app or call. Auth is OTP + PIN only (§1.10, PRD FR-10–FR-13).
4. **No claimed Aadhaar eKYC integration.** If Aadhaar is mentioned anywhere (docs, UI copy, pitch materials), it must be framed as a future path requiring regulated-entity status, never as something this build does.
5. **No quantum computing.** The "what if" simulation is a Monte Carlo/bootstrap resample over the empirical CDF (ImplementationPlan Phase 1, `simulate.ts`). Nothing quantum-flavored belongs in code, naming, or copy.
6. **No raw transaction data reachable from the voice worker.** The voice worker's service token must only ever call `/insight/:userId` and `/schemes/match` (ImplementationPlan §4). This is enforced at the API-gateway/middleware layer, not by convention — a code change that adds any other scope to that token is a rules violation, not a judgment call.
7. **No fabricated confidence.** If an inflow stream has fewer than ~6 observed cycles, it is `lowConfidence: true` and must be presented as such everywhere it's shown or spoken (dashboard, call). Do not round low-confidence outputs into confident-sounding language to make a demo look cleaner.
8. **No insurance-style framing of the warranty.** Copy referring to the ₹49/month product must describe a service-level remedy (crediting fees, reimbursing a levied charge), never risk pooling or indemnity language (PRD §8.4). Run new warranty-facing copy against this check before merging.
9. **No silent degraded-mode.** If `balance.ts` falls back to interpolation because `currentBalance` isn't populated per-transaction, the resulting `degradedMode: true` flag must surface in the UI. Do not swallow it to make output look more complete than the underlying data supports.
10. **No scope creep into out-of-scope PRD items** without updating `PRD.md` first. Telephony/PSTN, RAG/embeddings/vector search, a working household-layer backend, production security certification, rate limiting — none of these get built opportunistically "since we're in the file anyway." If one turns out to be genuinely needed, that's a PRD change, not a silent addition.

---

## 2. Architectural invariants

- **Trust boundary**: only the API server (`backend/`) holds `AA_SANDBOX_CLIENT_ID`/`SECRET`, `LIVEKIT_API_KEY`/`SECRET`, and `JWT_SIGNING_SECRET`. The voice worker holds `GOOGLE_API_KEY` and its own scoped `VOICE_WORKER_SERVICE_TOKEN`, and nothing else. The browser holds no secrets. Any change that widens this must be treated as a security-relevant change, called out explicitly in the PR/commit description, and checked against §1 rule 6.
- **Data shapes are shared contracts.** `InflowStream`, `Obligation`, `InsightObject`, `ConsentRecord`, `SchemeRecord` (`ImplementationPlan.md` §3) are read by more than one service. Changing a shape means updating every consumer in the same change — do not patch one side and leave the others stale.
- **Explanations are deterministic.** `explain.ts` and the voice agent's `summaryText` rendering must derive from the same computed values used in the actual decision (optimiser output, coverage gate, CDF). No separate free-text generation step that could state a number the engine didn't actually produce.
- **FR traceability.** Each engine/auth/consent function should carry a comment referencing the `PRD.md` §6 functional requirement it implements (e.g. `// [FR-6]`). This isn't decoration — it's what lets you and a judge trace a claim back to code during Q&A.

---

## 3. Coding conventions

- TypeScript across backend, voice worker, and frontend (per `ImplementationPlan.md` §1/§7.1 stack choice). Python is acceptable for the engine module only if it's genuinely faster to iterate there — but if used, it must run as a clearly separated service/process, not inline-shelled from Node.
- No ML models in the engine for v1 — clustering, CDF construction, the optimiser, and the simulation are all meant to be explainable-by-construction (PRD §9 NFR). If a future phase genuinely needs ML, that's a PRD revision, not an incidental addition.
- Keep the static datasets (`schemes.json`) genuinely sourced — every `SchemeRecord.sourceUrl` must point to a real gov.in or equivalent official source before it ships. Do not fill placeholder schemes "to make the list look fuller."
- Prefer failing loudly over failing silently. Per `AppFlow.md` §1.10 and §4 (edge cases), every unrecoverable error routes to a legible state — no unhandled promise rejections left to surface as a blank screen or a dropped call.

---

## 4. Regulatory/compliance change control

Any change touching the following requires the specific written boundary text in `PRD.md` §8 to be re-checked, not just the code:
- Warranty enrollment or claim-payout copy (§8.4, IRDAI framing)
- Any text the voice agent or dashboard generates about "what to do with savings," "where to invest," or similar (§8.3, SEBI boundary)
- Consent parameter defaults (purpose code, fetchType, frequency, consentMode, dataLife, dataRange) (§8.2)
- The refusal statement's wording or placement (§8.5)

If you're unsure whether a change crosses one of these lines, treat it as if it does and flag it rather than deciding unilaterally — this is exactly the category of thing to bring back for a defense pass rather than resolve in code alone.

---

## 5. Testing and verification rules

- Don't build against sandbox assumptions that haven't been verified. Before writing `balance.ts` logic, confirm from Phase 0's real fixture whether `currentBalance` is actually populated per-transaction — code the fallback path either way, but know which path is actually active.
- The backtest harness (`ImplementationPlan.md` Phase 1) must produce and print an honest accuracy number, including a failure rate, before the optimiser is considered "done." A pipeline that only ever reports success on held-out data hasn't been tested — it's been demoed to itself.
- Any claim made in `docs/` (accuracy numbers, loss-ratio arithmetic, "N schemes covered") must be regenerated from actual code output before the deck is finalized — don't let a number in a slide go stale relative to the code that's supposed to produce it.

---

## 6. Documentation sync rule

`PRD.md`, `ImplementationPlan.md`, `AppFlow.md`, and this file are meant to stay accurate to the running code, not to the plan as originally imagined. If implementation reveals that a planned approach doesn't work (e.g. the sandbox genuinely can't supply per-transaction balance, or the 20-minute run-through doesn't fit with the call channel included), update the relevant doc in the same change that adjusts the code — don't let the docs describe a system that no longer exists. This matters specifically because these docs are also the source for the regulatory one-pagers and the architecture diagram; drift there is drift in what you can defend live.

---

## 7. When a task conflicts with a rule here

1. Stop before writing code that would violate §1 or widen the trust boundary in §2.
2. State plainly which rule the task conflicts with and why the task seems to require it.
3. Propose the narrowest version of the task that doesn't violate the rule, if one exists.
4. If no compliant version exists, say so and wait for an explicit decision rather than choosing a workaround unilaterally — this is the same posture the project already takes toward IRDAI/SEBI framing: state the boundary before it's raised, don't quietly route around it.
