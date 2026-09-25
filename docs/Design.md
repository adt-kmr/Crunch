# Design.md — Crunch

Companion to `AppFlow.md` (screens/states) and `PRD.md` (personas/constraints). This file is the visual and interaction design system — palette, type, components, and screen-by-screen notes — written for whoever builds the frontend, so the UI is consistent without every screen needing a separate design decision made from scratch.

---

## 1. Design principles

These four rules should resolve most design disagreements without escalation:

1. **Legible over polished.** The persona (PRD §3) has moderate-to-low digital literacy. A judge will forgive plain design; they won't forgive a dashboard that requires explanation to read. Every screen should be understandable at a glance, in low light, on a mid-range phone.
2. **Numbers earn their place.** This product's credibility rests on specific, honest numbers (accuracy rate, loss ratio, P(funded)). Don't bury them in decorative UI — surface them plainly, and never smooth over a low-confidence or degraded-mode state with confident-looking design (this is a `Rules.md` §1 rule 7/9 constraint, not just a style preference).
3. **Calm, not urgent.** This is financial-wellness software for someone already under timing stress. Avoid red-alert styling, countdown urgency, or anything that reads as pressure — even the bounce-risk warnings should read as informative, not alarming.
4. **One layout, reused.** Per the lean-build reality of an 8-day timeline: pick one card pattern, one modal pattern, one button hierarchy, and reuse them everywhere rather than inventing new patterns per screen.

---

## 2. Visual identity

### 2.1 Color palette
Keep this small — a primary, a neutral scale, and two semantic colors (not a full "success/warning/error/info" set, which tends toward alarm-styling).

| Token | Use | Suggested value |
|---|---|---|
| `--color-primary` | primary actions, active states, links | deep teal or indigo — calm, trustworthy, not a "fintech green" cliché |
| `--color-primary-dark` | hover/pressed states | darker shade of primary |
| `--color-bg` | page background | near-white (`#FAFAF8`) in light mode |
| `--color-surface` | card backgrounds | white, subtle shadow, not heavy borders |
| `--color-text` | primary text | near-black, not pure black (`#1A1A1A`) |
| `--color-text-muted` | secondary text, labels | mid-gray |
| `--color-positive` | funded/on-track states | muted green, not neon |
| `--color-caution` | low-confidence/degraded-mode flags | muted amber, informative not alarming |
| `--color-border` | dividers, input borders | light gray |

Dark mode: invert the neutral scale, keep primary/positive/caution recognizable but desaturated slightly so they don't glow against a dark background. Define both via CSS variables so the theme switches cleanly (see `publishing_artifacts` conventions if this ever becomes a published page — otherwise standard `prefers-color-scheme` handling in the app itself is enough).

### 2.2 Typography
- One typeface family, system-first for performance and familiarity: `-apple-system, "Segoe UI", Roboto, "Noto Sans", sans-serif` — include Noto Sans explicitly, since it has strong support for Devanagari and other Indian scripts if any UI copy appears in a regional language.
- Scale: 3–4 sizes total. Body ~16px minimum (never smaller — this persona should not need to zoom). Headings one or two steps up. Avoid a large decorative display size; this isn't a marketing site.
- Line height generous (1.5+) for body text — legibility over density.

### 2.3 Iconography
- Use a single consistent icon set (e.g. Lucide, already available per the React library list) — don't mix icon styles.
- Icons should always pair with a text label in this product; never icon-only for a primary action, given the literacy constraint. Icon-only is acceptable only for universally understood actions (back arrow, close X).

### 2.4 Spacing and layout
- 8px base spacing unit.
- Single-column layout on mobile (primary target — this persona is more likely on a phone than a desktop). Cards stack vertically; avoid horizontal scroll except where explicitly noted (e.g. a calendar strip, which should scroll horizontally with visible affordance).
- Generous touch targets (minimum 44px height) throughout — this is both an accessibility and a "used on a budget phone, possibly with a cracked screen" consideration.

---

## 3. Component specs

### 3.1 Card (the base unit — used for insight cards, warranty status, scheme matches)
- White/surface background, subtle shadow (not a hard border), rounded corners (8–12px).
- Structure: icon/label row → primary number or statement (large, bold) → supporting explanation text (smaller, muted) → optional action button.
- Never more than one primary action per card.

