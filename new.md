# HR-Project — Comprehensive Technical Report (Dec 22, 2025)

## 1. Executive Summary
This repository implements an **HR task-oriented conversational agent**. The system:
- Accepts a user message (typically via a Flask API).
- Detects what HR task the user wants (intent).
- Collects required information (slots) in a guided, step-by-step flow.
- Executes deterministic actions (e.g., Slack notifications, optional SMS via Twilio).
- Logs conversations and action executions for auditability (commonly via Supabase or a storage layer).

The architecture emphasizes **deterministic workflow execution**: the “brain” is a controlled dialog manager / state machine, while any LLM usage (if enabled) is used mainly for **natural language phrasing** (e.g., rewriting questions, composing messages), not for deciding business logic.

### 1.1 Key idea (in one sentence)
**Conversation = State machine + schema-driven slot filling + deterministic actions; models only assist with language.**

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

### 2.1 What “slots” mean in this project
A **slot** is a structured field required to complete a workflow, e.g.:
- `start_date`, `end_date`, `reason` for time off
- `meeting_title`, `participants`, `date_time` for meetings
- `issue_type`, `device`, `urgency`, `description` for IT tickets

The bot must collect all required slots (and validate them) **before** executing any real-world action.

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

### 3.3 Sequence diagram (conceptual)
```text
User -> API(Flask) -> Agent -> IntentRouter -> DialogueManager(FSM)
                                           -> SlotExtractor/Selector
                                           -> Actions(Slack/Twilio)
                                           -> Storage(Audit/State)
Agent -> API(Flask) -> User
```

### 3.4 Deterministic control points
The system should only perform side-effects when:
- Intent is known and allowed (allow-list)
- Required slots are present
- Slot values pass validation / normalization
- FSM state indicates `READY_TO_EXECUTE`

---

## 4. Repository Structure (What Each Folder Is For)
> This section describes the intended responsibilities. Your exact modules may vary, but the roles should map closely.

### 4.1 `slots/` — Slot definitions & slot-filling utilities
Files:
- `slots/schemas.py`
  - Defines the *data schema* for each intent: required slots, optional slots, types, validation rules, default prompts, and canonical names.
- `slots/slot_extractor.py`
  - Extracts slot values from user input (e.g., dates, times, names, departments, phone numbers).
  - Often includes parsing helpers (regex, date parsing, lookup tables).
- `slots/slot_selector.py`
  - Chooses the “next best question” (which slot to ask next) based on:
    - which slots are missing
    - what was last asked (avoid repeats)
    - retry policies (max retries, clarify questions)
- `slots/__init__.py`
  - Module exports / packaging.

**Purpose**: Make the conversation deterministic by using known slot requirements per intent.

### 4.2 `dialogue/` — Conversation state & deterministic flow
Common responsibilities:
- Maintains structured state:
  - `intent`
  - collected `slot_values`
  - missing slots
  - last asked slot
  - retries and user confirmations
- Moves through an FSM so the agent doesn’t “wander” or hallucinate flows.
- Produces the next assistant action:
  - Ask a slot question
  - Retry slot (clarify)
  - Execute action
  - Finish

**Typical outputs from the dialogue manager**
- `action = "ask_slot"` + `slot_name`
- `action = "retry_slot"` + guidance
- `action = "execute"` + payload
- `action = "complete"` + summary

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

**What you’re using here**
- Twilio Python SDK (`twilio` package)
- Environment-variable based configuration
- Deterministic error handling for operations

**Why it matters**
- This service is “safe” in the sense that it never relies on an LLM to decide recipients, channels, or whether to send. It only executes explicit calls made by the deterministic controller.

### 4.4 `llm/` (if present) — Natural language assistance (phrasing, normalization)
Typical responsibilities (project-dependent):
- Rewrite slot questions to be more natural (without changing meaning).
- Compose Slack/SMS messages from structured slot values.
- Normalize ambiguous values (e.g., “next Monday”) into canonical forms, then request confirmation.

**Important**
- LLM output should not directly decide business logic—only text phrasing.
- LLM outputs should be validated and constrained (e.g., JSON schema, allow-lists).

### 4.5 `interface/` or `api/` — Web server endpoints
Typical responsibilities:
- Flask endpoints such as `POST /api/chat`
- Serves UI assets (if any)
- Handles CORS, JSON parsing, returning responses
- Basic request validation (required fields, conversation id)

### 4.6 Storage (Supabase / DB / logging)
Typical responsibilities:
- Persist `conversations` (session metadata)
- Persist `messages` (user/bot)
- Persist `fsm_states` (state snapshots)
- Persist `action_executions` (audit)

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

### 6.1 Recommended configuration practices
- Keep secrets out of Git (use `.gitignore` for `.env`).
- Use separate credentials for dev vs production.
- Add “dry-run” modes (log actions without sending) for safe testing.

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

