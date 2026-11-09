## 1. Core Architecture & Concepts

* **Multi-Agent Hierarchical Aggregation (Parent-Child Pattern):**
* A primary orchestrator agent (Agent E – *Leave Balance Aggregator*) coordinates specialized downstream subordinate agents (Agent B – *Employee Leave Allowance Checker* and Agent D – *Employee Leave Tracker*) to execute composite business logic without housing domain data directly.
* **Decoupled Knowledge Architecture:** The aggregator agent operates with zero native knowledge sources (no uploaded spreadsheets, Dataverse tables, or files). Domain knowledge and retrieval mechanics remain strictly encapsulated within the respective child agents.


* **Connected Agents (Sub-Agents / Child Agents):**
* Child agents are exposed as callable capabilities to the parent agent via the **Agents** configuration interface.
* Execution can be invoked individually or concurrently: the generative orchestrator resolves whether a query requires single-agent invocation (e.g., retrieving only the allowance from Agent B or only the history from Agent D) or joint multi-agent coordination (retrieving from both B and D to perform arithmetic subtraction).


* **State & Conversation History Passing:**
* **Context Propagation:** The parent agent can share its full session conversation history with connected child agents via the `Pass conversation history to this agent` configuration toggle.
* **Data Boundary & Privacy Isolation:** Disabling conversation history prevents upstream/downstream context leakage, ensuring that Personally Identifiable Information (PII) or sensitive employee attributes discussed in the main thread do not cross security or authorization perimeters into child agent logs or processing pipelines.
* **Explicit Parameter Mapping (Fallback):** When conversation history passing is disabled, communication across the agent boundary must be strictly handled through defined typed input parameters (e.g., `employeeName: String`) and structured output parameters.


* **Dynamic Intent Delegation & Generative Orchestration:**
* The orchestrator dynamically routes sub-queries at runtime without static topic branching logic, evaluating natural language intent against the connected agents' names, descriptions, and dynamic slash-reference (`/`) bindings defined in the system prompt.



---

## 2. Platform Features & Capabilities

* **Publishing Dependencies & Pre-Requisites:**
* A child agent **must be published** before it can be discovered, bound, and successfully executed by a parent orchestrator in production or integration test sessions. If a child agent has pending edits or has never been published, execution against stale snapshots or complete runtime invocation failures will occur.


* **Connecting & Configuring Sub-Agents UI Path:**
* **Navigation:** Parent Agent Canvas $\rightarrow$ **Overview** tab $\rightarrow$ scroll to **Agents** card $\rightarrow$ click **Add agent** (`+`).
* **Agent Selection & Metadata Scrubbing:** Select target agent from the tenant/environment catalog.
* **Description Refinement:** The agent description must accurately define the agent’s functional capability (e.g., changing *"provides employees with their remaining vacation and sick leave allowances based on uploaded spreadsheet"* to *"provides vacation and sick leave allowances"*). System-level implementation details (such as the underlying storage mechanism like Excel/SharePoint) should be omitted to prevent orchestration hallucinations.
* **Toggle Switch:** `Pass conversation history to this agent` (Checked by default; uncheck to enforce strict parameterization and prevent leakage of sensitive multi-turn context).
* **Confirmation:** Click **Add and configure**.


* **Instruction Prompt Engineering with Slash Commands (`/`):**
* **Dynamic Entity/Agent Injection:** In the **Instructions** editor under the **Overview** tab, typing `/` triggers an autocomplete picker to bind specific connected agents directly into the prompt (e.g., `/Employee Leave Allowance Checker` and `/Employee Leave Tracker`).
* **Syntactic Auto-Suggestions:** The `/` selector lists tools, topics, and connected agents. Arrow keys (`Up`/`Down`) navigate candidates, and `Tab` or `Enter` inserts the tokenized agent reference.


* **Test Canvas Tracing & Debugging:**
* The side test canvas provides live telemetry indicating which connected agents are accessed (`Opening Agent B...`, `Opening Agent D...`).
* Allows single-variable intent verification (e.g., testing only allowance retrieval, testing only taken-leave retrieval) before verifying joint synthesis (calculating net remaining balance).


* **In-Canvas Feedback Loop:**
* Utilizing the inline **Thumbs Up / Thumbs Down (Like/Dislike)** controls on test chat bubbles to capture telemetry and refine prompt boundaries for edge cases (e.g., missing records or unindexed entities).



