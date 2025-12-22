# HR-Project — Comprehensive Technical Report (Dec 22, 2025)

## 1. Executive Summary
This repository implements an **HR task-oriented conversational agent**. The system:
- Accepts a user message (typically via a Flask API).
- Detects what HR task the user wants (intent).
- Collects required information (slots) in a guided, step-by-step flow.
- Executes deterministic actions (e.g., Slack notifications, optional SMS via Twilio).
- Logs conversations and action executions for auditability (commonly via Supabase or a storage layer).

The architecture emphasizes **deterministic workflow execution**: the “brain” is a controlled dialog manager / state machine, while any LLM usage (if enabled) is used mainly for **natural language phrasing** (e.g., rewriting questions, composing messages), not for deciding business logic.

---

## 2. What the Project Does (User-Facing Behavior)
The bot supports HR workflows such as:
- Time-off requests
- Meeting scheduling
- IT ticket submission
- Medical claim filing

For each workflow, the bot:
1. Identifies the intended task (intent).
2. Determines which required fields are missing (slots).
3. Asks the user targeted questions to collect those fields.
4. Normalizes/validates values when needed (e.g., dates, phone numbers).
5. Once all required details are collected, triggers an action (notifications, ticket creation, etc.).
6. Produces a final confirmation response to the user and logs the action outcome.

---

## 3. High-Level Architecture
### 3.1 Main components (conceptual)
- **API/UI Layer**: Receives user messages and returns assistant replies.
- **Agent Orchestrator**: Coordinates storage, intent routing, dialog manager, and action execution.
- **Intent Router**: Maps free-form user text to a known task type (or “general chat”).
- **Dialogue Manager + FSM**: Controls conversation state and slot filling deterministically.
- **Slots subsystem**: Defines schemas, extracts candidate values, chooses next missing slot to ask.
- **Actions subsystem**: Executes side effects (Slack messages, SMS notifications via Twilio, etc.).
- **Storage/Audit**: Persists conversations, states, messages, and action execution logs.

### 3.2 Typical message lifecycle (end-to-end)
1. **User sends message** → API endpoint receives it.
2. API forwards message to **agent** with a conversation/session id.
3. Agent loads prior state from storage (if any).
4. **Intent detection** determines which workflow applies.
5. **FSM** transitions:
   - INIT → collecting → ready_to_execute → completed (names vary)
6. **Slot filling**:
   - Extract slot candidates from user text
   - Validate/normalize
   - Ask the next missing slot question
7. When all required slots are present:
   - Build action payload
   - Execute deterministic action(s)
8. Log results and respond to the user.

---

## 4. Repository Structure (What Each Folder Is For)
> Adjust this section if your actual tree differs.

### 4.1 `slots/` — Slot definitions & slot-filling utilities
Files:
- `slots/schemas.py`
  - Defines the *data schema* for each intent: required slots, optional slots, types, validation rules, etc.
- `slots/slot_extractor.py`
  - Extracts slot values from user input (e.g., dates, times, names, departments, phone numbers).
- `slots/slot_selector.py`
  - Chooses the “next best question” (which slot to ask next) based on the schema and current state.
- `slots/__init__.py`
  - Module exports / packaging.

**Purpose**: Make the conversation deterministic by using known slot requirements per intent.

### 4.2 `dialogue/` — Conversation state & deterministic flow
Common responsibilities:
- Maintains a structured state (current intent, collected slots, missing slots, last asked slot, retries).
- Moves through an FSM so the agent doesn’t “wander” or hallucinate flows.
- Produces the next assistant action:
  - Ask a slot question
  - Retry slot
  - Execute action
  - Finish

### 4.3 `actions/` — Real-world side effects (no LLM reasoning)
Typical responsibilities:
- Send Slack messages to HR/IT/manager channels.
- Send SMS notifications via Twilio.
- Return structured results `{ success, error, ids }` so the agent can log and confirm.

#### Example: `actions/twilio_service.py`
The `TwilioService` is a deterministic SMS sender:
- Loads credentials from constructor parameters or environment variables:
  - `TWILIO_ACCOUNT_SID`
  - `TWILIO_AUTH_TOKEN`
  - `TWILIO_FROM_NUMBER`
