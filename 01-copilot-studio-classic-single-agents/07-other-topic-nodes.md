## 1. Core Architecture & Concepts

* **Interactive Form Architecture via Adaptive Cards:**
* **Adaptive Cards Framework:** An open standard, platform-agnostic declarative JSON schema developed by Microsoft to render structured, interactive UI cards natively across multiple channels (Teams, Web Chat, mobile shells).
* **"Ask with Adaptive Card" vs. "Send a Message (Adaptive Card)":**
* *Send a message (Adaptive Card):* Unidirectional, display-only payload. Renders static UI blocks or simple links; does not pause topic execution to wait for user input or extract returned payload schemas into topic variables.
* *Ask with Adaptive Card:* Bidirectional, blocking conversational node. It halts the conversational execution thread, renders input elements, and waits for a deterministic client-side event before advancing.


* **Submission Contract (`Action.Submit`):**
* An interactive input card *must* contain an execution action (`Action.Submit` or action button inside an `ActionSet`) to pass state back to the Copilot Studio engine.
* The input controls' property IDs directly map to strongly typed variable outputs inside the Copilot Studio canvas.




* **Variable Dynamic Typing & Lifecycle:**
* **Implicit Type Inference:** When creating a variable via the *Set variable value* node, its data type defaults to `Unknown` (untyped). Assigning a literal value (e.g., `0`) immediately casts the variable to a strict type (e.g., `Number`). Assigning text or string literals (e.g., `"none"`) dynamically casts it to `String`.
* **Variable Management Operations:** Enables explicit assignment, value re-casting, manipulation, and centralized auditing across all topic or global state parameters via the variable side pane (*List all variables*).


* **Topic Execution Flow & Control Architecture:**
* **Topic Management Nodes:** Explicit control flow mechanisms that direct how the runtime dialog stack evaluates the lifecycle of conversational threads:
* *Go to another topic (Redirect):* Calls a child topic or branches directly to an external topic, optionally passing input/output variables.
* *End current topic:* Terminates only the active topic canvas; execution returns to the invoking parent topic if a redirection occurred.
* *End all topics:* Force-clears the internal dialog execution stack without terminating the user's active session.
* *End conversation:* Explicitly signals conversational closure. Closes the dialog loop and prevents the orchestration engine or fallback topics from unintentionally matching follow-up queries or lingering context.
* *Transfer conversation / Escalate:* System routing nodes used to hand off the conversation context, variables, and transcripts to a human live agent (e.g., Omnichannel for Customer Service or third-party contact centers).




* **Extensibility & Agent Orchestration Elements:**
* **Tools & Connectors:** Functional building blocks that permit the bot to call external APIs, Power Platform connectors, Power Automate flows, or custom REST actions.
* **Multi-Agent Architecture (Connected Agents):** Enables an agent to invoke or delegate specific domain tasks and intents to specialized secondary/child agents.



---

## 2. Platform Features & Capabilities

* **Adaptive Card Interactive Node ("Ask with Adaptive Card"):**
* **Node Insertion Path:** Canvas node menu (`+`) $\rightarrow$ **Ask with Adaptive Card** (or modifying message payload to an interactive card).
* **Card Visual Authoring:**
* Click **Edit Adaptive Card** to open the integrated visual layout designer.
* **Elements Canvas:** Supports drag-and-drop structural elements (Containers, ColumnSets, FactSets, TextBlocks, ImageSets).
* **Input Controls:** Drag-and-drop input blocks, including `Input.Text`, `Input.Number`, `Input.Date`, `Input.Time`, `Input.Toggle`, and `Input.ChoiceSet`.
* **Control Parameters:**
* *Label:* Descriptive prompt text displayed above the input.
* *Placeholder:* Ghost text shown inside the unfilled input container.
* *Id:* Logical schema identifier that automatically exposes this input as a consumable downstream variable in Copilot Studio.


* **Actions:** `ActionSet` container configured with `Action.Submit`.
* **Responsive Rendering:** Layouts adapt dynamically to client container constraints (mobile screen, sidebar iframe, wide desktop web chat).


* **Card Outputs Exposed in Canvas:**
* `ActionSubmitId`: Built-in output tracking which button triggered the submission (useful when multiple submit buttons exist).
* `Custom Variable Outputs`: Output properties named after the control `Id` values configured in the JSON/designer (e.g., `Topic.numberDays`).




* **Variable Management Node Configuration:**
* **Node Insertion Path:** Canvas node menu (`+`) $\rightarrow$ **Variable management** $\rightarrow$ **Set variable value**.
* **Configuration Controls:**
* *Select a variable:* Allows selecting an existing variable or choosing **Create a new variable**.
* *Variable Properties Pane:* Configures variable display name, data type, scope (*Topic* vs. *Global*), and *Sensitive data* masking.
* *To value:* Target input box accepting static literals (integers, strings, booleans) or dynamic Power Fx expressions.


* **Variable Inspection:** *List all variables* opens a flyout pane auditing all variables active within the current agent scope.


* **Topic Management Navigation:**
* **Node Insertion Path:** Canvas node menu (`+`) $\rightarrow$ **Topic management**.
* Available choices:
* *Go to another topic*
* *End current topic*
* *End all topics*
* *End conversation*
* *Transfer conversation* (Escalate)




* **Agent Integration / Tools:**
* Options exposed directly on node addition:
* *Add a tool:* Basic system tools, Power Platform prebuilt/custom connectors, and API actions.
* *Add an agent:* Multi-agent delegation hooks.
* *Advanced:* Specialized low-level orchestration nodes (HTTP requests, parse JSON, event triggers).





---

## 3. Formulas, Expressions & Schemas

### Adaptive Card JSON Schema (Generated by Visual Designer)

