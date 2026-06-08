# ServNest Conversational Voice Agent Implementation Plan

> Grounded in the live stack inspected on 2026-06-08: ServNest CRM in Dokploy, active n8n workflows, healthy backend, Postgres/Redis/MinIO present, ElevenLabs + Vonage envs configured.

## Goal
Turn the current bounded outbound estimate-calling workflow into a production-safe conversational voice agent without breaking the working booking flow.

## Current live baseline

### Infrastructure already available
- Dokploy project: **ServNest CRM**
- Live containers:
  - `servnest-894oon-backend-1` (healthy)
  - `servnest-894oon-frontend-1`
  - `servnest-894oon-postgres-1`
  - `servnest-894oon-redis-1`
  - `servnest-894oon-minio-1`
- n8n instance live at `https://n8n.rivsconsole.shop/`
- Active relevant workflows:
  - `ServNest Estimate Voice Agent`
  - `Vonage Voice Answer`
  - `Vonage Voice Events`

### What is already implemented
- Website estimate lead enters ServNest
- ServNest fires `estimate.lead.created`
- n8n creates outbound Vonage call
- Vonage answer/selection/events endpoints exist
- ElevenLabs greeting generation is wired
- Current call UX is DTMF-driven slot selection
- Booking result is written back into ServNest
- `voice_call_sessions` table exists live
- Voice Agent settings UI exists with call history/detail support

### What is not implemented yet
- Caller speech recognition
- Realtime conversational loop
- Transcript population
- Extracted intent population
- Human handoff/transfer flow
- Broader receptionist behavior

## Architecture target
Keep **ServNest backend** as the source of truth, **Postgres/Redis/MinIO** as core infra, and **n8n** only for async orchestration. Move latency-sensitive voice logic into ServNest first, then add a realtime conversational runtime later.

---

# Phase 0 — Lock the current foundation

## Objective
Make the existing system deterministic, deployable, and observable before adding speech or LLM behavior.

## 0.1 Clean the source-of-truth mismatch
Right now the live runtime contains new voice-agent code, but the VPS checkout is still dirty on `feat/voice-agent-hardening`.

### Required actions
- Commit the current voice-agent changes cleanly
- Ensure the deployed image maps to that commit
- Eliminate the current runtime-vs-repo ambiguity

### Files already involved
- `backend/src/modules/public/public-booking.service.ts`
- `backend/src/modules/webhooks/webhooks.controller.ts`
- `backend/src/modules/tenants/voice-agent-calls.service.ts`
- `backend/src/entities/voice-call-session.entity.ts`
- `backend/src/migrations/1774100000000-AddVoiceCallSessions.ts`
- frontend settings files under `frontend/src/features/settings/`

## 0.2 Make ServNest the primary call-control owner
### Target split
**ServNest owns:**
- lead state
- call session state
- answer NCCO generation
- slot offer logic
- selection validation
- booking writeback

**n8n owns:**
- async trigger/orchestration
- retries/campaign scheduling
- notifications
- non-latency-sensitive side effects

## 0.3 Replace `/tmp` audio cache with durable object storage
Current estimate-agent audio caching is container-local under `/tmp`.

### Required change
Move generated greeting assets to **MinIO** using a stable fingerprinted object key.

### Implementation direction
- Generate cache key from tenant/job/customer/service fingerprint
- Store generated mp3 in MinIO
- Return object URL usable by Vonage
- Keep cache-hit behavior, but back it with MinIO instead of local disk

### Why
`/tmp` is fragile and container-local; MinIO is already live and is the correct persistence layer.

## 0.4 Normalize terminal outcomes
Required terminal outcomes:
- `answered_booked`
- `answered_declined`
- `answered_callback_requested`
- `voicemail`
- `no_answer`
- `invalid_number`
- `failed_transport`
- `selection_timeout`
- `selection_invalid`
- `booking_save_failed`

## 0.5 Add end-to-end smoke validation
Verify:
- public estimate lead creates job
- `estimate.lead.created` fires
- n8n outbound call workflow runs
- Vonage answer webhook returns valid NCCO
- selection webhook books correctly
- `voice_call_sessions` updates correctly
- settings UI reflects final result

---

# Phase 1 — Harden the bounded voice agent

## Objective
Make the current DTMF-driven system production-grade before conversational expansion.

## 1.1 Formalize `voice_call_sessions` as a state machine
Required phases:
- `lead_created`
- `call_queued`
- `dialing`
- `answered`
- `prompted`
- `awaiting_selection`
- `booked`
- `followup_required`
- `completed`
- `failed`

### Persist on each session
- `provider`
- `provider_call_id`
- `attempt_number`
- `offered_slots_json`
- `selected_slot_json`
- `terminal_reason`
- `next_action`
- `recording_url` (future-compatible)
- `transcript_text` placeholder

## 1.2 Add retry policy in ServNest
Rules should live in ServNest even if n8n schedules execution.

### Required rules
- no answer → retry later
- voicemail → send SMS or retry once
- invalid selection → SMS scheduling link
- booking save failed → office alert
- repeated failure → human follow-up queue

## 1.3 Add separate voicemail / no-answer / timeout branches
Explicitly handle:
- voicemail detected
- timeout/no selection
- no answer
- transport failure before answer

