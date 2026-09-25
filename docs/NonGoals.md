# NonGoals.md — Crunch

Every other doc in this set (`PRD.md`, `Rules.md`, `ImplementationPlan.md`) mentions scope exclusions in passing. This file exists so there's one place that says, plainly, what Crunch does **not** do for this build, why not, and what to say if a judge asks. Treat an item appearing here as settled — reopening it mid-build requires updating `PRD.md` first (per `Rules.md` §1 rule 10), not just writing the code anyway because it seemed easy.

---

## 1. How to use this document

- If a task looks like it needs something listed below, stop and check here before building it.
- Each entry has a **reason** (why not) and an **if asked** line (the honest, prepared answer for Q&A) — use the "if asked" line verbatim or close to it; it's already been checked against the regulatory/feasibility reasoning, so improvising a different answer live risks contradicting it.
- Some entries have a **revisit when** condition — that's the only legitimate trigger for un-deferring something mid-build.

---

## 2. Regulatory-forced non-goals

### 2.1 Personalized investment advice
- **What**: recommending specific securities, mutual funds, or specific allocations.
- **Reason**: requires SEBI Registered Investment Adviser registration under the SEBI (Investment Advisers) Regulations, 2013. Unregistered advice is a real exposure, not a hackathon technicality.
- **If asked**: "We deliberately stay in descriptive analytics and education — we show the gap between what a family pays in EMI interest and what they earn in savings, but we never name an instrument or an allocation. Anything that looks like a recommendation routes to a licensed distributor."
- **Revisit when**: never, for this build. A future product phase could partner with a registered adviser, but that's a partnership decision, not a feature to build here.

### 2.2 Insurance-style warranty framing
- **What**: presenting the ₹49/month product as risk pooling or an indemnity contract.
- **Reason**: would trigger IRDAI regulation.
- **If asked**: "The warranty is a service-level remedy — on a covered miss, we credit our own fees and reimburse the levied charge. There's no risk pooling and no indemnity contract. Partnering with an insurer is the clean path if we later want true indemnity."
- **Revisit when**: only with an actual insurance partner in a later phase, and only with that framing written and reviewed first.

### 2.3 Aadhaar eKYC as a live integration
- **What**: claiming the app performs Aadhaar-based identity verification.
- **Reason**: use of Aadhaar for authentication is restricted to specific regulated entity categories under UIDAI rules — a student prototype shouldn't claim this is live.
- **If asked**: "That's a future integration path that would require RE/AA-participant status — for this build, we authenticate with mobile OTP and a PIN, which is what's actually appropriate at this stage."
- **Revisit when**: if Crunch becomes an actual regulated entity — not a build-timeline question.

### 2.4 Voice or fingerprint biometric authentication
- **What**: any biometric verification step, on the app or the call.
- **Reason**: cost and infeasibility in an 8-day build, and it invites a materially harder privacy conversation (biometric data is treated as sensitive personal data under DPDP) than a PIN-based flow does.
- **If asked**: "We use caller-ID plus a spoken or DTMF PIN on the call, and OTP plus PIN in the app — enough to bind the session to the account without taking on biometric-data obligations we haven't built infrastructure for."
- **Revisit when**: not planned; PIN-based auth is the intended long-term approach for this persona, not just an MVP shortcut.

---

## 3. Technically-forced non-goals

### 3.1 Quantum computing for the "what if" simulation
- **What**: any quantum-computing-based simulation of financial outcomes.
- **Reason**: no accessible quantum hardware for this in India today; the cost and infeasibility would be indefensible in an 8-day build, and claiming it invites exactly the skepticism the rubric's technical-feasibility criterion is designed to catch.
- **If asked**: "We looked at that and decided against it — the same explainability we want comes from a much simpler Monte Carlo resample over the empirical arrival-day distribution we're already building for the optimiser. It's cheaper, faster, and just as honest."
- **Revisit when**: never, realistically — the Monte Carlo approach isn't a placeholder for quantum, it's the actual right tool here.

### 3.2 Blockchain / distributed-ledger consent tracking
- **What**: Hyperledger Fabric or any DLT-based consent audit trail.
- **Reason**: large, hard-to-defend architectural claim for the timeline; a simple audit-logged relational table satisfies everything a judge will actually ask ("where's the consent record, can you show me it being revoked, is it timestamped") without inviting a much harder conversation.
- **If asked**: "Our consent ledger is a straightforward audit-logged table — every grant, pause, revoke, and deletion is timestamped and queryable. We didn't see a reason to add blockchain complexity when the actual requirement is auditability, not decentralization."
- **Revisit when**: only if a future version genuinely needs multi-party trustless verification — not something this product's current design requires.

