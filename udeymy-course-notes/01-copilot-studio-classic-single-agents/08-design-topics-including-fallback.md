## 1. Core Architecture & Concepts

* **Global Variable Scoping & Memory Model:**
* Variables prefixed with `Global.` exist within the global memory space of the conversational session.
* They persist across topic boundaries, meaning any topic within the agent can read, initialize, or mutate a global variable.
* In contrast, un-prefixed variables default to topic scope (`Topic.`), which isolates their state to the active dialog canvas.


* **Slot Filling & Dynamic Topic Jumping (Interruption/Pre-Slot Routing):**
* When a topic references an uninitialized global variable (`Global.employeeName`) within a node (such as a *Send a message* node), the Copilot Studio dialog runtime detects the missing required value.
* **Dynamic Topic Redirection / Slot-Fill Jumps:** The runtime halts execution of the calling topic and traverses to the topic containing the definition/collection node for that variable (e.g., the `Leave allowance` topic where `Global.employeeName` is collected).
* **Return-to-Caller Semantics:** Once the variable is successfully resolved (e.g., the user selects "Susan"), the dialog stack pops back to the original calling topic (`employee name topic`) and resumes execution at the exact node requiring the value. The remaining downstream nodes of the redirected topic are **bypassed/skipped**.


* **Topic Classification Architecture:**
* **Custom Topics (4 Prebuilt Defaults):** Baseline user conversation flows provided out-of-the-box (`Goodbye`, `Greeting`, `Start Over`, `Thank You`). Custom topics can be modified, disabled, or **deleted** entirely by the maker.
* **System Topics (9 Built-in Control Loops):** Core runtime lifecycle handlers governing platform plumbing, session routing, authentication, and error trapping. System topics **cannot be deleted**, but their internal nodes and canvas flows **can be customized/edited**.


* **Variable Data Type Architecture:**
* In addition to primitive data types (`String`, `Number`, `Boolean`), Copilot Studio supports rich and structural types:
* `Boolean`: Strict true/false logic.
* `Record`: Key-value pairs representing a single row or entity (e.g., `{ color: "blue", size: "big" }`).
* `Table`: Structured arrays containing multiple rows and columns.
* `Choice`: Strongly typed option-set enumerations.
* `DateTime`: Standard ISO date/time representations.
* `Blank`: Null/uninitialized state.





---

## 2. Platform Features & Capabilities

* **Custom Topic Authoring & Cross-Topic Resolution:**
* New topic created: `employee name topic`.
* Trigger configured: *User says a phrase* using 5 trigger phrases:
* `employee name`
* `person name`
* `what employee was I referring to`
* `the current person`
* `the people name`


* Action: *Send a message* referencing `{X} Global.employeeName` directly.


* **The 4 Prebuilt Custom Topics:**
* **Greeting:** Triggers on conversational greetings (`Hello`, `Hi`); defaults to *"Hello, how can I help you today?"*.
* **Goodbye:** Handles user departures (`Bye`, `See you`).
* **Thank You:** Handles acknowledgments and gratitude.
* **Start Over:** Handles user requests to reset the conversational context.


* **The 9 Built-in System Topics & Execution Triggers:**
* **Conversation Start:**
* Automatically fires on initial session initialization before user input.
* In **Microsoft Teams**, this topic executes **exactly once** when the user installs/adds the agent to Teams.


* **Conversational Boosting:**
* Automatically triggered when an incoming utterance cannot be mapped to any custom topic intent and generative fallback is invoked.
* Contains the **Create generative answers** node (accessible via *+ Add node* $\rightarrow$ *Advanced* $\rightarrow$ *Generative answers*), querying configured knowledge sources.


* **End of Conversation:**
* Explicitly triggered when an *End conversation* node is reached in a flow.
* Initiates a two-step resolution assessment:
1. Presents a resolution confirmation question: *"Did that answer your question?"* (maps to a `Boolean` variable).
2. If resolved, triggers a CSAT survey: *"Can you rate this conversation out of 5 stars?"*.


* Officially marks the session telemetry state as **Resolved**.


* **Escalate:**
* Triggered by the *Transfer conversation* node or phrases like `talk to an agent` or `call a person`.
* Designed to route the interaction context, transcript, and attached metadata to live service desk platforms (e.g., **Dynamics 365 Omnichannel for Customer Service**). Marks the session telemetry as **Escalated**.


* **Fallback:**
* Triggered when Natural Language Understanding (NLU) fails to match user input to any active topic.
* **Looping Boundary:** Default logic tracks failure count; if triggered **3 consecutive times**, it automatically redirects to the **Escalate** system topic.
* Can be re-architected to query external knowledge engines (Azure AI Search, Azure AI Language Question Answering) or LLMs (Azure OpenAI Service) to bypass the agent's 1,000-topic design cap.


* **Multiple Topics Matched ("Did You Mean?"):**
* Triggered when NLU calculates near-identical confidence scores across two or more topics that have *Include in multiple topic matches* enabled.
* Surfaces a disambiguation list allowing the user to select the correct intended topic.


* **On Error:**
* Triggered during runtime execution exceptions (distinguishing user-driven execution errors from raw infrastructure system downtime).
* Surfaces critical diagnostic metadata in its default payload: **Error Code**, **Conversation ID**, and **Error Timestamp**.


* **Reset Conversation:**
* Triggered by programmatic redirection.
* Clears all topic and global variable states in memory, reloads the latest published version of the agent, and starts a fresh conversation loop.


