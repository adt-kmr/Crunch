# AppFlow.md — Crunch
Companion to `PRD.md` and `ImplementationPlan.md`. This document maps every screen/state a user (or caller) moves through, what triggers each transition, and what happens on failure. Use it to build the frontend routes and the voice agent's conversation states in lockstep with the backend's actual capabilities — nothing here should describe a screen the backend in `ImplementationPlan.md` doesn't support.

---

## 0. Two entry channels, one account

A user reaches Crunch through either the **app** (web dashboard) or the **call** (voice channel). Both authenticate against the same account, but the call channel is intentionally read-only against a narrower slice of data (see PRD §8.3/FR-17–FR-20). Treat this document's app flow (§1) and call flow (§2) as parallel, not sequential — a user may never open the app and only ever call, or vice versa, except that AA consent (§1.4) can currently only be granted through the app.

---

## 1. App flow

### 1.1 Screen inventory

| # | Screen | Route | Purpose | Entry condition |
|---|---|---|---|---|
| 1 | Login | `/login` | phone entry | unauthenticated |
| 2 | OTP verify | `/login/otp` | OTP + PIN check | phone submitted |
| 3 | Profile setup | `/onboarding/profile` | capture non-financial fields | first login only |
| 4 | Consent intro | `/consent/intro` | explain the 4 AA parameters before redirect | post-profile, or returning user with no active consent |
| 5 | Consent redirect (external) | AA/FIU-hosted | user approves in their AA app | user taps "Continue to AA" |
| 6 | Consent confirmed | `/consent/confirmed` | show granted parameters, trigger fetch | AA redirect returns success |
| 7 | Fetching | `/consent/fetching` | loading state while engine runs | fetch triggered |
| 8 | Dashboard | `/dashboard` | insight cards, warranty status, refusal statement | engine run complete |
| 9 | Obligation detail | `/dashboard/obligation/:id` | before/after + explanation for one obligation | tap a card on dashboard |
| 10 | Warranty | `/warranty` | enrollment, loss-ratio dial | tap "Warranty" from dashboard |
| 11 | Consent management | `/consent/manage` | pause / revoke / delete, audit trail | tap "Manage consent" from dashboard |
| 12 | Household (wireframe) | `/household` | static mockup only | tap "Household" (visible after ~3 months simulated, or always-visible-but-labeled-preview for demo) |
| 13 | Fundability (static) | `/fundability` | one static screen | tap "Proof export" from dashboard |
| 14 | Call launcher | `/call` | in-app "call me" / browser call UI | tap "Talk to Crunch" from anywhere |
| 15 | Error/fallback | `/error` | generic failure screen | any unrecoverable error |

### 1.2 Login → OTP → session

```
[Login] --submit phone--> [OTP sent] --enter OTP + PIN--> [Session created] --> [Dashboard or Profile setup]
                                  |
                                  +--wrong OTP (3x)--> [Locked, retry in N min]
```
- **New user** (no profile row): route to Profile setup (§1.3).
- **Returning user, no active consent**: route to Consent intro (§1.4).
- **Returning user, active consent + insight objects exist**: route straight to Dashboard.
- **Returning user, active consent but insight stale (>24h, per DAILY frequency)**: Dashboard loads with a "refreshing" badge while a background fetch runs; show last-known insight in the meantime, don't block on it.

### 1.3 Profile setup (first login only)

Fields: household size, dependents, occupation type, self-reported existing debt/insurance (yes/no + rough category, not amounts — amounts come from AA data, not self-report). Keep this to under 10 fields; each field must map to a stated purpose (scheme matching, mainly) — display that purpose inline next to the field, don't just collect it silently.

→ On submit: proceed to Consent intro.

### 1.4 AA consent flow