The declarative JSON payload representing the multi-input adaptive card designed in the transcript:

```json
{
  "type": "AdaptiveCard",
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "version": "1.5",
  "body": [
    {
      "type": "TextBlock",
      "text": "How many vacation hours do you want to book?",
      "wrap": true,
      "weight": "Bolder",
      "size": "Medium"
    },
    {
      "type": "Input.Number",
      "id": "numberDays",
      "label": "Please enter it below",
      "placeholder": "number of days",
      "min": 0,
      "max": 365
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "id": "submitVacationRequest",
      "title": "Submit or Send"
    }
  ]
}

```

### Copilot Studio Node YAML Definition (Ask with Adaptive Card)

Under the hood, Copilot Studio binds the Adaptive Card JSON directly to a topic dialog action node and maps the input ID to a topic-level variable:

```yaml
kind: AdaptiveDialog
beginDialog:
  kind: OnRecognizedIntent
  id: main
  actions:
    - kind: AdaptiveCardPrompt
      id: ask_adaptiveCard_vacation
      card:
        type: AdaptiveCard
        version: "1.5"
        body:
          - type: TextBlock
            text: How many vacation hours do you want to book?
          - type: Input.Number
            id: numberDays
            label: Please enter it below
            placeholder: number of days
        actions:
          - type: Action.Submit
            title: Submit or Send
      output:
        binding:
          actionSubmitId: Topic.actionSubmitId
          numberDays: Topic.numberDays

    - kind: SetVariable
      id: set_hours
      variable: Topic.numberOfHours
      value: 0

    - kind: EndConversation
      id: end_conv_node

```

### Power Fx Variable Assignments & Conversions

Equivalent programmatic assignments executed within the *Set variable value* node:

```powerfx
// Number assignment (casts variable to Number)
Set(Topic.numberOfHours, 0)

// String assignment (casts variable to String)
Set(Topic.numberOfHours, "none")

// Converting Adaptive Card string/numeric output for downstream calculation
Set(Topic.TotalCalculatedHours, Value(Topic.numberDays) * 8)

```

---

## 4. Best Practices & Design Patterns

* **Compound Form UX (Adaptive Cards vs. Sequential Questions):**
* When an agent needs to collect 3 or more related inputs (e.g., Start Date, End Date, Reason, Quantity), use a single **Ask with Adaptive Card** node rather than stacking 3 or 4 consecutive *Ask a question* nodes. This minimizes round-trip network latency, prevents user conversation fatigue, and reduces total billable message turns.


* **Input Validation in Cards:**
* Always provide meaningful fallback values, explicit `min`/`max` constraints on `Input.Number`, and `isRequired: true` on critical card fields to enforce client-side validation before the submission payload hits the Copilot Studio runtime.


* **Deterministic Termination via "End Conversation":**
* Place an **End conversation** node at the completion of a self-contained transactional process. Without this explicit boundary, an end-user entering ambiguous follow-up statements risks having the dialog engine misinterpret leftover context or inappropriately bleed into an unmanaged conversational fallback.


* **Explicit Variable Typing:**
* While Copilot Studio dynamically infers variable types based on initial inputs, always initialize variables with a clean literal default (e.g., `""` for strings, `0` for numbers) when authoring enterprise-grade topics to prevent downstream runtime type mismatch exceptions in Power Fx logic.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Action.Submit Requirement:**
* An Adaptive Card used in an **Ask with Adaptive Card** node **will not function** if an `Action.Submit` button is omitted. Without `Action.Submit`, the user cannot complete the action, leaving the dialog permanently paused in a waiting state.


* **Channel Compatibility Constraints:**
* Adaptive Card schemas vary by host client version. High-level features (e.g., advanced card input masking or specific compound layouts) supported in Microsoft Teams or Modern Web Chat may fail or degrade to raw JSON/unsupported messages on limited channels (such as SMS or telephony IVR).


* **Card Output Variable Collision:**
* Ensure every input control inside an Adaptive Card has a unique `id`. Duplicate `id` strings result in overwritten values or schema parsing failures when Copilot Studio binds outputs back to topic variables.


* **End Current Topic vs. End Conversation Bleed:**
* *End current topic* does **not** terminate the overall chat session. If the user invokes an utterance immediately afterward, the agent may attempt to resume contextual evaluation based on previously cached slot memory unless an *End conversation* or state reset was explicitly invoked.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — "Send a Message" Card vs. "Ask with Adaptive Card":**
* *Question Scenario:* The requirements state that the agent must display a form to collect an employee's ID and return date simultaneously.
* *Trap:* Recommending a *Send a message* node with an attached Adaptive Card.
* *Fact:* A *Send a message* node **cannot collect inputs**. You must use **Ask with Adaptive Card** to pause execution and store the input values in variables.


* **Exam Trap — Topic Control Flow Distinctions:**
* **End current topic:** Exits the immediate topic canvas and returns control to the calling topic (if triggered via a *Go to another topic* redirect).
* **End all topics:** Clears the active dialog hierarchy, but the agent remains in the conversation without resetting the session.
* **End conversation:** Formally signals completion of the conversation turn, triggers survey/satisfaction tracking (if enabled), and closes the active transaction context.


* **Exam Trap — Variable Type Casting:**
* A newly created variable in the *Set variable value* node has an `Unknown` type until a value is assigned. Assigning `0` forces the type to `Number`; assigning `"none"` forces the type to `String`. Types cannot be arbitrarily mixed within downstream Power Fx comparisons without explicit conversion functions (`Value()`, `Text()`).


* **Exam Trap — Adaptive Card Schema Binding:**
* The outputs generated by an *Ask with Adaptive Card* node are strictly determined by the `id` attributes assigned to the input elements within the card's JSON schema, alongside the built-in `ActionSubmitId`.