---

## 3. Formulas, Expressions & Schemas

### Generative Orchestrator System Instructions (Production Formulation)

When configuring the Parent Orchestrator (Agent E), instructions must define retrieval order, role bindings via slash tokens, fallback boundaries for null records, and arithmetic synthesis:

```text
You are the Leave Balance Aggregator. Your role is to determine an employee's remaining leave balance across vacation and sick leave categories by coordinating /Employee Leave Allowance Checker and /Employee Leave Tracker.

Workflow:
1. Identify the target employee name from the user's prompt.
2. Retrieve the total annual leave entitlement for that employee by invoking /Employee Leave Allowance Checker.
3. Retrieve the leave history for that employee by invoking /Employee Leave Tracker.
4. Calculate the net balance:
   - Remaining Vacation Days = (Vacation Allowance from /Employee Leave Allowance Checker) - (Vacation Days Taken from /Employee Leave Tracker).
   - Remaining Sick Leave Days = (Sick Leave Allowance from /Employee Leave Allowance Checker) - (Sick Leave Days Taken from /Employee Leave Tracker).

Null Handling & Edge Case Logic:
- If /Employee Leave Tracker returns no entries, no records found, or indicates the employee is absent from the leave records:
  - Do NOT raise an error.
  - Assume Vacation Days Taken = 0 and Sick Days Taken = 0.
  - The remaining vacation and sick leave balance is equal to 100% of the total allowance retrieved from /Employee Leave Allowance Checker, as the employee has taken zero leave for the current year.

Output Formatting:
- Present a structured summary listing:
  * Total Entitlement (Vacation and Sick Leave)
  * Total Days Taken (Vacation and Sick Leave)
  * Net Remaining Balance (Vacation and Sick Leave)

```

### Deterministic Power Fx Alternative (For Custom Topic / Action Nodes)

If the aggregation logic is handled via a deterministic topic rather than pure generative orchestration:

```powerfx
// Step 1: Initialize values and handle null/empty responses from Agent D (Tracker)
Set(Topic.VacationAllowance, Value(Topic.AgentB_Output.vacationAllowance));
Set(Topic.SickAllowance, Value(Topic.AgentB_Output.sickAllowance));

// Step 2: Coalesce absent tracker records to 0
Set(Topic.VacationTaken, 
    If(
        IsBlank(Topic.AgentD_Output.vacationTaken) || Topic.AgentD_Output.isRecordNotFound, 
        0, 
        Value(Topic.AgentD_Output.vacationTaken)
    )
);

Set(Topic.SickTaken, 
    If(
        IsBlank(Topic.AgentD_Output.sickTaken) || Topic.AgentD_Output.isRecordNotFound, 
        0, 
        Value(Topic.AgentD_Output.sickTaken)
    )
);

// Step 3: Compute net balances
Set(Topic.RemainingVacation, Topic.VacationAllowance - Topic.VacationTaken);
Set(Topic.RemainingSick, Topic.SickAllowance - Topic.SickTaken);

// Step 4: Construct structured response payload
Set(Topic.FinalMessage, 
    $"Summary for {Topic.EmployeeName}: " & Char(10) &
    $"• Vacation Leave: {Topic.RemainingVacation} days remaining (Allowance: {Topic.VacationAllowance}, Taken: {Topic.VacationTaken})" & Char(10) &
    $"• Sick Leave: {Topic.RemainingSick} days remaining (Allowance: {Topic.SickAllowance}, Taken: {Topic.SickTaken})"
);

```

### Input/Output Parameter Schema for Explicit Connected Agent Contracts

When `Pass conversation history to this agent` is deselected, parameters must adhere to strongly typed contracts:

#### Agent D Input Contract (`cr_leaveTrackerInputSchema.json`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "LeaveTrackerInput",
  "type": "object",
  "properties": {
    "employeeName": {
      "type": "string",
      "description": "Full name of the employee whose leave history is being queried."
    }
  },
  "required": ["employeeName"]
}

```

#### Agent D Output Contract (`cr_leaveTrackerOutputSchema.json`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "LeaveTrackerOutput",
  "type": "object",
  "properties": {
    "recordsFound": {
      "type": "boolean",
      "description": "Indicates whether leave records were located for the employee."
    },
    "vacationTaken": {
      "type": "integer",
      "default": 0,
      "description": "Total number of vacation days taken during the current year."
    },
    "sickTaken": {
      "type": "integer",
      "default": 0,
      "description": "Total number of sick days taken during the current year."
    }
  },
  "required": ["recordsFound", "vacationTaken", "sickTaken"]
}

```

