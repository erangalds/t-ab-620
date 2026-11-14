## 1. Core Architecture & Concepts

* **Prompt Tool Integration in Agent Orchestration:**
* *Prompt Action Encapsulation:* When a custom prompt is added to an agent's tool catalog, it functions as a modular runtime capability. The generative orchestrator treats the prompt tool as an executable node capable of accepting dynamic slot inputs (e.g., `EmployeeName`) and emitting deterministic outputs (e.g., `JobRole`).
* *Multi-Model Orchestration Boundary:* The root agent orchestrator can operate on a primary enterprise model, while the prompt tool executes against a dedicated model configured in Prompt Builder (including custom models deployed via Microsoft Foundry/Azure AI Foundry). This compartmentalizes model cost, latency, and context windows.


* **Structured Instruction Triad (Goal, Action, Transition):**
* Autonomous agent system instructions rely on a structured semantic framework for each logical step:
* **Goal:** Explicit objective statement defining what the step accomplishes (e.g., *"Retrieve job role for the employee"*).
* **Action:** The specific operational mechanism or tool binding executed by the orchestrator (e.g., *"Use the [Tool: employee role] to return the job role matching the employee name"*).
* **Transition:** Deterministic boundary directive dictating execution flow upon data retrieval (e.g., *"Once data is retrieved, look up the employee job role in step 4 and then proceed to step 5"*).




* **Instruction Object Referencing (`/` Notation):**
* Typing a forward slash (`/`) inside the instruction authoring canvas opens an inline object picker. Selecting an asset binds an encapsulated reference token (rendered as an object badge/pill) rather than plain text. This links the agent’s reasoning engine directly to the underlying Dataverse tool schema and input/output contracts.


* **Multi-Step Autonomous Orchestration Traces:**
* As tools are integrated into autonomous event loops, the execution lifecycle scales from single-phase execution to multi-phase pipeline execution:

$$\text{Trigger Ingestion} \longrightarrow \text{Tool Invocation (Prompt Action)} \longrightarrow \text{Grounding (Knowledge Retrieval)} \longrightarrow \text{Downstream Action (Email Dispatch)}$$


* The **Activity** tab audits each step independently, capturing individual latency metrics, input bindings, retrieved records, and completion status across each discrete stage.


* **Runtime Environment Decoupling (Draft vs. Published State):**
* *Draft Runtime (Test Harness):* Testing via the developer canvas or the **Test trigger** interface executes against the agent's current, unpublished working definition.
* *Live Runtime (Autonomous Polling/Webhooks):* Incoming external events (e.g., real emails sent to monitored mailboxes) execute strictly against the **Published snapshot** of the agent. Modifications made to tools, instructions, or trigger criteria remain inactive for live events until the maker selects **Publish**.



---

## 2. Platform Features & Capabilities

* **Instruction Authoring Canvas:**
* *Navigation:* Agent Overview $\rightarrow$ **Instructions** $\rightarrow$ Click **Edit**.
* *Step Re-indexing:* When adding intermediate capabilities, existing steps must be explicitly shifted (e.g., renumbering original Step 4 $\rightarrow$ Step 5, and Step 5 $\rightarrow$ Step 6) to preserve deterministic execution order.


* **Configuring Prompt Steps within Instructions:**
* **Goal Specification:** Define targeted outcome.
* **Action Specification:** Invoke the `/` context menu to select the designated prompt tool (`employee role`).
* **Transition Specification:** Define deterministic hand-off rules between the output of the prompt tool and the input of downstream knowledge searches or output dispatchers.
* **Interaction Examples:** In-prompt few-shot framing provided to guide response composition (e.g., *"We received your vacation request. He is an accountant and current allowances..."*).


* **Prompt Tool Settings & Hyperparameters:**
* *Temperature:* Controls generation variance ($0.0$ for deterministic classification/lookups, $1.0$ for creative tasks).
* *Record Retrieval Ceiling:* Limits rows fetched from attached data sources ($30$ to $1,000$ records).
* *Code Interpreter:* Executes sandboxed Python code to perform complex calculations, structured data manipulation, and table updates.
* *Content Moderation:* Configurable safety filtering tiers (`Low`, `Medium`, `High`).


