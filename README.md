# TripPilot ✈️ — Virtual Travel Agent

**VoyageHack 4.0 · Problem Statement: Virtual Travel Agent**

TripPilot is an agentic virtual travel agent that behaves like a knowledgeable human consultant across the whole travel lifecycle: inspiration → planning → booking → trip management → post-booking support. It turns a traveller's intent ("5 days in Bali in March, ₹1.5L, love beaches and food") into a personalised, priced, bookable itinerary, and keeps looking after the trip afterwards.

---

## The Problem We're Solving

Traditional travel chatbots answer questions. They don't *do* things. Travellers still juggle many tabs for flights, hotels, transfers and activities, and are left on their own when a flight is delayed or a booking has to change.

TripPilot goes further. It:

- Understands preferences, budget, dates, interests and constraints through natural conversation, asking follow-up questions when something is missing.
- Recommends destinations and builds itineraries that combine flights, hotels, transfers, sightseeing and experiences.
- Discovers and compares options, verifies live price and availability, and estimates total trip cost.
- Refines the plan dynamically when preferences or constraints change.
- Supports the journey after booking: confirmations, vouchers, reservation questions, cancellations, modifications, rebooking and disruptions.
- Always explains the options and implications, and **asks for confirmation before any consequential action** (booking, paying, cancelling).

---

# TripPilot: Technical Approach

## 1. Design Goals

| Goal | Technical meaning |
|---|---|
| Agentic, not a chatbot | LLM plans and calls typed tools in a loop. It does not only generate text. |
| Reliable | Deterministic code owns money, state and validation. The LLM never executes side effects directly. |
| Safe | Every consequential action passes a confirmation gate that the LLM cannot bypass. |
| Personalised | Long-term preference memory is retrieved and applied during ranking. |
| Demoable end to end | Every partner API has a real-sandbox adapter *and* a simulated mock with the same interface. |

**Core principle:** *The LLM proposes, the code disposes.* The model decides what to do next. Typed tools, validators and the gate decide whether it is allowed and actually do it.

---

## 2. The Agent Loop

### 2.1 Orchestration pattern

We use **LLM function calling in a bounded loop** (ReAct style):

```
while not done and steps < MAX_STEPS:
    context  = build_context(session_state, memory, tool_results)
    decision = llm(context, tool_schemas)        # text reply OR tool call(s)
    if decision.is_tool_call:
        results = router.execute(decision.calls) # parallel where independent
        state.append(results)
    else:
        respond(decision.text); done = True
```

Safeguards on the loop:
- `MAX_STEPS` cap and a token/cost budget per turn.
- Per-tool timeout, retry with backoff, and a circuit breaker for failing partners.
- Parallel execution for independent calls (flights, hotels and activities in one step).

### 2.2 Conversation state machine

The logical flow is also modelled as an explicit state machine, so progress is inspectable and resumable rather than implicit in the prompt.

| State | Entry condition | Exit |
|---|---|---|
| `UNDERSTANDING` | New or changed user message | Slots complete → `SEARCHING`; else ask follow-up |
| `SEARCHING` | Slots complete | Results validated → `PROPOSING`; budget/availability fail → re-rank |
| `PROPOSING` | Priced itinerary ready | User edits → `REFINING`; user approves → `AWAITING_CONFIRMATION` |
| `REFINING` | Edit request | Re-price → `PROPOSING` |
| `AWAITING_CONFIRMATION` | Plan approved | Explicit "yes" → `EXECUTING` |
| `EXECUTING` | Confirmed | Book → pay → vouchers → `MONITORING` |
| `MONITORING` | Trip booked | Disruption event → `REPLANNING` |
| `REPLANNING` | Disruption/user change | Alternatives proposed → `AWAITING_CONFIRMATION` |

### 2.3 Natural language understanding and slot filling

The LLM extracts a structured **TripIntent** using schema-constrained output (JSON schema / structured outputs):