```
[Consent intro] --"Continue to AA"--> [AA/FIU external flow] --approve--> [Consent confirmed] --auto--> [Fetching] --> [Dashboard]
        |                                        |
        |                                        +--user declines/cancels--> [Consent intro, with "why we ask" expanded]
        +--user taps "why we ask"--> [inline expansion, no navigation]
```
- Consent intro screen must render, before redirect: purpose code, fetchType/frequency, consentMode, dataLife, dataRange — this is the FR-15 requirement, don't defer it to a post-consent summary only.
- Consent confirmed screen re-displays the same four parameters as granted (not just "success") — this is the pair-shot for the demo's trust beat.
- Fetching screen: show a short, honest loading message (engine run may take a few seconds on real data) — do not fake instant results.

### 1.5 Dashboard

Primary elements, top to bottom:
1. Refusal statement banner (FR-24) — persistent, not dismissible on first view, dismissible-but-recallable after.
2. Obligation summary: "N obligations reviewed, M moved to reduce bounce risk" with a before/after calendar strip.
3. Warranty status card (enrolled / eligible-not-enrolled / not-yet-eligible).
4. "Talk to Crunch" call CTA.
5. Links to Consent management, Household (preview), Fundability (static).

Each obligation in the calendar strip is tappable → Obligation detail (§1.6).

**Degraded-mode state**: if the engine flagged `degradedMode: true` (balance interpolation fallback, per ImplementationPlan §Phase 1), show a small, honest inline note on the dashboard — do not hide this from the user or the demo audience. This is a credibility asset, not a bug to disguise.

**Low-confidence state**: if an inflow stream is `lowConfidence: true` (<6 cycles observed), obligations depending on it show a "still learning your pattern" tag instead of a confident recommendation.

### 1.6 Obligation detail

Shows: original scheduled day, original P(funded), optimised day (if moved), optimised P(funded), the deterministic explanation string, and — if above the coverage gate — a warranty-eligibility badge with a link to §1.7.

### 1.7 Warranty screen

- If not enrolled and eligible: enrollment CTA, step-up OTP required before confirming (FR-11).
- Loss-ratio dial: a slider over the confidence threshold, live-updating displayed loss ratio and which obligations currently clear the gate — this is built for the demo's "turn the dial" beat, so it must be interactive, not a static number.
- Revenue/claim-cost arithmetic shown plainly (₹49×12 vs ~₹295/claim) so the economics are visible, not just claimed in a slide.

### 1.8 Consent management

Four actions, each behind step-up OTP: pause, revoke, request deletion, (view-only) audit trail of every past action with timestamps. This screen is the one the consent-lifecycle demo beat (grant→fetch→pause→revoke→deletion attestation) is performed on live — build it to be legible on a projector, not just functionally correct.

```
[Consent management] --pause--> [Paused state, fetch suspended] --resume--> [Active]
                     --revoke--> [Revoked state, no further fetch] (terminal for that consent)
                     --request deletion--> [Deletion attested, timestamp logged] (terminal)
```
Each transition writes a `ConsentRecord` row (ImplementationPlan §3.4) and immediately reflects in the audit trail list on the same screen — no page reload needed to see it.

### 1.9 Household (wireframe) and Fundability (static)

Both are non-functional per PRD scope. Build them as real routes with real (static) content so they're navigable during the demo, but do not wire any backend logic — no second AA consent flow, no export logic. Label them subtly as "preview" if a judge might otherwise assume they're live (honesty here is worth more than the illusion of completeness).

### 1.10 Error/fallback

Any unrecoverable error (sandbox timeout, engine failure, auth failure after retries) routes here with: a plain-language explanation, no stack traces, and a single clear next action ("try again" or "contact support"). Per PRD §9 NFR, this should never silently fail — always land the user somewhere legible.

---

## 2. Call flow (voice channel)

### 2.1 States

```
[Idle] --incoming call / "call me" tap--> [Call-auth]
                                              |
                          caller-ID mismatch  |  caller-ID match
                                   +----------+----------+
                                   |                     |
                          [Auth failed, offer            [PIN prompt]
                           app-based verification]              |
                                                    wrong PIN (3x)  correct PIN
                                                        |               |
                                              [Auth failed, escalate]  [Authenticated]
                                                                          |
                                                              +-----------+-----------+
                                                              |                       |
                                                    [Insight explanation]   [Scheme eligibility]
                                                              |                       |
                                                              +-----------+-----------+
                                                                          |
                                                                    [Follow-up loop]
                                                                          |
                                                                    [Call ends]
```

