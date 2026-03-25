# Shipday AI Sales Agent -- Architecture

## System Overview

```
              +--------------+--------------+
              |                             |
    +---------+---------+        +----------+----------+
    |   Next.js 14      |        |   Voice Agent       |
    |   (Port 3000)     |        |   (Port 3006)       |
    |                   |        |                     |
    |   App Router      |        |   Express Server    |
    |   API Routes      |        |   WebSocket Handler |
    |   React UI        |        |   SSE Feed (3007)   |
    +---------+---------+        +----------+----------+
              |                             |
              +-------------+---------------+
                            |
                 +----------+----------+
                 |   PostgreSQL        |
                 |                     |
                 |   Agent tables:     |
                 |   - crm.contacts    |
                 |   - crm.touchpoints |
                 |   - crm.scheduling_*|
                 |   - brain.*         |
                 +---------------------+
```

## Web Chatbot Architecture

### Request Lifecycle

```
Client (Browser/Widget)
  |
  | POST /api/chat/prospect
  | Body: { messages[], orgId, sessionId, visitorContext }
  |
  v
+--------------------------------------------------+
|  API Route (route.ts, 880 lines)                 |
|                                                  |
|  1. Rate Limiting (per-IP)                       |
|  2. Session Resolution                           |
|  3. Brain Loading (cached):                      |
|     - Internal content                           |
|     - Call patterns (effectiveness-scored)        |
|     - Live deal stats (win rate, MRR, phrases)   |
|     - Social proof statements                    |
|     - Business lookup (Google rating, reviews)   |
|  4. Guardrail Checks:                            |
|     - PII fence (hard reject)                    |
|     - Topic fence (redirect)                     |
|     - Escalation detection                       |
|     - Quality scoring                            |
|     - Length controls                            |
|  5. ROI Computation (if 3+ slots filled)         |
|  6. Calendar Tool Setup (event type resolution)  |
+--------------------------------------------------+
  |
  v
+--------------------------------------------------+
|  AI Engine (ai.ts, 2,914 lines)                  |
|  Function: prospectChat()                        |
|                                                  |
|  System Prompt Assembly (400+ lines):            |
|  +--------------------------------------------+ |
|  | Identity Block                              | |
|  | Conversation Rules (1Q/turn, no em-dash)    | |
|  | Guardrail Context                           | |
|  | Qualification State (13 slots + stage)      | |
|  | Brain Knowledge (patterns, phrases, stats)  | |
|  | ROI Data (computed savings + chart)          | |
|  | Calendar Instructions                       | |
|  | Tier-Specific Tone + Step Energy            | |
|  +--------------------------------------------+ |
|                                                  |
|  Model Selection:                                |
|  - Haiku: early discovery (cost-efficient)       |
|  - Sonnet: closing + tool calling (accurate)     |
|                                                  |
|  Tool Calling Loop (max 3 iterations):           |
|  +--------------------------------------------+ |
|  | check_availability:                         | |
|  |   -> computeAvailableSlots()               | |
|  |   -> Google Calendar FreeBusy API          | |
|  |   -> Filter by constraints + conflicts     | |
|  |                                            | |
|  | book_demo:                                  | |
|  |   -> createBooking() (FOR UPDATE lock)     | |
|  |   -> Google Calendar event + Meet link     | |
|  |   -> Contact link + touchpoint             | |
|  |   -> Confirmation email                    | |
|  +--------------------------------------------+ |
+--------------------------------------------------+
  |
  v
+--------------------------------------------------+
|  Post-Processing                                 |
|                                                  |
|  - Extract qualification slots from response     |
|  - Extract lead info (name, email, company)      |
|  - Generate suggested follow-up prompts          |
|  - Strip em-dashes (safety net)                  |
|  - Log conversation (PII-redacted)               |
+--------------------------------------------------+
  |
  | Response: {
  |   reply, qualification, suggested_prompts,
  |   roi_chart, lead_captured, quality_score,
  |   escalation, guardrail_triggered, ...
  | }
  v
Client
```

### Guardrail System Detail

```
Input
  |
  v
+--[ PII Fence ]--+  Hard reject: SSN, credit card, bank account,
|  PASS  |  BLOCK  |  driver license, passport patterns
+--------+---------+
  |
  v
+--[ Topic Fence ]-+  Redirect: politics, religion, legal,
|  PASS  |  BLOCK   |  investments, personal advice
+---------+---------+
  |
  v
+--[ Pricing Fence ]+  Route to human: discount requests,
|  PASS  |  ESCALATE |  free trials, price negotiation
+---------+----------+
  |
  v
+--[ Competitor Fence ]+  Reframe: competitor mentions,
|  PASS  |  REFRAME    |  force value-differentiation
+---------+-------------+
  |
  v
+--[ Promise Fence ]+  Soften: guarantees, certainties,
|  PASS  |  SOFTEN   |  enforce "typically see" language
+---------+----------+
  |
  v
  Quality Score (0-100):
    - Relevance: 30%
    - Pipeline Advancement: 40%
    - Question Quality: 30%
  |
  Escalation Detection:
    - Frustration patterns (keyword weights)
    - Engagement decline tracking
    - Action: immediate_handoff | offer_human | adjust_tone
```

