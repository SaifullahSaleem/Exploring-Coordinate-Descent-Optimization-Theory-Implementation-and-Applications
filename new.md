# HR-Project — Comprehensive Technical Report (Dec 22, 2025)

## 1. Executive Summary
This repository implements an **HR task-oriented conversational agent** designed to complete real HR workflows through a controlled multi-turn conversation. The system:
- Accepts a user message (typically through a **Flask API**).
- Detects what HR task the user wants (**intent detection**).
- Collects required information (**slots**) in a guided, step-by-step flow.
- Executes deterministic actions (for example: Slack notifications and optional SMS via **Twilio**).
- Logs conversations and action executions for auditability (often through a database layer such as Supabase or an equivalent storage implementation).

A defining characteristic of this project is the emphasis on **deterministic workflow execution**. The “brain” of the assistant is not free-form text generation; instead it is a **dialog manager** governed by an explicit state machine and schema-driven slot requirements. If an LLM or transformer model is used, it should be used in constrained roles (classification, extraction, or phrasing) and its outputs should be validated and mapped into fixed, known structures.

This report explains what the project is doing, how the main subsystems fit together, which external services are involved, and how model selection is tested and compared using the tooling documented in `tests/README_MODEL_TESTING.md`.

---

## 2. What the Project Does (User-Facing Behavior)
From a user’s perspective, the HR agent behaves like a “workflow assistant” rather than a generic chatbot. It supports HR-related tasks such as:
- **Time-off requests**
- **Meeting scheduling**
- **IT ticket submission**
- **Medical claim filing**

Each workflow has a set of required fields the system must gather (e.g., dates, reasons, participants, urgency, attachments/claim metadata). The bot does not proceed to “execution” until the minimum set of fields is collected and validated. A typical interaction looks like this:

1. The user states a goal in their own words (example: “I need to take leave next week”).
2. The system identifies the intended workflow (example: `request_time_off`).
3. The system extracts any slot values present in the message (example: “next week” → candidate date range).
4. The system asks targeted follow-up questions for missing required slots (example: “What start date?” then “What end date?” then “Reason?”).
5. Once all required slots are filled and validated, the system executes the workflow (example: notify manager/HR through Slack and/or SMS).
6. The user receives a confirmation message, and the system records the action outcome for auditing.

The value of this approach is that the assistant remains reliable even when user language is messy or incomplete. The user does not need to provide everything perfectly in one message; the system is explicitly built to ask for what it needs and to recover from missing/ambiguous inputs.

---

## 3. Core Design Principle: Determinism First
The project’s architecture is intentionally designed to avoid common “chatbot failure modes” such as:
- Taking action based on partial or misunderstood information
- Switching topics mid-workflow
- Producing untraceable decisions (“Why did it send that?”)

To prevent this, the system is built around:
- A **finite state machine (FSM)** (or equivalent controlled state model)
- A **schema-driven slot system** (what fields are required for each intent)
- **Deterministic action execution** modules (Slack/Twilio services that run explicit code)
- Structured logging so every step can be audited (inputs, detected intent, slots, state transitions, and action results)

Even if a transformer model is used, the system should treat model outputs as *candidates* that must be validated and constrained. This is especially important in HR domains where mistakes can cause real operational and privacy issues.

---

## 4. High-Level Architecture (How the Pieces Fit)
### 4.1 Main subsystems
The project can be understood as a pipeline with controlled feedback loops:

- **API/UI Layer**
  - Receives user messages (often JSON over HTTP via Flask).
  - Returns assistant responses.
  - Handles request parsing, CORS (if enabled), and session/conversation IDs.

- **Agent Orchestrator**
  - Coordinates the overall processing of each message.
  - Loads conversation state from storage, calls intent routing, calls the dialogue manager, and finally triggers actions when permitted.