```json
{
  "origin": "HYD",
  "destination_candidates": ["Bali", "Phuket"],
  "dates": {"start": "2027-03-10", "end": "2027-03-15", "flexible_days": 2},
  "travellers": {"adults": 2, "children": 0},
  "budget": {"amount": 150000, "currency": "INR", "scope": "total"},
  "interests": ["beaches", "food"],
  "constraints": ["no red-eye flights", "vegetarian-friendly"],
  "missing_slots": ["origin"]
}
```
---

## 3. Tool Layer (MCP + REST Adapters)

Every capability is a **typed, idempotent tool** with a strict schema. Tools are exposed to the LLM via the Model Context Protocol (MCP) or a thin REST wrapper.

### 3.1 Tool catalogue

| Tool | Type | Description |
|---|---|---|
| `search_flights(origin, dest, dates, pax, filters)` | read | Returns ranked fares |
| `search_hotels(dest, dates, pax, filters)` | read | Returns rooms and rates |
| `search_transfers(from, to, time, pax)` | read | Airport/hotel transfers |
| `search_activities(dest, dates, interests)` | read | Sightseeing and experiences |
| `verify_offer(offer_id)` | read | Re-checks live price and availability |
| `get_weather(dest, dates)` / `get_flight_status(flight)` | read | Context and disruption data |
| `estimate_trip_cost(itinerary)` | compute | Deterministic cost roll-up |
| `create_booking(offer_id, idempotency_key)` | **write** | Gated |
| `charge_payment(booking_id, idempotency_key)` | **write / money** | Gated |
| `cancel_booking(booking_id)` / `modify_booking(booking_id, changes)` | **write** | Gated |
| `get_booking(booking_id)` / `get_voucher(booking_id)` | read | Trip management |

### 3.2 Tool contract (example)

```json
{
  "name": "create_booking",
  "risk_tier": "HIGH",
  "requires_confirmation": true,
  "idempotent": true,
  "input_schema": {
    "type": "object",
    "required": ["offer_id", "idempotency_key", "confirmation_token"],
    "properties": {
      "offer_id": {"type": "string"},
      "idempotency_key": {"type": "string"},
      "confirmation_token": {"type": "string"}
    }
  }
}
```

### 3.3 Adapter pattern: real and simulated behind one interface

```
        ┌── SandboxFlightsAdapter  (real API sandbox)
FlightsTool ┤
        └── MockFlightsAdapter     (simulated, deterministic, can inject faults)
```

- Selected by `USE_MOCK_APIS` or per-partner config.
- Mocks return realistic, seedable data and can **simulate failures, price changes and webhooks** (delay, cancellation) on demand. This makes the demo and the tests reproducible.

---

## 4. Safety: Confirmation Gate and Guardrails

### 4.1 Risk tiers

| Tier | Examples | Policy |
|---|---|---|
| **LOW** (read) | search, status, get voucher | Auto-execute |
| **MEDIUM** | save/hold an itinerary, change preferences | Execute and notify |
| **HIGH** (consequential) | book, pay, cancel, modify, rebook | **Explicit user confirmation required** |

### 4.2 Confirmation gate (enforced in code, not in the prompt)

1. The LLM requests a HIGH-risk tool. The router **intercepts** it.
2. The gate builds a **human-readable summary**: what will happen, total cost, cancellation terms and refund impact.
3. The summary is shown to the user. The system issues a **single-use `confirmation_token`** bound to the exact action, amount and offer.
4. Only after the user explicitly approves does the token unlock the tool call.
5. If the price or availability changed between summary and execution, the token is **invalidated** and the user is re-prompted.

The LLM cannot execute a HIGH-risk action by itself, because the tool rejects any call without a valid token.

### 4.3 Other guardrails