### 3.3 RAG / embeddings / vector search
- **What**: retrieval-augmented generation over financial data or scheme documents.
- **Reason**: the scheme dataset is small and static; a fixed-tool lookup against a sourced JSON dataset is more auditable and more explainable than RAG for this scale, and RAG over raw financial data specifically would violate the voice-channel data-minimization design (`Rules.md` §1 rule 6).
- **If asked**: "At this scale, a fixed dataset with explicit eligibility fields is more explainable and easier to audit than RAG — we know exactly which record produced which answer."
- **Revisit when**: if the scheme dataset grows into the hundreds and static matching becomes unwieldy — a genuine future scaling question, not a v1 gap.

---

## 4. Timeline-forced non-goals

### 4.1 Telephony infrastructure (PSTN/SIP/Exotel/toll-free/missed-call)
- **What**: real phone-network calling, as opposed to the LiveKit browser/app-originated call used in this build.
- **Reason**: a separate infrastructure layer (per the Caller Module docs' own phased plan) — not needed to demonstrate the voice-channel concept, and building it would consume days better spent on the engine and consent lifecycle.
- **If asked**: "The call channel today runs over LiveKit/WebRTC from the browser or app — real PSTN/SIP telephony is a planned next layer, not something we needed to prove the concept works."
- **Revisit when**: a later product phase, once the core insight/consent flow is validated.

### 4.2 Household Layer 2 as a working feature
- **What**: a second AA consent from a family member, real cross-household arbitrage detection, intra-family loan mechanics.
- **Reason**: explicitly wireframe-only per the Study Brief's own "build one, wireframe one, say one" discipline — building it for real would dilute focus on the Rhythm engine, which is the actual differentiator.
- **If asked**: "This is wireframed intentionally — we didn't want to build a second consent flow and dilute the depth of the engine you're seeing live. The design is real, the backend isn't yet."
- **Revisit when**: after ~3 months of validated Layer 1 usage data, per the product's own stated sequencing logic — not a build-week decision.

### 4.3 Fundability Layer 3 (proof export)
- **What**: actual exportable credit-identity documents for lenders.
- **Reason**: one closing slide is enough to make the point; building export logic wouldn't change what's being evaluated this week, and there's no named buyer yet (see PRD open items).
- **If asked**: "This is the long-term value capture — we've deliberately kept it to one slide because the real unlock here is getting a lender to confirm they'd pay for the signal, which is an outreach problem this week, not an engineering one."
- **Revisit when**: once a lender/buyer conversation is underway.

### 4.4 Production security, rate limiting, abuse prevention, multi-tenant scale
- **What**: everything needed to run this for real users at scale.
- **Reason**: out of scope for a demo-scale prototype; the brief explicitly doesn't require "production-ready commercial application" or "enterprise infrastructure deployment."
- **If asked**: "This is a working prototype at demo scale, not a production deployment — the trust boundaries and data-minimization choices are real and would hold at scale, but rate limiting, abuse prevention, and multi-tenant hardening are next-phase engineering work."
- **Revisit when**: post-competition, if the project continues.

### 4.5 Comprehensive automated test suite
- **What**: full behavioral/integration test coverage across the codebase.
- **Reason**: the backtest harness and the structural voice-worker access check (both required, see `Rules.md` §5) matter far more for defensibility than general test coverage does in an 8-day window — prioritize those two over broad coverage.
- **If asked**: (unlikely to come up directly, but if it does) "We prioritized testing the two things that are actually load-bearing for our claims — the backtest accuracy and the voice channel's data-access boundary — over broad test coverage, given the timeline."
- **Revisit when**: ongoing, as the highest-value tests are added first and coverage grows opportunistically after.

---

## 5. Master exclusion list (quick reference, maps to competition brief's own "Not required" column)

| Excluded | Category |
|---|---|
| Personalized investment advice | Regulatory |
| Insurance-style warranty framing | Regulatory |
| Aadhaar eKYC (live) | Regulatory |
| Voice/fingerprint biometrics | Regulatory |
| Quantum computing simulation | Technical |
| Blockchain/DLT consent tracking | Technical |
| RAG/embeddings/vector search | Technical |
| PSTN/SIP/Exotel/toll-free telephony | Timeline |
| Working Household Layer 2 backend | Timeline |
| Fundability Layer 3 export logic | Timeline |
| Production security/scale/rate limiting | Timeline |
| Full automated test coverage | Timeline |
| Full financial "super-app" | Brief scope (explicit) |
| Production-grade cybersecurity certification | Brief scope (explicit) |
| Formal legal certification of compliance | Brief scope (explicit) |
| Actual fundraising or commercial launch | Brief scope (explicit) |

---

## 6. Relationship to the other docs

- `PRD.md` §2.2/§4.2 is where these exclusions are first stated, in the context of what the product is.
- `Rules.md` §1 is where the regulatory- and technically-forced exclusions become hard constraints on code, not just planning intentions.
- `Tracker.md` §4 checks these at submission time.
- This file is the one to hand someone (a teammate, or yourself under Q&A pressure) who needs the *reasoning*, not just the rule — use it to prepare, not just to comply.
