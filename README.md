# warm-dms
Cold reach out to Hot reach out 
# warm-dms — Product & System Plan v0.1

**Status:** Draft for review. Not implementation-ready until reviewer feedback is integrated.
**Date:** 2026-04-19.

---

## 0. Thesis, in one line

> A conversational agent that turns a goal ("reach ML-systems profs at NeurIPS for a PhD rec", "warm up 20 seed VCs for my raise", "get the design lead at Figma to remember me") into researched, factually verified, personalised outreach — delivered as a written message (DM / letter) plus an optional physical gesture (flowers, coffee, postal card) routed to an office or school address.

Three things make it distinct from Clay / Apollo / Lemlist / Sendoso:

1. **Goal-first, not list-first.** Users describe intent in voice or text. The agent *finds* the people, researches them, and proposes who to target. Competitors require the user to build a list first.
2. **Evidence-bound drafting.** Every factual claim in the DM/letter is traced to a source snippet the LLM saw. Multiple verification passes. No hallucinated "congrats on the Stanford MBA" when they went to Wharton.
3. **Paired physical gesture.** A DM alone is ignored. A DM plus flowers-on-their-desk the same day is remembered. MVP uses deep-linked pre-filled checkouts (user clicks "order" on a Zomato / Ferns&Petals / Lob page we've pre-filled with product + address). Full partner-API integration is v1+.

### Non-goals (v0)
- Bulk cold-email blasting. This is 1:1 high-effort warm outreach.
- Automated LinkedIn mass-messaging — ban risk too high; humans paste.
- Anonymous / spoofed sending. Every touch is sender-identified.
- Home addresses. Office / school / published PO box only.

---

## 1. Problem & user

The target user is *anyone trying to pierce cold-outreach noise* when the relationship matters:

- **Seed founder** raising → 40 VCs
- **PhD applicant** → 15 professors
- **Job hunter** targeting 10 dream companies → 30 hiring managers / skip-levels
- **BD / partnerships** → 20 enterprise contacts
- **Academic** → 25 researchers at a conference

The common shape: *small N, high per-target value, high cost of getting it wrong (sounding generic, wrong facts, creepy, spammy).*

The common pain: manual research (30-60 min per target) is the real bottleneck — LinkedIn stalking, paper skim, X feed scan, drafting, address lookup, gift ordering. Nobody does this well at N=30, so most people send a templated DM and lose.

warm-dms compresses the 30-60 min of manual work to ~2 min of review + click.

### Personas and their variant flows

| Persona | Primary channel | Gesture | Research depth |
|--|--|--|--|
| Founder → VC | LinkedIn DM + email | Coffee delivery to office morning of pitch | VC's portfolio, recent tweets, fund thesis |
| PhD applicant → prof | Email (not DM) | Handwritten-style letter via Lob | Prof's recent papers, lab page, grants |
| Job hunter → hiring | LinkedIn + email | Card + small gift to office | Company posts, role, hiring manager's past moves |
| BD rep | LinkedIn + email | Catered lunch to team address | Company news, funding, exec changes |
| Conference networking | Email + in-person nudge | None (in-person) | Their talk, recent work |

MVP targets **founder→VC** and **PhD applicant→prof** because (a) both already culturally accept unsolicited outreach, (b) both have clearly public addresses (fund office / university dept), (c) legal / harassment risk is lowest.

---

## 2. Core UX — two flows

### 2.1 Primary flow: Goal → outreach (new session)

```
[1] User speaks or types:
    "I'm raising a $3M pre-seed for a dev tools company. Find me 20
     early-stage investors in the US+India who've backed infra in the
     last 18 months. Help me reach out warmly."

[2] Agent parses intent via Gemini 2.5 Pro:
    {
      intent: "raise_fundraising",
      sender_context: "dev tools pre-seed, $3M",
      target_profile: "early-stage VCs, infra thesis, US+India, <18mo activity",
      n_targets: 20,
      tone: "founder-to-investor",
      gesture_budget: unspecified  → ask user later
    }

[3] Agent asks 2-3 clarifying questions ONLY if ambiguous
    (e.g., "any specific funds to exclude?"). Otherwise proceeds.

[4] Research phase (background job, ~60-120s, streamed progress):
    - Crust Data people+company search → candidate list (50-100)
    - Per-candidate: Crust activity pull (posts, tweets, fund news)
    - Cross-reference: recent portfolio adds, tweets on infra, blog posts
    - Score + rank by fit → top 20 + "why" line for each

[5] User reviews target table. UI: candidate card = photo, name, fund,
    why-fit blurb, evidence snippets, office address (if found).
    User selects who to pursue (e.g., 12 of 20), optionally adds own notes.

[6] Draft phase (per selected target, in parallel, ~10s each):
    - Per target, Gemini drafts:
        - DM version (LinkedIn, ~150 words)
        - Letter version (Lob, ~250 words)
        - Email version (~200 words)
    - Each draft has inline <claim>…</claim> tags linked to evidence.
    - Hallucination guard (see §5) scrubs/rewrites unverified claims.

[7] User reviews drafts. For each target, user can:
    - accept as-is
    - edit inline
    - regenerate
    - skip

[8] Physical gesture phase (optional per target):
    - System proposes product based on profile (coffee shop near their
      office, flowers for PhD reco letter thanks, book matching a
      recent paper they wrote, etc.)
    - Generates pre-filled vendor deep-link:
        - Zomato/Swiggy for food (IN)
        - DoorDash/UberEats for food (US)
        - Ferns&Petals for flowers (IN)
        - UrbanStems / BloomNation (US)
        - Lob for a printed card / letter
        - Amazon-gift-card fallback
    - User clicks → opens pre-filled cart → completes payment.

[9] Sending phase:
    - DMs: v0 is "copy to clipboard + open LinkedIn compose" — human
      hits send. No automation risk.
    - Emails: sent via user's authenticated Gmail/Outlook (OAuth).
    - Letters: submitted to Lob API → mailed next-day.
    - Gestures: user completes vendor checkout.

[10] Tracking phase:
    - Lob delivery status → polled and shown.
    - Email opens/replies → Gmail thread watch (user OAuth scope).
    - Reply detected → agent drafts a follow-up with same evidence rules.
```

### 2.2 Secondary flow: Bookmark → monitor → trigger (existing relationships)

After initial outreach, user can "bookmark" a target for ongoing monitoring. A cron-style worker polls Crust Data every N minutes:

```
for target in user.bookmarks:
    new_activity = crust.fetch_activity(target, since=last_check)
    for event in new_activity:
        if trigger_matcher.is_warm_moment(event):
            queue_alert(user, target, event)
            maybe_draft_followup(user, target, event)
```

"Warm moments" are events worth reaching about:
- Target posts new content matching user's thesis
- Target changes jobs / gets promoted
- Target's company raises / ships a product
- Target publishes a paper / gives a talk
- Target wins an award
- Target engages with content similar to user's

**Cadence:** 5-minute polling per bookmarked target for VIPs; 1-hour for standard; 1-day for archival. Driven by per-target tier set by user.

**Rate control:** at most *one* follow-up outreach per target per 14 days, regardless of signals. Hard cap — protects against "I got pinged every time I tweeted" harassment patterns.

---

## 3. System architecture

```
┌────────────────────────────────────────────────────────────────┐
│                        Frontend (Next.js 15)                   │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ Voice   │  │ Target   │  │ Draft    │  │ Order    │        │
│  │ input   │  │ list     │  │ review   │  │ confirm  │        │
│  └─────────┘  └──────────┘  └──────────┘  └──────────┘        │
└──────────────┬────────────────────────────────┬────────────────┘
               │ SSE / WebSocket                │ REST
               ▼                                ▼
┌────────────────────────────────────────────────────────────────┐
│                  Orchestrator (FastAPI + Python)               │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐     │
│  │        Agent loop (Gemini 2.5 Pro, tool-calling)     │     │
│  │  tools: crust_search, web_search, fetch_paper,       │     │
│  │         enrich_target, draft_dm, draft_letter,       │     │
│  │         fact_check, gen_gesture_link                 │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ Session  │ │ Research │ │ Draft    │ │ Delivery │           │
│  │ manager  │ │ service  │ │ service  │ │ service  │           │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │
└──────────┬──────────────┬────────────┬────────────┬────────────┘
           │              │            │            │
           ▼              ▼            ▼            ▼
┌──────────┐  ┌────────────┐  ┌────────────┐  ┌─────────────────┐
│ Postgres │  │ Redis      │  │ Object     │  │ Background      │
│          │  │ (cache +   │  │ store      │  │ worker (Celery  │
│          │  │ queue)     │  │ (drafts,   │  │ or APScheduler) │
│          │  │            │  │ artifacts) │  │ — cron polling  │
└──────────┘  └────────────┘  └────────────┘  └─────────────────┘

External:
  ─ Crust Data API (people, company, activity) — rate-limited, paid
  ─ Gemini 2.5 Pro / Flash (reasoning, drafting, classification)
  ─ Whisper / Deepgram (voice-to-text)
  ─ Lob (printed mail)
  ─ Zomato / Swiggy / DoorDash / Ferns&Petals (deep-links, no API in v0)
  ─ Gmail API (send + watch, user OAuth)
  ─ Supabase / Clerk (auth)
```

### 3.1 Component responsibilities

- **Frontend** is "thin": streams progress, renders cards, captures reviews. No business logic.
- **Agent loop** is the only place LLM reasoning lives. All domain actions (search, enrich, draft, verify) are tools it calls.
- **Session manager** persists long-running agent sessions. A session is a campaign — prompt → targets → drafts → sends. Sessions are resumable.
- **Research service** wraps Crust + web search + paper fetch. Cache-heavy (24h TTL per target identity). Rate-limit aware.
- **Draft service** is pure LLM + evidence. Input: target + intent + user context. Output: draft with `<claim evidence-id="…">…</claim>` tags.
- **Delivery service** generates pre-filled vendor URLs and handles Lob submissions.
- **Background worker** handles (a) cron polling for bookmarks, (b) async research jobs, (c) Gmail watch processing.

### 3.2 Agent loop shape

Gemini 2.5 Pro with function calling. Loop shape:

```
system_prompt (role, paranoia rules, sender context)
user_prompt (parsed intent)
loop:
    gemini.generate_with_tools(history, tools)
    if tool_call: run tool, append result to history
    if final_answer: break
    if iterations > 30: break (safety)
```

Tools return structured JSON; model reads result and decides next step. This is a standard ReAct loop — no LangGraph, no Temporal, hand-rolled in ~300 lines.

**Why not LangGraph/CrewAI:** they hide the control flow. For a system that has to be paranoid about verification, explicit control over every tool call is more important than convenience.

---

## 4. Data model

```sql
-- Users and auth handled by Supabase.

Session(
  id uuid pk,
  user_id uuid,
  intent jsonb,             -- parsed from prompt
  raw_prompt text,
  voice_recording_url text,
  status enum('researching','reviewing','drafting','sending','done','failed'),
  created_at, updated_at
);

Target(
  id uuid pk,
  canonical_name text,      -- normalized
  role text, org text,
  linkedin_url, twitter_url, email,
  office_address jsonb,     -- {line1, city, country, postal, verified_source}
  school_address jsonb,     -- for profs
  last_enriched_at,
  crust_id text             -- stable handle to Crust entity
);

SessionTarget(                -- join: a target inside a campaign
  id uuid pk,
  session_id, target_id,
  reason text,                -- "why this target for this user"
  fit_score float,
  user_status enum('proposed','accepted','skipped'),
  created_at
);

Activity(                     -- raw events we've observed about a target
  id uuid pk,
  target_id,
  source enum('crust','arxiv','twitter','linkedin','web'),
  kind enum('post','paper','jobchange','funding','award','talk'),
  url, content_excerpt text,
  observed_at, fetched_at
);

Evidence(                     -- atomic factual claims with sources
  id uuid pk,
  target_id,
  claim text,
  source_url, source_snippet text,
  verifier enum('rule','llm','human'),
  verified_at, confidence float
);

Draft(
  id uuid pk,
  session_target_id,
  channel enum('linkedin_dm','email','letter','twitter_dm'),
  body_markdown text,
  body_with_claims text,      -- <claim id="xyz">…</claim>
  used_evidence_ids uuid[],
  user_status enum('generated','edited','approved','sent','skipped'),
  sent_at
);

Order(
  id uuid pk,
  session_target_id,
  vendor text,                -- 'zomato','fnp','lob','ubereats'
  product_summary text,
  deep_link_url text,
  delivery_address jsonb,
  status enum('drafted','user_confirmed','completed','failed'),
  created_at, completed_at
);

Bookmark(                     -- post-initial-contact tracking
  id uuid pk,
  user_id, target_id,
  tier enum('vip','standard','archival'),
  last_poll_at,
  created_at
);

Alert(                        -- emitted when a warm moment fires
  id uuid pk,
  user_id, target_id,
  activity_id,
  trigger_reason text,
  status enum('new','seen','drafted','dismissed'),
  created_at
);
```

One deliberate choice: **Target is global, Session-scoped data is separate.** If two users both target "Marc Andreessen," there's one `Target` row, two `SessionTarget` rows. Lets us share expensive enrichment across users while keeping campaigns private.

---

## 5. LLM layer + hallucination guard

This is the highest-leverage component. A single wrong fact in a letter to a professor and the user is burned.

### 5.1 Evidence-required drafting

The drafter prompt explicitly instructs:

> Every factual claim about the target — their role, affiliation, recent work, achievements, personal details — must be grounded in an evidence snippet from the retrieved context. Wrap each claim in `<claim id="…">…</claim>` tags matching an evidence id. **If you cannot ground a claim, do not make it.** Write around it or omit it.

### 5.2 Three verification passes

After the first draft:

**Pass 1 — structural (regex / rule):** every `<claim>` tag must have an `id`; every id must exist in the Evidence list passed in. Ungrounded → reject, regenerate.

**Pass 2 — semantic (cheap LLM, Gemini Flash):** for each (claim, evidence) pair, ask "does evidence entail claim?" with 3-way output (entails / contradicts / insufficient). Anything non-entailing → flag.

**Pass 3 — human:** UI highlights every claim on hover shows the source. User must see the flags before approving. Flagged claims default to "strip," user can override.

### 5.3 Categories that need extra paranoia

- **School / degree claims** — cross-check against public CV / uni page
- **Named papers / work** — must have arXiv or DOI as evidence
- **Direct quotes** — require source URL, never paraphrase-then-quote
- **Personal details** (family, hobbies, location) — disallowed unless from the target's own public profile *and* sender explicitly opts in per-target

### 5.4 Sender-facts also go through the same path

If the sender says "I'm raising $3M for a dev tools company," we don't push that as a claim to the recipient unless the sender marks it as stable. Sender details are in their user profile; we pull verified fields only.

---

## 6. Voice + intent layer

### 6.1 Capture
- Browser: Web Speech API (free, works) for live dictation.
- Fallback / higher quality: record audio → send to Whisper v3 large on our backend (100-200ms accuracy penalty for much better transcription).
- Alternative: Deepgram Nova-3 for realtime streaming if we want sub-second latency.

**MVP: Web Speech API + Whisper fallback on low-confidence segments.** Deepgram is v1.

### 6.2 Parse
Gemini 2.5 Pro with a strict JSON schema:

```json
{
  "intent": "fundraising|academic_outreach|job_search|bd|networking|other",
  "sender_context": { "role": "...", "pitch": "...", "constraints": ["..."] },
  "target_profile": {
    "role_keywords": ["..."],
    "org_constraints": ["..."],
    "geography": ["..."],
    "recency_constraints": "...",
    "exclusions": ["..."]
  },
  "n_targets": 20,
  "channel_prefs": ["linkedin_dm","email","letter"],
  "gesture_budget_usd": 25,
  "gesture_kinds": ["coffee","flowers","book","card"],
  "tone": "formal|warm|founder|academic",
  "deadline_iso": "..."
}
```

If any field is critically missing and can't be inferred, Gemini returns a `needs_clarification` array. Frontend asks up to 3 questions max, then proceeds with labeled defaults.

---

## 7. Channels & sending

| Channel | Send mechanism (v0) | Risk | Notes |
|--|--|--|--|
| LinkedIn DM | Copy-to-clipboard → user pastes | Ban risk 0 | No automation — human sends |
| X DM | Same — copy + paste | Ban risk 0 | Same |
| Email | Gmail API via user OAuth | Low (CAN-SPAM OK if sender-identified) | Auto-send, logged |
| Physical letter | Lob API | Zero — legal | Auto-send |
| SMS | Not in v0 | — | — |

**Explicit stance:** no LinkedIn automation in v0. The industry has beaten this to death — every LinkedIn automation tool either gets accounts banned or runs a cat-and-mouse with LinkedIn's detectors. We *generate* the DM copy; the human sends. This kills the "scale" story but preserves the sender's account. v1 can revisit via LinkedIn's official Sales Navigator API for paid tiers.

---

## 8. Delivery vendors & physical gestures

### 8.1 Per-vendor MVP path

| Vendor | MVP path | v1 path |
|--|--|--|
| Zomato / Swiggy (IN food) | Deep-link to cart with pre-filled items + address string | Partner API if attainable |
| DoorDash / UberEats (US food) | Deep-link to restaurant + address | DoorDash Drive API (paid, feasible) |
| Ferns&Petals (IN flowers) | Deep-link product + address form | Affiliate partnership |
| UrbanStems / BloomNation (US flowers) | Deep-link | Partnership |
| Lob (mail) | Full API — auto-submit | Already done in v0 |
| Amazon gift card | Send via Amazon Incentives API or email code | Same |

**MVP delivery UX:**

1. System proposes product + address. User confirms.
2. Frontend renders an order card: vendor logo, product, address, price.
3. User clicks "Order on Zomato" → new tab opens vendor page with as much pre-filled as deep-linking allows.
4. User finishes payment on the vendor's site.
5. Frontend asks "did it go through?" → user confirms → `Order.status = completed`.

Lossy, but it sidesteps a dozen partner agreements. And crucially, *user pays the vendor directly* — we don't touch payment, skipping a payment-processor nightmare at MVP.

### 8.2 Product recommendation logic

Per-target product recommendation via Gemini Flash given:
- Target role + interests (from research)
- Sender intent
- User's gesture budget
- Address's country

Simple prompt, structured output: `{ vendor, product_name, product_url, price_usd, rationale }`. Always surfaced for user to override.

### 8.3 Lob flow (the underrated MVP path)

Lob's API prints + mails in 1-3 days. For academic outreach in particular, a physical letter to a professor's department is far more memorable than a DM and far safer than unsolicited food. I'd argue **Lob letters are the MVP hero gesture, not food delivery.** Food is flashier but:
- Legal complexity is higher
- Address accuracy is more critical
- Failure modes are worse (food arrives at wrong desk)
- Cost per touch is higher

Lob is: $1.50 / letter, legal everywhere, hand-addressed style templates available, reliable. Food delivery integration can be v1.

---

## 9. Polling & "warm moment" detection

### 9.1 Polling strategy

**Per-user-bookmark, tiered:**
- VIP: every 5 min
- Standard: every 60 min
- Archival: every 24 hours

**Batched at Crust level:** one cron per tier pulls activity since last_poll_at for all targets in that tier in one API call where possible. Cache per-target 24h.

Rough cost model: 100 active users × 20 bookmarks × (avg tier = 60 min cadence) = 100 × 20 × 24 = 48,000 polls/day. Crust pricing needs to be confirmed but this is the sizing exercise. If each poll = 1 credit, we're burning 1.4M credits/month — not nothing. **We need to confirm Crust pricing before finalizing cadence.** [OPEN QUESTION]

### 9.2 Trigger matcher

Classifier (Gemini Flash, cheap) evaluates each new Activity against the bookmark's tags + sender intent:

```
input: target, activity, sender_intent, prior_outreach
output: { is_warm_moment: bool, confidence: float, reason: str, suggested_angle: str }
```

Only moments with confidence ≥ 0.75 surface as alerts. Hard cap: 1 alert per target per 14 days.

---

## 10. Ethics, legal, safety — the real risk surface

Per user decision: no consent, surprise is core, office/school addresses only. This works **if and only if** the following guardrails hold:

### 10.1 Hard rules (product-level, non-negotiable)

1. **Office / school / PO box only.** Home addresses are explicitly disallowed, even if available in Crust. Enforced by a classifier on every Address before we render the order form.
2. **Sender identity mandatory.** Every DM, letter, order slip includes the sender's full name + context. No anonymous sends. Reduces harassment risk because the target knows *who* to respond to or block.
3. **One-shot cap.** Max one physical gesture per target per 30 days. Max one unsolicited outreach per target per 14 days. No exceptions (including "they didn't respond, send again").
4. **Target lookup + takedown.** Public URL `warm-dms.com/optout?email=...&phone=...` lets anyone request never-contact. Entries are hashed and every outgoing send checks the blocklist.
5. **Categorical blocklist.** No minors (school address of a K-12 is blocked), no politicians in certain jurisdictions, no people from sanctioned regions.
6. **No impersonation.** The sender cannot pretend to be someone else. Checked at profile setup.
7. **Content filters.** Drafts go through a safety classifier (toxicity, harassment, sexual, political-extreme). Anything flagged = blocked.
8. **Professional context required.** The intent must be professional (networking, recruiting, academic, commercial). Romantic outreach is out of scope and blocked.

### 10.2 Jurisdiction

MVP launch: US + India. Both have accepted cultures of unsolicited professional outreach via LinkedIn / email and of sending flowers / gifts to offices. EU / UK / Australia deferred — GDPR, anti-spam laws, and stricter harassment standards need legal review before we enable.

### 10.3 Terms of service risk

- LinkedIn: violated only if we automate. v0 doesn't. ✅
- X: same. ✅
- Crust Data: subject to their ToS for derivative data use. We need to confirm resale/redistribution is allowed for our use case. [OPEN QUESTION]
- Zomato / Ferns&Petals deep links: public URLs; deep-linking a cart is legal (browsing product pages is legal). Their ToS may restrict automated interaction — we don't automate, user clicks.

### 10.4 The reputational cliff

One viral Twitter thread — "warm-dms sent me an unsolicited pizza because I mentioned being hungry" — and the product is dead. Mitigations:
- Press the *sender identity mandatory* rule. Every gesture includes a note making it clear it's from the sender, not from "warm-dms."
- If a target reports a sender, sender's account gets reviewed; three reports = ban.
- Internal red-team: before MVP ships, actively send test gestures to friendly test-targets and ask for honest creepy/delightful scoring.

---

## 11. Risks & mitigations

| Risk | Likelihood | Impact | Mitigation |
|--|--|--|--|
| Crust API cost explosion | High | High | Batch polling, aggressive caching, per-user credit meter |
| LLM hallucinates a fact, letter goes out, sender credibility destroyed | Medium | Very High | Three-pass verification, user review mandatory, fact-check mode for top-risk fields |
| Target reports harassment → press coverage | Medium | Extreme | All §10 guardrails, sender identity, opt-out portal, 1-shot cap |
| Vendor deep-link breaks silently | High | Medium | Per-vendor integration tests nightly, fallback to Amazon gift card |
| LinkedIn account banned | Low (v0 no automation) | Medium | Humans send manually |
| Lob letter misprints address | Low | Low | Lob has 98%+ delivery, we show preview, sender confirms |
| Gemini rate-limited / down | Medium | High | Fallback to Claude API for drafting, pre-cached templates for degraded mode |
| Crust data stale / wrong | Medium | High | Evidence-required drafting means stale data just means we write around it, not lie |
| User misuse — romantic / stalking intent | Medium | Extreme | Intent classifier at session start, blocked if non-professional, TOS prohibits |
| GDPR-style complaint in a covered jurisdiction | Low (US+IN only) | High | Geography gate at signup, data retention policy, deletion API |

---

## 12. MVP scope (v0) → v1 → v2

### v0 (4-6 weeks, ship-for-design-partners)

**Goal:** prove the "goal → researched outreach" loop works for **founder→VC** and **PhD-applicant→prof** personas.

**Cut from v0:**
- Bookmark + monitor + cron polling (secondary flow entirely)
- Food delivery integration (flowers + letter only — stronger for both personas)
- Warm-moment alerting
- Voice (text input for MVP; add voice in v0.1)
- Auto-email sending (user copies for v0; OAuth Gmail in v0.1)

**In v0:**
1. Text-prompt intake (no voice yet)
2. Gemini intent parse
3. Crust people/org search → candidate list
4. Per-target research (Crust activity + web + arxiv)
5. Target list review UI
6. Per-target draft generation with evidence-required
7. Three-pass hallucination guard
8. Draft review UI with source highlighting
9. Copy-to-clipboard for DM
10. Lob letter submission (auto-mail)
11. Ferns&Petals / UrbanStems deep-link for optional flower gesture (v0.5 — cut if time short)
12. Sender profile + identity verification
13. Opt-out portal (simple form + email-to-block)

### v0.5 (2 weeks after v0)
- Voice input (Web Speech + Whisper fallback)
- Gmail OAuth + auto-send email
- Food deep-links (Zomato, DoorDash, Swiggy, UberEats)

### v1 (~3 months post-v0)
- Bookmark + cron monitoring + warm-moment alerting
- Job-search + BD personas
- Gmail thread tracking for replies + auto-follow-up drafts
- Pipeline analytics (response rates, meeting-booked rate)
- Team plans (multiple senders share a bookmark set)

### v2 (later)
- Partner APIs with DoorDash / Swiggy / Zomato
- LinkedIn Sales Navigator integration (paid tier)
- CRM integrations (HubSpot, Salesforce)
- Multi-sender campaigns (co-authored outreach)
- EU launch with GDPR compliance

---

## 13. Stack

**Chosen:**
- Frontend: Next.js 15 + Tailwind + shadcn/ui + tRPC
- Voice: Web Speech API → Whisper v3 large (via our backend)
- Backend: FastAPI (Python 3.12) + SQLAlchemy + asyncpg
- LLM: Gemini 2.5 Pro (reasoning, drafting) + Gemini Flash (classification, trigger matching)
- Secondary LLM: Claude Opus (fallback + high-stakes verification pass)
- DB: Postgres (Supabase)
- Cache/queue: Redis + Celery (or APScheduler for simpler cron)
- Auth: Supabase Auth (email + Google OAuth)
- Object store: Supabase storage
- Mail: Lob
- Transactional email: Resend
- Hosting: Fly.io (backend) + Vercel (frontend); scale to k8s later
- Monitoring: Sentry + PostHog + Axiom (structured logs)

**Why not:**
- LangGraph / CrewAI: opacity; hand-rolled loop is clearer for a verification-heavy agent.
- OpenAI / GPT-4: Gemini 2.5 Pro is equal-or-better at the specific tasks (long context for research synthesis, tool use), and Google's pricing is better at our expected volumes.
- Vercel AI SDK for orchestration: its abstractions conflict with our evidence-flow requirements; we pass it through our own tool layer.

---

## 14. Open questions (not blocking plan, blocking execution)

1. **Crust Data pricing tier** — blocks the cost model. Before writing code, confirm $/1000 calls and rate limits.
2. **Crust ToS on derivative storage** — can we persist enriched Target rows across sessions? Required for the "one Target row per person" model.
3. **Lob's template options for "feels-handwritten"** — is fontface + hand-addressed-envelope template in their standard offering?
4. **LinkedIn DM character limits in 2026** — last I checked ~300 chars for connection message, 8000 for InMail; verify current caps.
5. **Whisper v3 large cost at expected volume** — or is Web Speech API actually sufficient? A/B test during v0.
6. **Gemini 2.5 Pro context window + pricing for 2026** — the research synthesis step may want 500k+ tokens of context per target.

---

## 15. Success metrics (how we know it's working)

**Product:**
- Prompt-to-first-draft latency < 120 s at P50
- Per-target-draft latency < 15 s at P50
- Hallucination rate < 1 claim per 100 drafts (measured by spot-check + user report)

**Outcome:**
- Response rate vs sender's pre-warm-dms baseline (self-reported) — target 2x
- Meeting-booked rate among responded — target 30%
- Sender NPS at 4 weeks — target 50+

**Safety:**
- Target complaint rate per 1000 sends — target < 2
- Opt-out portal usage per 1000 sends — target < 5

---

## 16. Summary of decisions taken (vs deferred)

**Taken:**
- Goal-first conversational agent, not list-first CRM
- MVP personas: founder→VC, PhD-applicant→prof
- Stack: Next.js + FastAPI + Postgres + Gemini 2.5 Pro + Lob
- Channels v0: copy-paste for DMs, Lob for letters, email via user OAuth (v0.5)
- Hallucination guard: three-pass, evidence-required, user-review-mandatory
- Physical gesture v0: **Lob letters + Ferns&Petals / UrbanStems deep-links only**. Food is v0.5+.
- Hard rules: office/school only, sender identity mandatory, 1-shot cap, opt-out portal
- Jurisdiction v0: US + India

**Deferred:**
- Crust pricing → before first code
- Voice latency choice (Whisper vs Deepgram) → during v0
- LinkedIn automation → likely never for v0; revisit in v2 with official Sales Navigator API
- Bookmark + monitor + cron → v1
- Food delivery integration → v0.5

---

*End of draft v0.1. Awaiting reviewer passes.*