### 7.4 Security considerations (recommended)
- Validate and sanitize all user input (especially if used in Slack messages, ticket descriptions).
- Apply allow-lists for intents and action destinations.
- Avoid sending sensitive medical/HR data to third-party services unless explicitly allowed and protected.
- Log carefully: store “just enough” for debugging; avoid storing secrets.

---

## 8. Model Selection — Comparison Tables (Intent, Slots, Generation, Normalization)

### 8.1 Intent Classification (route user message → workflow)
| Option | Examples | Accuracy (task routing) | Latency | Cost | Determinism | Offline/On‑prem | Implementation effort | Recommended when |
|---|---|---:|---:|---:|---:|---:|---:|---|
| Rule-based | keyword/regex, patterns | Low–Med (coverage-based) | Very Low | Very Low | High | Yes | Low | Small set of intents, strict compliance needed |
| Traditional ML classifier | TF‑IDF + Logistic Regression/SVM | Med | Low | Low | High | Yes | Med | You have labeled data; want stable routing |
| Zero-shot NLI | `bart-large-mnli` style | Med | Med | Low–Med | Med | Yes (if local) | Med | Quick baseline without labeled data |
| Small/fast hosted LLM | provider-hosted small LLM | Med–High | Low–Med | Med | Low–Med | No | Low | Better paraphrase handling; acceptable variability |
| Large hosted LLM | frontier LLM | High | Med | High | Low | No | Low | Maximum robustness; add strict guardrails |

**Recommended pattern here**: rules for high-precision triggers + model fallback + allow-list mapping.

### 8.2 Slot Extraction (extract structured fields from text)
| Option | Examples | Extraction quality | Latency | Cost | Determinism | Offline/On‑prem | Implementation effort | Recommended when |
|---|---|---:|---:|---:|---:|---:|---:|---|
| Regex + heuristics | phone/email/date regex, enums | Low–Med | Very Low | Very Low | High | Yes | Med | Slots are simple and formatted |
| Deterministic libraries | `dateparser` etc. | High (dates/times) | Low | Low | High | Yes | Low | Date/time normalization is required |
| Classic NER | spaCy NER/CRF | Med | Low | Low | High | Yes | Med–High | Domain entity extraction with training |
| LLM JSON extraction | prompt outputs JSON | High | Med | Med–High | Low | No | Low | Messy text; then validate strictly |
| Hybrid | rules + libs + LLM fallback | High | Low–Med | Low–Med | Med–High | Partial | Med | Robust extraction while preserving control |

### 8.3 Response Generation (phrasing, question rewriting, message composition)
| Option | Examples | Text quality | Latency | Cost | Determinism | Safety/Control | Recommended when |
|---|---|---:|---:|---:|---:|---:|---|
| Template-based | f-strings/Jinja | Med | Very Low | Very Low | High | High | Compliance-heavy orgs; predictable responses |
| Small LLM | fast hosted LLM | Med–High | Low–Med | Med | Low–Med | Med | Better UX text, keep logic deterministic |
| Large LLM | frontier | High | Med | High | Low | Med (guardrails needed) | Best naturalness; requires controls |

### 8.4 Normalization & Validation
| Option | Examples | Reliability | Determinism | Effort | Recommended use |
|---|---:|---:|---:|---:|---|
| Strict validators | enums/regex/ranges | High | High | Med | Final gate before action execution |
| Deterministic parsers | date/time libraries | High | High | Low | Normalize dates/times without LLM |
| LLM normalization | “interpret next Monday” | Med–High | Low | Low | Fallback + require user confirmation |

### 8.5 Recommended “Safe Defaults”
| Layer | Default | Reason |
|---|---|---|
| Intent | rules + fallback model + allow-list | Robust but controlled |
| Slots | deterministic extractors + schema validation | Prevents wrong executions |
| Dialogue | FSM-driven | Testable and predictable |
| Actions | deterministic services | Auditable, safe side-effects |
| Logs | structured audit records | Debuggability and compliance |

---

## 9. How to Run (Typical Steps)
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

## 10. Testing Strategy (Recommended)
### 10.1 Unit tests
- Slot extraction:
  - “Given message X, extracted slots should equal Y”
- Slot selector:
  - “Given missing slots, next asked slot should be Z”
- FSM transitions:
  - “Cannot execute unless all required slots validated”
- Action services:
  - Mock Twilio/Slack clients and assert payload formation

### 10.2 Integration tests
- Conversation scripts:
  - Multi-turn test conversations that end in successful execution
  - Failure/retry paths (invalid phone, missing date, etc.)

---

## 11. Suggested Next Improvements (Optional)
- Add strict JSON-schema validation for any model outputs (if used).
- Add idempotency keys for actions (avoid sending duplicate SMS/Slack on retries).
- Add role-based routing (employee vs manager vs HR).
- Add PII redaction in logs and “privacy modes”.

---

## 12. Appendix: Action Service Example (Twilio)
The Twilio service:
- Normalizes numbers
- Validates SID format
- Creates a Twilio client
- Sends SMS and returns `{success, sid, error}`

This is a good example of the project’s “deterministic execution” principle: no LLM reasoning is involved.

---
End of report.