### 2.2 Call-auth
- Caller-ID checked against the phone number on the account (FR-12). If it doesn't match (e.g. calling from a different phone), do not proceed by PIN alone — offer the user a path to verify in-app instead. This keeps the "voice channel is narrowly scoped" claim intact; don't loosen it under a "just let them in with a PIN" convenience pressure during the demo.
- Spoken or DTMF PIN, 3 attempts, then escalate to app-based re-auth.

### 2.3 Insight explanation state
- Agent calls `getInsightSummary(userId)` (ImplementationPlan §3, only tool with access to pre-computed insight objects).
- Delivers the dashboard's `summaryText` in natural, possibly vernacular language — this is a rendering/translation step, not a new computation, and it must not introduce numbers that weren't already in the InsightObject.
- If the user asks a question the agent can't answer from the InsightObject (e.g. "what about next month"), the agent should say so plainly and offer to check the app or call back — it should not improvise a number.

### 2.4 Scheme eligibility state
- Agent calls `matchSchemes(profileFields)` using the profile captured in §1.3 (income band, occupation, family size) — never raw transaction data.
- On a match, agent explains the scheme and the stated `nextSteps` from the SchemeRecord.
- Follow-up questions re-invoke `matchSchemes` with narrower parameters (FR-20) rather than open-ended reasoning — if the dataset genuinely doesn't cover a follow-up, the agent says so rather than guessing.

### 2.5 Call end
- No re-verification needed to end (FR-13). Session token invalidated server-side on disconnect.
- No summary SMS/email in this build (out of scope — telephony/SMS integrations are explicitly deferred per PRD §2.2).

---

## 3. Cross-channel consistency rules

- Any number spoken on a call must trace back to a field already present in an `InsightObject` or `SchemeRecord` — if it isn't in one of those two shapes, the agent cannot say it. This is the actual mechanism behind the "voice channel never touches raw data" claim, so it belongs in this flow doc, not just the architecture doc.
- The refusal statement (FR-24) should have a call-channel equivalent: the agent should be able to state, if asked, that it only discusses money and scheduling, not health or personal life — this should be in the persona's system prompt, not improvised.
- Step-up-OTP-gated actions in the app (consent changes, warranty enrollment) are **not** available on the call in this build — the call channel is read/explain-only. If a caller asks to revoke consent or enroll in the warranty, the agent should direct them to the app rather than attempting the action itself.

---

## 4. Edge cases and error states

| Situation | Behavior |
|---|---|
| AA sandbox fetch times out | Dashboard shows last-known insight (if any) with a "couldn't refresh" note; retry button; no silent failure |
| Engine produces zero reschedulable obligations | Dashboard states this plainly ("nothing needed moving this cycle") rather than showing an empty/broken-looking state |
| User has <6 cycles of data | All affected obligations show "still learning" tag; no warranty eligibility offered on those yet |
| Call caller-ID doesn't match account | No PIN fallback that fully authenticates — escalate to app-based verification |
| Voice agent asked something outside its two tools' scope | States it doesn't have that information rather than guessing; offers app or callback |
| Consent revoked mid-session | Any in-progress fetch is aborted; dashboard reflects revoked state immediately; call channel, if mid-call, informs the caller consent was revoked and ends the personalized part of the call |
| User requests deletion | Attestation logged immediately; actual data deletion follows whatever the sandbox's deletion contract specifies — document this gap explicitly rather than implying instant erasure if the sandbox doesn't guarantee it |

---

## 5. What this flow deliberately does not include

Per PRD §2.2 / §4.2: no PSTN/SIP call routing (this is a LiveKit browser/app-originated call only for the build), no Aadhaar/biometric verification step anywhere in either flow, no working household-layer backend behind the `/household` screen, no investment-recommendation state in the call flow (the agent's tool set structurally cannot produce one — see §2.4's scope limit).