* **Testing Mechanics (Test Trigger vs. Live External Trigger):**
* *Test Trigger:* Initiated via **Triggers** $\rightarrow$ **Test trigger** $\rightarrow$ Select historical execution payload $\rightarrow$ Click **Start testing**. Bypasses external network polling, executing immediately against the draft instruction set.
* *Live Trigger Execution:* Initiated by dispatching an external email matching the filter criteria (e.g., subject line containing `"vacation"`). Operates via the underlying Power Automate asynchronous cloud flow.


* **Activity Console Observability:**
* *Navigation:* Agent Canvas $\rightarrow$ **Activity** tab.
* *Trace Step Decomposition:* Multi-step autonomous runs visually report each stage:
1. Tool Execution: `employee role` prompt tool invocation.
2. Knowledge Search: Retrieval queries against `leave allowance.xlsx`.
3. Connector Invocation: `Send an email (V2)`.





---

## 3. Formulas, Expressions & Schemas

### Agent System Instructions Schema (Multi-Step Tool-Integrated Logic)

The complete system instruction set compiled across the agent's authoring canvas:

```markdown
1. Ingestion:
   - Goal: Parse incoming email triggers.
   - Action: Extract the employee name and requested leave types (vacation, sick leave) from the trigger body.
   - Transition: Once extracted, proceed to step 2.

2. Knowledge Search:
   - Goal: Retrieve leave entitlements.
   - Action: Search the uploaded "leave allowance" spreadsheet for rows matching the extracted employee name.
   - Transition: Once data is retrieved, look up the employee job role in step 4.

3. Intermediate Validation:
   - Goal: Ensure data completeness before proceeding.
   - Action: Verify extracted employee records exist in knowledge sources.
   - Transition: Pass control to Step 4.

4. Lookup Employee Job Role:
   - Goal: Retrieve job role for the employee.
   - Action: Use the [Tool: employee role] to return the job role matching the employee name.
   - Transition: Once data is retrieved, prepare the email response in step 5.

5. Compose Response Email:
   - Goal: Synthesize retrieved data into an outbound notification.
   - Action: Draft the email including the employee name, employee job role, vacation leave allowance, and sick leave allowance.
   - Interaction Example: "We received your vacation request. Jane is a social media manager and current allowances are: Vacation: 29 days, Sick Leave: 4 days."
   - Transition: Proceed to step 6.

6. Dispatch Notification:
   - Goal: Send the formatted details to the user.
   - Action: Use [Tool: Send an email (V2)] to dispatch the message to the target recipient.
   - Transition: Complete execution.

```

### Prompt Input/Output Execution Contract (JSON Schema)

The data payload contract exchanged between the Copilot Studio orchestrator and the `employee role` prompt tool:

```json
{
  "promptAction": "employee_role_tool",
  "requestSchema": {
    "type": "object",
    "properties": {
      "EmployeeName": {
        "type": "string",
        "description": "The target employee name extracted from context"
      }
    },
    "required": ["EmployeeName"]
  },
  "responseSchema": {
    "type": "object",
    "properties": {
      "JobRole": {
        "type": "string",
        "description": "The exact professional title retrieved from the internal directory"
      }
    }
  }
}

```

### Activity Run Telemetry Trace (Multi-Step Execution Payload)

```json
{
  "executionId": "b8f411cd-0922-4a8a-bfe9-1b39e6a992cf",
  "triggerType": "Office365Outlook_WhenNewEmailArrives_V3",
  "status": "Completed",
  "executionDurationMs": 2840,
  "steps": [
    {
      "stepIndex": 1,
      "type": "PromptTool",
      "toolName": "employee role",
      "input": { "EmployeeName": "Maria" },
      "output": { "text": "Secretary" },
      "status": "Succeeded"
    },
    {
      "stepIndex": 2,
      "type": "KnowledgeRetrieval",
      "source": "leave allowance.xlsx",
      "query": "Maria leave allowance vacation sick",
      "status": "Succeeded"
    },
    {
      "stepIndex": 3,
      "type": "ConnectorAction",
      "toolName": "Send an email (V2)",
      "parameters": {
        "to": "curious_test_recipient@curiouselds.com",
        "subject": "Vacation allowance details for Maria",
        "body": "We received your vacation request. Maria is a secretary and current allowances are: Vacation: 30 days, Sick Leave: 5 days."
      },
      "status": "Succeeded"
    }
  ]
}

```