- **Intent Routing**
  - Takes raw text and maps it to a known, allow-listed workflow intent (or falls back to a safe mode such as “general chat” or “unknown intent”).
  - May use a model (NLI / classifier), a rule-based baseline, or a hybrid.

- **Dialogue Manager + FSM**
  - Owns the conversation state.
  - Decides what happens next: ask a question, retry a slot, confirm a normalized value, execute an action, or complete.
  - Prevents action execution when state is incomplete.

- **Slots subsystem (`slots/`)**
  - Encodes the requirements per workflow (schemas).
  - Extracts candidate slot values from user text.
  - Selects the next slot to ask about (slot selection strategy).

- **Actions subsystem (`actions/`)**
  - Executes real side effects deterministically.
  - Example: Twilio SMS notifications using `actions/twilio_service.py`.
  - Returns structured results for logging and user confirmation.

- **Storage / Audit**
  - Saves conversation messages, slot values, FSM state, and action execution results.
  - Enables debugging and compliance/audit trails.

### 4.2 Typical message lifecycle (step-by-step)
1. **Message intake**: user message arrives via the web endpoint.
2. **State load**: system retrieves conversation record and current state (if this is not the first message).
3. **Intent detection**: system determines which workflow the user is trying to complete.
4. **Slot extraction**: system extracts any values present in the user message that correspond to known slots for the detected intent.
5. **Validation / normalization**: system validates extracted values (formats, ranges, known enums). For ambiguous values, it may normalize and ask for confirmation.
6. **Slot selection**: system identifies which required slots are still missing and picks the next best question to ask.
7. **Execution gate**: if all required slots are present and validated, and the FSM indicates it is safe to proceed, the system executes the action.
8. **Logging and response**: results are logged; the system returns a final confirmation or an error message with next steps.

This lifecycle repeats for each user message until the workflow reaches completion.

---

## 5. Repository Structure (What Each Folder Is For)
### 5.1 `slots/` — Slot definitions & slot-filling utilities
The `slots/` package is responsible for converting natural language into structured workflow inputs.

- `slots/schemas.py`
  - Defines workflow schemas: which slots exist per intent and which are required vs optional.
  - Provides slot prompts / question text and expected types.
  - Acts as the “contract” between dialogue control and action execution.

- `slots/slot_extractor.py`
  - Extracts slot values from user input.
  - May use rule-based logic, QA-style models, or hybrid methods.
  - Should be paired with validation logic so that wrong extractions do not lead to wrong actions.

- `slots/slot_selector.py`
  - Chooses the next slot to request from the user.
  - May use rule-based priorities or a model-assisted strategy.
  - Should avoid repeatedly asking the same slot without changing phrasing or providing clarification.

The presence of these files indicates that this project is built as a slot-filling system rather than an open-ended chatbot.

### 5.2 `dialogue/` — Conversation state & deterministic flow
The dialogue layer is responsible for maintaining and progressing the conversation. It is expected to store:
- Current intent
- Collected slot values
- Missing required slots
- Last asked slot
- Retry counts and clarifications
- Whether the action is ready to execute

This is typically implemented via an FSM or a structured state object and a transition function.

### 5.3 `actions/` — Real-world side effects (no model reasoning)
This project uses deterministic action services. One concrete example is Twilio:

- `actions/twilio_service.py`
  - Wraps Twilio SMS sending in a deterministic class.
  - Loads credentials from environment variables or constructor arguments.
  - Normalizes the FROM number and the TO number (adds `+` when needed).
  - Adds sanity checks to catch common misconfiguration (for example, detecting `SK...` secret keys incorrectly placed into Account SID).
  - Returns structured results like:
    - `success` boolean
    - `sid` for message ID
    - `error` string for diagnostics

This kind of module is intentionally “boring” by design: it should be reliable, testable, and predictable, because it directly triggers external side effects.

---