* **Sign In:**
* Triggered when the agent's authentication mode is set to require user login, or when the execution thread encounters an action/node requiring an authenticated user identity token.





---

## 3. Formulas, Expressions & Schemas

### YAML Definition: Cross-Topic Global Variable Consumer

The declarative YAML representation of the newly created `employee name topic`:

```yaml
kind: AdaptiveDialog
beginDialog:
  kind: OnRecognizedIntent
  id: main
  intent:
    displayName: employee name topic
    triggerQueries:
      - employee name
      - person name
      - what employee was I referring to
      - the current person
      - the people name

  actions:
    - kind: SendMessage
      id: sendMsg_employeeName
      message: The current employee is {Global.employeeName}.

```

### System Topic: On Error Diagnostics Schema

When an unhandled execution error triggers the `On Error` system topic, it parses and outputs the following system-level diagnostic variables:

```powerfx
// Default runtime diagnostic message rendered in On Error
Concatenate(
    "An error has occurred.", Char(10),
    "Error code: ", Text(System.Conversation.ErrorCode), Char(10),
    "Conversation ID: ", Text(System.Conversation.Id), Char(10),
    "Time (UTC): ", Text(System.Conversation.ErrorTimestamp, "yyyy-MM-ddTHH:mm:ssZ")
)

```

### Record & Table Complex Data Structure Schemas

Examples of structured Data Types supported within variable management nodes:

```json
{
  "RecordExample": {
    "color": "blue",
    "size": "big",
    "department": "Human Resources"
  },
  "TableExample": [
    { "EmployeeId": 101, "Name": "Maria", "LeaveBalance": 35 },
    { "EmployeeId": 102, "Name": "Susan", "LeaveBalance": 29 },
    { "EmployeeId": 103, "Name": "Philip", "LeaveBalance": 30 }
  ]
}

```

---

## 4. Best Practices & Design Patterns

* **Global Variable Namespace Management:**
* Because global variables trigger automatic slot-filling redirections when uninitialized, minimize the number of global variables in an agent.
* Use strict, standardized prefixes (e.g., `Global.User_`, `Global.Session_`) to prevent unintended state mutation across disparate authoring teams.


* **Customizing System Topics for Enterprise Architectures:**
* **Fallback Topic Pattern:** Do not leave the Fallback topic as a static dead-end. Route it to Azure AI Language, Azure OpenAI Service, or a Power Automate flow that logs unmatched intent queries to a Dataverse audit table for continuous retraining.
* **Escalate Topic Pattern:** Ensure variables storing customer identifiers, intent, and issue summary are attached as contextual variables in the Omnichannel handoff record prior to calling the *Transfer conversation* node.


* **Session Lifecycle Discipline:**
* Always route successful goal completions through the **End of Conversation** system topic to record explicit CSAT ratings and capture **Resolved** metrics for the analytics dashboard.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Custom vs. System Topic Deletion Rules:**
* **Custom Topics:** Can be customized, deactivated, or permanently deleted.
* **System Topics:** **Cannot be deleted or renamed**. Attempting to delete a system topic is blocked by the platform schema, though makers can modify the visual nodes inside them.


* **Topic Count Platform Hard Limit:**
* Copilot Studio enforces a maximum boundary of **1,000 topics per agent**.
* Organizations requiring broader coverage must offload knowledge retrieval to external systems (Azure AI Search, generative answers via Conversational Boosting, or REST APIs) rather than attempting to author thousands of discrete topics.


* **Microsoft Teams Trigger Constraint:**
* The **Conversation Start** topic fires only **once per user** when the agent is installed into Microsoft Teams, rather than firing at the start of every daily chat session. Testing greeting logic in Teams requires removing and re-adding the agent or using the web chat test canvas.


* **Fallback Escalation Count Threshold:**
* The out-of-the-box Fallback system topic triggers escalation only after **3 consecutive failures**. If an intermittent successful intent matches between failures, the counter resets.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Uninitialized Global Variables:**
* *Trap:* What happens if Topic B reads `Global.VariableA`, but Topic A has not been executed yet?
* *Fact:* The agent does **not** crash or display a null reference error. Copilot Studio **automatically redirects execution to the topic where that variable is collected/initialized**, prompts the user, and immediately returns control to Topic B without running the rest of Topic A.


* **Exam Trap — Topic vs. Global Variable Prefixes:**
* Global variables are explicitly prefixed with `Global.` in formulas and YAML. Topic variables have no prefix or are explicitly scoped as `Topic.`.


* **Exam Trap — Custom vs. System Topics:**
* *Question Scenario:* A developer needs to remove unused default topics from an agent. Which topics can be deleted?
* *Rule:* Only the **4 Custom Topics** (`Greeting`, `Goodbye`, `Start Over`, `Thank You`) can be deleted. The **9 System Topics** (`Conversation Start`, `Conversational Boosting`, `End of Conversation`, `Escalate`, `Fallback`, `Multiple Topics Matched`, `On Error`, `Reset Conversation`, `Sign In`) cannot be deleted.


* **Exam Trap — Session Metrics Classification:**
* Reaching the `End of Conversation` node marks the session as **Resolved**.
* Reaching the `Transfer conversation` / `Escalate` node marks the session as **Escalated**.
* Ending conversations abruptly without routing to either node marks sessions as **Unresolved** or **Abandoned**.


* **Exam Trap — On Error Node Contents:**
* When asked what system parameters are automatically populated by the `On Error` trigger, remember the specific triplet: **Error code**, **Conversation ID**, and **Error timestamp**.