- **Input and output validation:** JSON-schema validation on every tool call and result.
- **Grounding:** prices, availability and policies shown to users must come from a tool result, never from LLM memory. Responses carry the offer ID and its verification timestamp.
- **Prompt-injection hygiene:** partner and web content is treated as data, never as instructions.
- **PII minimisation:** payment data goes straight to the payment provider (tokenised) and is never stored in prompts or logs.
- **Authorisation:** every tool call is scoped to the authenticated user's own trips.

---

## 5. Reliability Engineering

| Concern | Mechanism |
|---|---|
| Duplicate bookings or charges on retry | **Idempotency keys** on all write tools, stored and checked server-side |
| Stale prices | `verify_offer` before presenting a total and again immediately before booking; short-TTL Redis price cache |
| Partial failure (flight booked, hotel failed) | **Saga pattern** with compensating actions (cancel or refund steps) and a clear user explanation |
| Partner outage or slow API | Timeouts, retries with backoff, circuit breaker, fallback to alternate supplier or mock-labelled result |
| LLM errors or invalid tool args | Schema validation, one self-repair retry, then a graceful clarification to the user |
| Crash or restart | Stateless workers plus persisted session and booking state |

### Booking saga (example)

```
reserve_flight ─► reserve_hotel ─► reserve_transfer ─► charge_payment ─► issue_vouchers
      │ fail           │ fail           │ fail              │ fail
      ▼                ▼                ▼                   ▼
  release holds in reverse order, refund if charged, then explain and offer alternatives
```

---

## 6. Personalisation and Memory

Two memory layers:

1. **Structured profile (PostgreSQL):** home airport, passport/visa notes, dietary needs, budget bands, travel companions.
2. **Preference memory (pgvector):** embeddings of learned preferences ("likes boutique hotels", "avoids long layovers") and past-trip feedback.

How it is used:
- At `SEARCHING`, the top-k relevant preferences are retrieved and injected into the context.
- During ranking, a **scoring function** combines price, duration, rating and a preference-match score. The LLM explains the ranking but does not compute it.
- After a trip or edit, new preferences are extracted and written back, with the user able to view and delete them.

---

## 7. Data Model

```
users(id, profile_json, created_at)
sessions(id, user_id, state, intent_json, plan_json, updated_at)
offers(id, session_id, type, supplier, price, currency, expires_at, verified_at)
itineraries(id, session_id, version, items_json, total_cost, status)
bookings(id, itinerary_id, supplier_ref, status, idempotency_key, voucher_url)
payments(id, booking_id, provider_ref, amount, status, idempotency_key)
preferences(id, user_id, text, embedding VECTOR, source, created_at)
events(id, type, payload_json, matched_booking_id, handled_at)
traces(id, session_id, step, tool, args, result_summary, latency_ms, tokens)
```

---

## 8. API Surface (backend)

| Endpoint | Purpose |
|---|---|
| `POST /v1/chat` (WebSocket upgrade) | Stream agent responses and tool progress |
| `POST /v1/confirm` | Submit approval for a pending action (`confirmation_token`) |
| `GET /v1/trips` / `GET /v1/trips/{id}` | List trips and view itinerary, vouchers and status |
| `POST /v1/webhooks/{supplier}` | Receive partner events (signature-verified) |
| `GET /v1/health` | Liveness and dependency checks |

---

## 9. Tech Stack

| Layer | Choice |
|---|---|
| Frontend | React (chat UI, itinerary view, confirmation modal) |
| API | Python FastAPI (async) with WebSockets |
| Agent | LLM with function calling and structured outputs; MCP tool servers |
| State | PostgreSQL |
| Cache / sessions | Redis |
| Vector search | pgvector |
| Events | Redis Streams or a lightweight message queue |
| Payments | Stripe / PayPal (sandbox) |
| Travel data | Partner sandboxes or in-house mock services |
| Deployment | Docker, Kubernetes-ready stateless workers |
| Observability | Structured logs plus tracing (OpenTelemetry-style) |