## Voice Agent Architecture

```
Twilio (PSTN)
  |
  | WebSocket (Media Stream)
  | mulaw 8kHz mono audio
  |
  v
+--------------------------------------------------+
|  Server (server.ts, 618 lines)                   |
|                                                  |
|  WebSocket Events:                               |
|  - connected: init session                       |
|  - start: configure stream                       |
|  - media: route audio to STT                     |
|  - stop: end call, post-processing               |
+--------------------------------------------------+
  |                          |
  | audio bytes              | processed text
  v                          v
+------------------+  +---------------------------+
| Deepgram STT     |  | Conversation Manager      |
| (stt.ts, 242 ln) |  | (conv-mgr.ts, 1,054 ln)  |
|                  |  |                           |
| Nova-2 model     |  | Qualification Extraction  |
| Real-time WS     |  | (13 slots, auto-staging)  |
| VAD events       |  |                           |
| Word timings     |  | Voice System Prompt       |
|   -> WPM calc    |  | (1-2 sentences, no fmt)   |
|   -> pacing      |  |                           |
| Confidence       |  | Claude API + Tools        |
|   -> quality     |  | (check_avail, book_demo)  |
|   monitoring     |  |                           |
+------------------+  | Handoff Triggers:         |
                      | - prospect_request        |
                      | - pricing_negotiation     |
                      | - high_intent             |
                      | - emotional_escalation    |
                      | - stalled_conversation    |
                      +---------------------------+
                                  |
                                  | text response
                                  v
                      +---------------------------+
                      | ElevenLabs TTS            |
                      | (tts.ts, 358 lines)       |
                      |                           |
                      | Turbo v2.5                |
                      | Dynamic speed (WPM match) |
                      | Filler cache (8 phrases)  |
                      | Strategic pause (ROI)     |
                      | Barge-in abort            |
                      +---------------------------+
                                  |
                                  | mulaw audio
                                  v
                              Twilio -> Caller

Side Channels:
+---------------------------+  +---------------------------+
| Call Quality              |  | SSE Dashboard Feed        |
| (call-quality.ts)         |  | (sse.ts, 106 lines)       |
|                           |  |                           |
| STT confidence EMA        |  | Events: call_started,     |
| Error counts              |  | transcript_update,        |
| Reconnect tracking        |  | stage_change, roi_computed|
| Degraded mode trigger     |  | handoff_triggered,        |
+---------------------------+  | call_ended                |
                               +---------------------------+

Post-Call:
+---------------------------+
| Post-Call Processing      |
| (post-call.ts, 266 lines)|
|                           |
| Transcript archival       |
| Pattern extraction        |
| Contact update            |
| Touchpoint creation       |
| Brain learning pipeline   |
+---------------------------+
```

## Data Dependencies

The agent touches these database tables:

```
crm schema (scheduling + contacts):
  scheduling_event_types     -- Booking configurations
  scheduling_bookings        -- Confirmed bookings
  scheduling_availability    -- Weekly hours + overrides
  calendar_connections       -- Google OAuth tokens (encrypted)
  contacts                   -- Prospect records (upserted from chat)
  touchpoints                -- Activity history (all channels)

brain schema (knowledge + learning):
  internal_content           -- Knowledge base entries
  call_patterns              -- Extracted winning patterns
  conversation_logs          -- PII-redacted conversation logs
```

## External Service Integrations

```
+-------------------+     +-------------------+     +-------------------+
| Anthropic Claude  |     | Google Calendar   |     | Twilio            |
| - Chat (Sonnet)   |     | - OAuth2 flow     |     | - Media Streams   |
| - Chat (Haiku)    |     | - FreeBusy API    |     | - Phone numbers   |
| - Voice (Sonnet)  |     | - Event creation  |     | - Conference      |
| - Tool calling    |     | - Meet links      |     | - Call recording  |
+-------------------+     +-------------------+     +-------------------+

+-------------------+     +-------------------+
| Deepgram          |     | ElevenLabs        |
| - Nova-2 STT      |     | - Turbo v2.5 TTS  |
| - Real-time WS    |     | - Voice cloning   |
| - VAD events      |     | - Speed control   |
| - Word timings    |     | - Filler cache    |
+-------------------+     +-------------------+
```