---

## 4. Best Practices & Design Patterns

* **The Principle of Least Knowledge for Sub-Agents:**
* Do not pollute child agent descriptions with implementation details (e.g., "based on Excel", "retrieved from SQL table"). The orchestrator only requires functional intent: *what capability it provides* and *what outputs it produces*.


* **Data Boundary & Privacy Governance (DLP & Cross-Agent Leakage):**
* Always audit the `Pass conversation history to this agent` setting. For corporate agents interacting with HR, payroll, or compliance systems, deselect history passing. Pass only strictly validated input variables (e.g., `Employee ID`, `System Email`) to ensure that extraneous conversational text (e.g., medical reasons discussed during sick leave queries) is not passed to downstream agents.


* **Explicit Zero-State / Null-Value Handling:**
* Large Language Models by default interpret "no records found" as an unhandled runtime error or missing knowledge failure. Instructions must explicitly define the semantic meaning of an empty query result: **"No record = 0 units consumed"**.


* **Sub-Agent Error Suppression:**
* Configure child agents that query transactional stores to return safe fallbacks (e.g., `{ recordsFound: false, count: 0 }`) rather than surfacing technical error strings (e.g., "Employee Maria not found in worksheet") that derail parent synthesis.


* **Iterative Prompt Testing Cycle:**
* Validate multi-agent architectures using a three-tier test methodology:
1. *Atomic Test 1:* Query exclusively for Agent B's domain (verify single routing).
2. *Atomic Test 2:* Query exclusively for Agent D's domain (verify alternate single routing).
3. *Composite Integration Test:* Query requiring arithmetic reconciliation across both agents simultaneously.





---

## 5. Technical Limitations, Gotchas & Guardrails

* **Child Agent Unindexed/Unpublished State:**
* If a sub-agent is edited but not published, the parent orchestrator continues to bind to the last published snapshot. If the sub-agent has never been published, execution will fail silently or return a generic system fallback error (`ActionFailed` / `DependencyFailure`).


* **Hallucinatory Aggregation (Summation vs. Subtraction):**
* Generative orchestrators frequently default to summarizing or adding all retrieved numbers across connected agents. Explicit arithmetic rules must be documented in instructions (stating clearly that **Allowance minus Taken = Balance**, not the sum of both).


* **Token Consumption Overhead:**
* Enabling `Pass conversation history to this agent` across multiple sub-agents causes exponential token consumption: each sub-agent call includes the parent agent's conversation history, rapidly increasing latency and approaching topic/model context limits.


* **Circular Routing & Autocomplete Collision:**
* When using slash commands (`/`), verify the exact agent token selected. Because tools, standard actions, and connected agents share the autocomplete menu (e.g., typing `/D` displays both `Agent D` and default canvas actions starting with `D`), incorrect bindings lead to broken dependency trees.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Sub-Agent Publishing Order:**
* *Question Pattern:* "You add an existing agent as a connected agent to an orchestrator, but the orchestrator cannot execute actions against it during testing."
* *Root Cause:* The child agent was modified or created but **never published**. In Copilot Studio multi-agent setups, child agents must be actively published for their endpoints to be accessible by parent orchestrators.


* **Exam Trap — Conversation History Toggle Security:**
* *Question Pattern:* "You need to prevent an HR sub-agent from receiving confidential user inputs submitted to a front-end triage agent while still allowing it to look up records."
* *Action:* Uncheck **Pass conversation history to this agent** in the connected agent configuration panel, and map parameters explicitly via typed Inputs/Outputs.


* **Knowledge Placement Trap:**
* *Scenario:* A question asks where to upload an employee spreadsheet when building an aggregator agent that combines allowance and usage.
* *Correct Architecture:* Upload knowledge sources directly to the specialized child agents (*Allowance Checker* and *Leave Tracker*). The aggregator agent **must not** contain redundant knowledge bases; it functions strictly as a coordination and synthesis layer.


* **Generative Orchestrator Slash (`/`) Syntax:**
* In Copilot Studio generative prompts, typing a forward slash (`/`) is the platform-native mechanism to explicitly reference and bind tools, flows, topics, and connected agents into instruction steps.