## 6. External Services & Dependencies
### 6.1 Twilio (SMS)
Twilio is used for SMS notifications and requires:
- `TWILIO_ACCOUNT_SID` (should start with `AC...`)
- `TWILIO_AUTH_TOKEN`
- `TWILIO_FROM_NUMBER` (a Twilio-owned phone number)

The code normalizes phone formats and returns helpful errors when misconfigured. This lowers integration friction and makes failures easier to diagnose.

### 6.2 Slack (if configured)
Slack is commonly used to notify HR/IT/management when a workflow completes. This generally requires:
- A Slack bot token
- Target channel IDs
- A message formatting strategy (plain text, blocks, etc.)

### 6.3 Model runtime dependencies
This repository includes explicit tooling for comparing multiple models (see Section 8). This implies the project can use transformer-based models (often from Hugging Face) for tasks like:
- Intent detection
- Slot selection
- Slot extraction (QA-style)

---

## 7. Configuration and Operational Notes
Typical configuration is done through environment variables (Windows user/system environment variables) or a `.env` file if supported. Keep secrets out of Git.

Recommended operational rules:
- Add a “dry-run” mode for actions when testing.
- Do not store sensitive PII unnecessarily in logs.
- Apply strict validation before executing any action.
- Keep an allow-list of intents and action destinations (channels/phone numbers, etc.) where appropriate.

---

## 8. Model Testing and Selection (Based on `tests/README_MODEL_TESTING.md`)
This project includes a dedicated testing and comparison guide in:

- `tests/README_MODEL_TESTING.md`

That file describes a process and scripts for testing and comparing models across three core capabilities:
1. Intent detection
2. Slot selection
3. Slot extraction

### 8.1 How to run model comparisons (as documented in `tests/README_MODEL_TESTING.md`)
From the repository root, you can run:

- Full comparison (recommended):
  - `cd tests`
  - `python compare_all_models.py`
  - Expected runtime: **10–20 minutes** (first run may be slower due to model downloads and caching)

- Individual comparisons:
  - `python test_intent_detection.py --compare`
  - `python test_slot_selection.py --compare`
  - `python test_slot_extraction.py --compare`

The guide also notes that subsequent runs are faster because models are cached locally.

### 8.2 Output artifacts (as documented in `tests/README_MODEL_TESTING.md`)
After running the comparison scripts, the testing system produces JSON files:
- `intent_model_comparison.json`
- `slot_selection_model_comparison.json`
- `slot_extraction_model_comparison.json`
- `final_model_recommendations.json`

These outputs provide the empirical basis for selecting the “best” model for each category.

### 8.3 Scoring logic (as documented in `tests/README_MODEL_TESTING.md`)
The testing guide defines a scoring system:

`Score = Accuracy - (ResponseTime * Weight)`

Interpretation:
- Accuracy is prioritized.
- Response time is penalized, but less than accuracy.
- “Best” is therefore the best tradeoff between correctness and speed.

The guide also provides practical thresholds:
- Good intent model: **Accuracy > 75%**, response time **< 1–2 seconds**
- Good slot model: **Accuracy > 70%**, response time **< 1–2 seconds**
- Acceptable model: **Accuracy > 60%**, response time **< 2–3 seconds**
- Poor model: **Accuracy < 60%**, response time **> 3 seconds**

### 8.4 Models being tested (from `tests/README_MODEL_TESTING.md`)
The following tables list exactly the model families/options documented in the test guide. These tables are included to make the report actionable: they directly map to what the test scripts are intended to compare.

#### Table 1 — Intent Detection Models (from `tests/README_MODEL_TESTING.md`)
| Category | Models being tested (as listed in `tests/README_MODEL_TESTING.md`) |
|---|---|
| Intent Detection | DistilBERT-MNLI; BART-large-MNLI; DistilBART-MNLI; DeBERTa-v3-small; Rule-based |