- Normalizes phone numbers (ensures `+` prefix).
- Validates SID format (warns if not starting with `AC`, and blocks if `SK...` is used as SID).
- Sends SMS with `twilio.rest.Client.messages.create(...)`.
- Returns structured results:
  - `{'success': True/False, 'sid': ..., 'error': ...}`
- Adds helpful tips for common Twilio errors (e.g., FROM number not owned by Twilio).

**What you’re using here**:
- Twilio Python SDK (`twilio` package)
- Environment-variable based configuration
- Deterministic error handling for operations

### 4.4 `llm/` (if present) — Natural language assistance (phrasing, normalization)
Typical responsibilities (project-dependent):
- Rewrite slot questions to be more natural.
- Compose Slack/SMS messages from structured slot values.
- Normalize ambiguous values (e.g., “next Monday”) into canonical forms.

**Important**: LLM output should not directly decide business logic—only text phrasing.

### 4.5 `interface/` or `api/` — Web server endpoints
Typical responsibilities:
- Flask endpoints such as `POST /api/chat`
- Serves UI assets (if any)
- Handles CORS, JSON parsing, returning responses

### 4.6 Storage (Supabase / DB / logging)
Typical responsibilities:
- Persist `conversations`
- Persist `messages` (user/bot)
- Persist `fsm_states`
- Persist `action_executions`

If Supabase is used, a SQL schema file typically exists defining these tables.

---

## 5. External Services & Dependencies
### 5.1 Twilio (SMS)
- Used for sending SMS notifications (manager alerts, confirmations, escalations).
- Requires:
  - A valid Account SID starting with `AC...`
  - Auth token
  - A Twilio-owned `from` phone number

### 5.2 Slack (chat notifications) — if configured
- Used to notify HR/IT channels when a workflow completes.
- Usually requires a bot token and channel IDs.

### 5.3 LLM provider (optional, project-dependent)
- Used for language generation tasks (rewrite/compose), not deterministic logic.
- Requires an API key if enabled.

### 5.4 Python runtime & libraries
- Flask (if API server exists)
- Twilio SDK
- Any DB client (Supabase / Postgres)
- Any date parsing libs (if used by extractor/normalizer)

---

## 6. Configuration & Environment Variables (Typical)
- `TWILIO_ACCOUNT_SID`
- `TWILIO_AUTH_TOKEN`
- `TWILIO_FROM_NUMBER`
- (Optional) Slack tokens/channel IDs
- (Optional) DB/Supabase URL + service key
- (Optional) LLM API keys

Store these in:
- Windows environment variables, or
- a `.env` file (if `python-dotenv` is used)

---

## 7. Reliability / Safety Characteristics
### 7.1 Determinism
- Slot filling + FSM ensures the bot asks predictable questions and only executes actions when requirements are met.

### 7.2 Auditability
- Structured results and state transitions allow logging of:
  - what was asked
  - what values were collected
  - what action was executed
  - success/failure and error details

### 7.3 Failure modes
- Missing credentials disables actions gracefully (e.g., Twilio client not available).
- Invalid Twilio SID formats are detected early.
- Network/API failures return structured errors for user-facing messaging and logs.

---

## 8. How to Run (Typical Steps)
> Update commands based on your actual entrypoint (Flask app location).

1. Create and activate venv:
   - PowerShell:
     - `py -m venv .venv`
     - `.\.venv\Scripts\Activate.ps1`

2. Install dependencies:
   - `pip install -r requirements.txt`

3. Configure environment variables (Twilio/Slack/DB/LLM keys)

4. Start the server (example):
   - `py -m flask --app interface.app run --port 5000`

5. Test:
   - POST to `/api/chat` using Postman/curl.

---

## 9. Suggested Next Improvements (Optional)
- Add unit tests for slot extraction and validation.
- Add contract tests for action execution (mock Twilio/Slack).
- Add schema-driven validation errors surfaced to the user (friendly retries).
- Add a “dry-run” mode where actions are logged but not sent.

---

## 10. Appendix: Action Service Example (Twilio)
The Twilio service:
- Normalizes numbers
- Validates SID format
- Creates a Twilio client
- Sends SMS and returns `{success, sid, error}`

This is a good example of the project’s “deterministic execution” principle: no LLM reasoning is involved.

---
End of report.