---

## 4. Best Practices & Design Patterns

* **Explicit Step Sequential Coupling (Goal-Action-Transition):**
* Avoid vague conversational instructions (e.g., *"Also get the role and send an email"*).
* Always structure sequential agent instructions using **Goal**, **Action**, and **Transition** triplets to enforce deterministic tool chaining and prevent out-of-order execution.


* **Tokenized Tool Badging:**
* Never type tool names as literal plain text inside instructions. Always invoke the `/` slash menu to insert the native object reference. This guarantees that rename operations or schema refactorings in the tool's underlying Dataverse definition propagate without breaking the prompt's semantic binding.


* **Interaction Examples for Output Conditioning:**
* Provide explicit interaction examples (few-shot exemplars) in the agent instructions. This conditions the generative model on the expected tone, sentence structure, and dynamic token placement before executing output actions.


* **Publishing Hygiene in Autonomous Agent ALM:**
* *Cadence:* Establish a strict build/publish pattern. Any change made to agent instructions, prompt parameters, or tool mappings must be followed by an explicit **Publish** action before running end-to-end integration tests via external triggers.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **The "Unpublished Agent" Runtime Desynchronization Trap:**
* *The Gotcha:* A maker adds a new tool, updates instructions, and verifies that the agent works in the authoring canvas using **Test trigger**. However, when a live email arrives, the agent executes using the *old* logic (omitting the new tool).
* *Root Cause:* The interactive test harness tests the **Draft state**, whereas external event triggers (Power Automate flows) invoke the **Published state**. If an agent is not republished after modifying instructions, live event triggers will execute stale logic.


* **Instruction Indexing Cascades:**
* Inserting an intermediate step (e.g., Step 4) without updating upstream and downstream transitions (e.g., Step 3 pointing to Step 4, Step 4 pointing to Step 5) can cause the LLM to skip steps, jump directly to output generation, or terminate the pipeline prematurely.


* **Asynchronous Telemetry Polling Latency:**
* Live runs do not populate the **Activity** tab instantly. Power Automate polling and asynchronous event queues introduce a delay of **30 to 90 seconds** before run traces appear. Refreshing immediately may display `No activity to display`.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Draft Testing vs. Published Trigger Behavior:**
* *Exam Trap:* A question presents a scenario where an autonomous agent's instructions were updated to call a new prompt tool, and testing via "Test Trigger" succeeds, but real inbound emails continue to trigger the old response without calling the tool.
* *Correct Answer:* The maker failed to **Publish** the agent. Autonomous triggers listening to external connectors execute exclusively against the **last published snapshot**.


* **Prompt Tool Grounding & Capabilities:**
* *Exam Fact:* When configuring settings for a prompt tool, the parameters governing its behavior are:
1. **Temperature** ($0.0$ to $1.0$).
2. **Data source record retrieval limits** ($30$ to $1,000$).
3. **Code Interpreter toggle** (enables Python sandboxed execution for math, tabular files, and Dataverse writes).
4. **Content Moderation sensitivity** (`Low`, `Medium`, `High`).




* **Instruction Syntax for Object Binding:**
* *Exam Fact:* Typing a forward slash (`/`) is the platform-standard syntax used to embed reference objects (tools, knowledge sources, topics, or sub-agents) into Copilot Studio agent instructions.


* **Diagnosing Multi-Tool Execution Failures:**
* *Exam Scenario:* Where do you inspect the step-by-step reasoning trace, tool input values, and execution duration of an autonomous agent run triggered by an external connector?
* *Correct Answer:* The **Activity** tab within the Copilot Studio agent portal.