#### Table 2 — Slot Selection Models (from `tests/README_MODEL_TESTING.md`)
| Category | Models being tested (as listed in `tests/README_MODEL_TESTING.md`) |
|---|---|
| Slot Selection | DistilBERT-base (current default); Xtremedistil; ELECTRA-small; BERT-base; Rule-based |

#### Table 3 — Slot Extraction Models (from `tests/README_MODEL_TESTING.md`)
| Category | Models being tested (as listed in `tests/README_MODEL_TESTING.md`) |
|---|---|
| Slot Extraction | DistilBERT-SQuAD (current default); Xtremedistil; MiniLM-SQuAD2; RoBERTa-base-SQuAD2; Rule-based |

### 8.5 How these results feed back into the main code (from `tests/README_MODEL_TESTING.md`)
After you run the comparisons and review `final_model_recommendations.json`, the guide recommends updating model identifiers in:
- `intent/intent_router.py`
- `slots/slot_selector.py`
- `slots/slot_extractor.py`

It even provides an example of setting a default model name in the intent router initializer. The intent of this workflow is to keep model choices explicit and easy to change, without rewriting core dialogue logic.

### 8.6 Practical recommendation for using the test results
When selecting models using the guide’s output:
- Prefer high accuracy for **intent detection**, because wrong routing causes the entire conversation to go down the wrong workflow.
- Prefer balanced accuracy/speed for **slot selection**, because slot selection happens often (every turn).
- Prefer accuracy for **slot extraction**, but keep strict validation gates. A high-recall extractor that sometimes returns wrong values can still be safe if invalid values are rejected and reprompted.

---

## 9. How to Run the Project (Typical Steps)
Exact entrypoints depend on your configured Flask app, but the standard setup is:

1. Create and activate a virtual environment (PowerShell):
   - `py -m venv .venv`
   - `.\.venv\Scripts\Activate.ps1`

2. Install dependencies:
   - `pip install -r requirements.txt`

3. Configure environment variables:
   - Twilio credentials (if SMS enabled)
   - Slack credentials (if Slack enabled)
   - Storage credentials (if DB enabled)
   - Model/LLM keys (if any hosted providers are used)

4. Run the Flask server (example pattern):
   - `py -m flask --app interface.app run --port 5000`

5. Validate by sending a test request to the chat endpoint (Postman or curl).

---

## 10. Testing Strategy (Beyond Model Comparisons)
This repo already has model comparison tooling (Section 8). In addition, the project should be tested at three levels:

1. **Unit tests**
   - Slot extractor: parsing/normalization tests (dates, phones, IDs)
   - Slot selector: ensures correct “next question” behavior
   - FSM transitions: ensures execution is impossible without required slots

2. **Integration tests**
   - Multi-turn scripted conversations per workflow, including:
     - happy path completion
     - missing values
     - invalid values and retries
     - user corrections mid-flow (“Actually change the date to Friday”)

3. **Action contract tests**
   - Mock Twilio/Slack clients to ensure:
     - correct payload formation
     - correct handling of success/failure responses
     - correct error messages when credentials are missing

The goal is to ensure the system is reliable even before any model tuning.

---

## 11. Reliability, Security, and Compliance Notes
Because this is an HR-oriented assistant, reliability and privacy matter:

- Deterministic gating must be enforced before any side-effect (Slack/SMS).
- Limit what is logged. Avoid storing medical details or sensitive HR data unless necessary and permitted.
- Ensure phone numbers and channels are validated and normalized before sending.
- Prefer least-privilege API keys for Slack/Twilio/DB.
- Provide clear error messages and safe fallback flows when actions fail.

---

## 12. Appendix: Twilio Action Execution (Why it fits this architecture)
`actions/twilio_service.py` is a good representative of the project’s execution philosophy:
- It does not guess what to do.
- It does not interpret user meaning.
- It takes explicit inputs (to/from/message) and performs an explicit operation (send).
- It returns explicit results.

This makes the action layer testable and auditable, which is essential for HR workflows.

---
End of report.