## 1.4 Improve operator visibility in the Settings UI
Add to Voice Agent details:
- current phase
- terminal outcome
- attempt count
- offered slots
- selected slot
- timestamps
- next action
- manual retry trigger
- manual mark-for-human-followup trigger

---

# Phase 2 — Add speech-first interaction without a separate runtime yet

## Objective
Upgrade from keypad-only to speech + DTMF fallback while preserving the deterministic booking engine.

## 2.1 Add speech input to the current call flow
Current implementation is DTMF-only. Add speech alongside DTMF.

### Allowed intents for v1 speech
- choose slot 1/2/3
- repeat options
- request callback
- decline
- wrong number
- request office/operator

## 2.2 Add speech normalization in ServNest
Map raw speech into bounded normalized actions:
- `select_slot_1`
- `select_slot_2`
- `select_slot_3`
- `repeat_options`
- `request_callback`
- `decline`
- `wrong_number`
- `operator_request`
- `unrecognized`

## 2.3 Start populating transcript and extracted intents
For each session, persist:
- recognized utterance
- normalized intent
- confidence
- speech-vs-dtmf source
- transcript text

## 2.4 Keep DTMF as permanent fallback
Low-confidence speech should fall back to repeat prompt, keypad entry, and eventually follow-up.

---

# Phase 3 — Introduce a true conversational runtime

## Objective
Add multi-turn dialogue only after the bounded speech-first flow is stable.

## Runtime target
Deploy a dedicated voice runtime service inside the same Dokploy environment.

### Responsibilities of the new runtime
- turn-taking
- short-lived conversation context
- clarification/repeat logic
- calling ServNest tools for business actions

### ServNest remains authoritative for
- availability truth
- booking writes
- customer/job state
- persistent call/session history
- escalation rules

### n8n remains useful for
- retries
- campaigns
- notifications
- post-call automation

## Provider requirement for this phase
The current live backend does not expose envs for a speech/LLM provider beyond ElevenLabs. This phase requires adding at least one conversational/STT provider.

Recommendation:
- keep ElevenLabs for current/polished TTS if latency is acceptable
- add a speech/LLM provider for ASR + orchestration
- do not replace the current bounded booking engine

---

# Phase 4 — Human handoff and production safety

## Objective
Make the system safe for real customers and office operations.

## Add handoff triggers
- operator request
- low confidence after N turns
- unsupported question
- angry/frustrated signals
- booking tool failure
- repeated loop

## Persist handoff artifact
Store:
- lead/job link
- summary
- transcript
- extracted intent
- recommended next action
- slot discussion state

## Use n8n for async follow-up
- office/Discord notification
- callback queue
- task creation
- SMS fallback

---

# Phase 5 — Expand to broader receptionist behavior

## Expansion order
1. estimate scheduling
2. callback / reschedule
3. simple FAQ
4. service routing
5. office handoff
6. broader receptionist behaviors

Do **not** jump directly to unrestricted pricing/troubleshooting/office-manager behavior.

---

# Concrete execution order

## Step 1 — immediate
1. Clean/commit/deploy current voice-agent code
2. Move audio cache from `/tmp` to MinIO
3. Finalize session state machine fields
4. Add normalized terminal outcomes
5. Add real lead-to-booking smoke validation
6. Expose richer call detail in frontend

## Step 2 — near-term hardening
7. Add voicemail/no-answer/follow-up branches
8. Add retry policy owned by ServNest
9. Keep n8n only for orchestration and notifications
10. Verify active n8n voice workflows still align with backend assumptions

## Step 3 — biggest UX upgrade before full AI
11. Add speech input alongside DTMF
12. Normalize speech intents into bounded actions
13. Populate transcript/intents in `voice_call_sessions`
14. Surface speech confidence/fallback reasons in the UI

## Step 4 — real conversational agent
15. Add dedicated voice runtime service in Dokploy
16. Integrate speech/LLM provider
17. Use ServNest as the tool layer
18. Add human handoff + transcript summary + callback automation

---

# Keep vs change

## Keep
- ServNest as source of truth
- Dokploy deployment model
- Postgres/Redis/MinIO
- n8n for async orchestration
- ElevenLabs for current TTS
- DTMF fallback forever

## Change
- stop relying on local `/tmp` audio cache
- stop treating n8n as the long-term answer-path brain
- add bounded speech before unrestricted conversation
- add provider/envs for true ASR/LLM only after bounded speech is stable

---

# Success gates

## Phase 0 is done when
- repo and deployed runtime match
- MinIO-backed prompt assets work
- voice flow passes real smoke validation

## Phase 1 is done when
- every call has normalized outcome
- retries/follow-ups are deterministic
- settings UI is enough to debug without logs

## Phase 2 is done when
- callers can speak or press keys
- speech maps cleanly into bounded intents
- transcript + confidence are stored

## Phase 3 is done when
- multi-turn dialogue works
- bookings still require ServNest-confirmed tools
- low-confidence cases escalate safely

## Phase 4 is done when
- office receives useful summaries
- failures do not silently die
- callback queue is operationally usable

---

# Direct recommendation
Do **not** build a separate AI phone-agent platform first.

Best path with the current live stack:
1. harden the bounded ServNest/Vonage flow
2. move to speech + DTMF
3. then add a conversational runtime on top of the same CRM/state machine

That path uses infrastructure you already have live and reduces delivery risk.