### 3.2 Before/after obligation strip
- Horizontal, scrollable row of small date-chips, each showing original day (crossed through or dimmed) → arrow → optimised day (bold), color-coded by `aboveCoverageGate` status (positive) vs not-yet-covered (neutral, not caution — "not yet eligible" isn't a problem state).
- Tapping a chip opens Obligation detail (`AppFlow.md` §1.6).

### 3.3 Confidence/loss-ratio dial (warranty screen)
- A horizontal slider, not a dial graphic — simpler to build and to read precisely. Label both ends plainly ("fewer obligations covered, lower cost" ↔ "more covered, higher cost"). Live-update the displayed loss ratio and the count of currently-covered obligations as the slider moves. This is a demo-critical interactive element — test it under actual projector/screen-share conditions before the run-through, not just on a dev laptop.

### 3.4 Consent parameter display
- A simple table or labeled key-value list (not a paragraph) showing purpose code, fetchType/frequency, consentMode, dataLife, dataRange — this needs to be readable at a glance by a judge from across a room, so use larger text here than elsewhere on that screen.

### 3.5 Degraded-mode / low-confidence badge
- Small, muted-amber pill badge with a short label ("Still learning your pattern" / "Estimated — limited data"), placed inline next to the affected number, never as a full-screen warning or modal interruption. It should read as an honest footnote, not an error state.

### 3.6 Refusal statement banner
- Persistent but quiet — a thin banner or footer-style strip, not a modal that blocks the first view. One sentence, plain language: "We only look at your money and your schedule — never your health or your personal life." Dismissible after first view, recallable from a settings/info icon.

### 3.7 Call UI
- Reuse LiveKit's `agent-starter-react` call components largely as-is (per `PRD.md` §7.1) — don't redesign the call surface from scratch given the timeline. Customize only: color tokens to match §2.1, the transcript panel's typography (match §2.2), and add the call-auth PIN entry step as a pre-call screen (per `AppFlow.md` §2.2) styled consistently with the OTP entry screen elsewhere in the app.

### 3.8 Buttons
- One primary style (filled, primary color) for the single main action per screen.
- One secondary style (outlined or text-only) for everything else.
- No more than one primary button visible at a time on any screen.

---

## 4. Screen-by-screen design notes (mapped to `AppFlow.md` §1)

| Screen | Key design note |
|---|---|
| Login / OTP | Minimal fields, large numeric keypad-friendly OTP input, clear retry/resend affordance |
| Profile setup | One field per view or a short single-page form — state inline why each field is asked (ties to purpose-limitation requirement, and it's also just good trust-building) |
| Consent intro | This is a trust-critical screen — give the four consent parameters (§3.4) real visual weight, not a small-print afterthought |
| Consent confirmed | Mirror the same parameter display as confirmation — the repetition is intentional (the demo's trust beat) |
| Fetching | Simple progress indicator with honest copy ("reviewing your last few months") — avoid fake-progress animations that imply more precision than the engine actually reports |
| Dashboard | Refusal banner (§3.6) at top, obligation strip (§3.2) as the primary content, warranty and call CTAs below — don't let secondary CTAs compete visually with the core insight |
| Obligation detail | One card, the explanation text given real space — this is where explainability has to actually read as an explanation, not a data dump |
| Warranty | Dial (§3.3) is the centerpiece; arithmetic (revenue/claim cost) shown plainly beside it, not hidden in a tooltip |
| Consent management | Clear, separated action buttons (pause/revoke/delete) each with distinct visual weight matching their severity — revoke and delete should look more deliberate to trigger (e.g. require a confirm step) than pause |
| Household (preview) | Visually distinct "preview" treatment (e.g. a subtle watermark or badge) so it reads honestly as non-functional, per `AppFlow.md` §1.9 |
| Fundability (static) | Same preview treatment |
| Call launcher | Large, simple "Talk to Crunch" affordance, available from the dashboard persistently, not buried in a menu |
| Error/fallback | Calm, non-technical copy, one clear next action, consistent with principle 3 (calm, not urgent) |

---

## 5. Accessibility and inclusivity

- Minimum 16px body text, 44px touch targets (already noted in §2.4/§2.2 — repeated here because it's a requirement, not a nice-to-have for this persona).
- Color is never the only signal — every status (funded/not-funded, covered/not-covered, degraded-mode) pairs a color with a text label or icon.
- Design for low-bandwidth conditions: no large hero images, no autoplaying video, minimal JS-heavy animation. This matters both for the actual target persona and for demo-day reliability on possibly-imperfect venue wifi.
- If any UI copy is offered in a regional language (even just a few key screens, given the accessibility framing in the Study Brief), ensure the chosen typeface (§2.2) actually renders that script correctly — test this explicitly, don't assume system font fallback handles it.
- Voice channel exists precisely because dashboard literacy shouldn't be a hard requirement — don't let the app design silently assume everyone will use the dashboard; keep the "Talk to Crunch" path visible and equally weighted, not a secondary afterthought link.

---

## 6. Copy/tone guidelines

- Plain language over financial jargon. "Your payment was moved to land after your pay" over "obligation rescheduled to align with inflow probability distribution."
- Never use urgency/scarcity language ("Act now!", "Don't miss out!") — this is a wellness product, the tone should feel like a competent, calm helper, not a sales funnel.
- State uncertainty honestly in copy, matching the engine's own honesty (`Rules.md` §1 rule 7) — "we're still learning your pattern" rather than presenting a low-confidence estimate as fact.
- The refusal statement, consent parameters, and regulatory boundary text (SEBI/IRDAI, `PRD.md` §8.3/§8.4) should be written once, reviewed, and reused verbatim across screens and the voice agent's persona — don't let each surface reword these independently.

---

## 7. What to avoid

- No dense data tables as the primary dashboard view — this isn't a trading terminal.
- No red/alarm styling for bounce-risk information — muted amber (caution token) is the ceiling of visual intensity anywhere in this product.
- No stock fintech clichés (neon green, generic "growth chart" hero illustrations) — they read as generic and don't match the calm, trustworthy tone this persona needs.
- No separate design pattern per screen — reuse the card/button/badge system in §3 everywhere; a new pattern is a cost, not a feature, on an 8-day build.
- No animated "AI thinking" flourishes on the voice call UI that imply more computation is happening live than actually is (the insight was already pre-computed — the call is explaining it, not deriving it in real time; the UI shouldn't visually contradict that).
