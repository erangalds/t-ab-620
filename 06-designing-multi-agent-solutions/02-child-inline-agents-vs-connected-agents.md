## 1. Core Architecture & Concepts

* **Single-Agent vs. Multi-Agent Paradigm:**
* *Single-Agent Architecture:* Centralized reasoning engine, single conversational context, and unified system prompt. Offers lower computational overhead, high predictability, simplified authoring, and consolidated debugging. Suffers from scaling limits, context window exhaustion, tight coupling, and a single point of failure.
* *Multi-Agent Architecture:* Distributed orchestrations where specialized micro-agents operate autonomously or semi-autonomously, communicating via message-passing protocols or structured context handoffs.


* **Inline Agents (Child Agents):**
* *Definition:* Specialized functional sub-agents compiled directly within the parent agent's solution boundary.
* *Context & Topic Inheritance:* Child agents share the parent agent's conversational topics. **Child agents cannot have their own independent topics.**
* *Knowledge & Tool Scoping:* Can leverage the parent's tools and knowledge sources, or declare scoped subsets.
* *Lifecycle & Deployment:* Not deployed or published independently. They publish atomically whenever the root parent agent is published.
* *Execution Arbitration (Priority Indexing):* Priority is numerically weighted. Lower numerical values indicate higher operational priority ($1$ takes precedence over $1000$). When intent resolution conflicts between an inline agent and a conversational topic, the lower-numbered priority fires first.


* **Connected Agents:**
* *Definition:* Fully independent, externally published, modular agent instances invoked via agent-to-agent communication interfaces.
* *Autonomy & Encapsulation:* Maintain completely independent system instructions, topics, trigger configurations, knowledge bases, connection references, and role-based access control (RBAC).
* *Execution Topology (No Multi-Tier Chaining):* Connected agent topologies are strictly **hub-and-spoke (single-tier depth)**. A parent agent can connect to multiple target connected agents; however, a connected child agent **cannot connect to another downstream agent** ($\text{Agent } A \rightarrow \text{Agent } B$ is valid; $\text{Agent } A \rightarrow \text{Agent } B \rightarrow \text{Agent } C$ is blocked).


* **External Connected Agent Endpoints:**
* Copilot Studio supports binding to external agents built across the Microsoft AI ecosystem:
* **Microsoft Fabric AI Agents**
* **Microsoft Foundry / Azure AI Foundry Agents**
* **Microsoft 365 Agents SDK** implementations
* Standard **Agent-to-Agent (A2A)** interoperability protocols.




* **Conversation History Telemetry & Boundary Governance:**
* Both inline and connected agents support toggling whether the accumulated conversational session history is marshaled across the invocation boundary.
* *Governance Rule:* Disabling history forwarding prevents cross-boundary data leakage (e.g., exposing PII or confidential parent session variables) and mitigates token context window bloat, reducing invocation latency.



---

## 2. Platform Features & Capabilities

### Inline (Child) Agent Configuration

* **Navigation:** Overview tab $\rightarrow$ **Agents** section $\rightarrow$ Select **+ Add an agent** $\rightarrow$ Choose **Inline Agent**.
* **Triggers:** Default trigger is **Agent Chooser** (generative orchestrator delegates execution based on the parent's dynamic intent routing).
* **Runtime Priority:** Configured via a positive integer field. Determines conflict arbitration when an inline agent and a topic share matching trigger conditions.
* **Input Management & Extraction:**
* *Data Types:* Text, Number, Boolean, Choice, Custom Entities.
* *Advanced Settings:*
* `Should prompt user`: Dictates whether the agent initiates an interactive slot-filling dialogue if an input parameter is missing.
* `Condition validation`: Evaluates conditional criteria on extracted values.
* `Condition not met prompt`: Custom error/clarification dialogue presented when validation fails.
* `Action if no entity is found`: Configures fallback logic if extraction slots remain unresolved.
* `Reprompt limits`: Restricts retry attempts to a maximum of **up to 2 times**.
* `Custom retry prompt`: Tailored prompt delivered to the user upon input extraction failure.




* **Operational State:** Individual inline agents can be toggled **Enabled** or **Disabled** without deleting their underlying configuration.

### Connected Agent Configuration & Handshake

* **Prerequisites for Connection:**
1. The target connected agent must be **Published**.
2. The target connected agent must explicitly grant inbound agent-to-agent permissions via:
* **Settings** $\rightarrow$ **Generative AI** $\rightarrow$ **Connected agents** section $\rightarrow$ Toggle **"Let other agents connect to and use this one"** to **On**.




* **Data Contract Definition:**
* The parent agent's configuration must declare explicit input schemas (payload passed to the target agent) and output schemas (context marshaled back upon completion).


* **Lifecycle Management (ALM) Independence:**
* Connected agents can reside in different Power Platform environments, belong to separate ALM pipelines/solutions, and be updated or maintained by separate specialized development teams.



---

## 3. Formulas, Expressions & Schemas

### Child Agent Parameter Validation & Slot Evaluation (Power Fx)

When validating dynamic inputs passed into a child agent or evaluating missing slot reprompts:

```powerfx
// Evaluates if required input 'EmployeeID' was provided; triggers reprompt if invalid
If(
    IsBlank(Topic.EmployeeID) || !IsMatch(Text(Topic.EmployeeID), "^\d{6}$"),
    Set(Topic.IsValidInput, false);
    Set(Topic.ValidationErrorMessage, "Please provide a valid 6-digit employee identification number."),
    Set(Topic.IsValidInput, true)
);

// Priority evaluation expression simulated across competing handlers
If(
    Topic.InlineAgentPriority < Topic.CompetingTopicPriority,
    Invoke(InlineAgent.LeaveProcessor),
    Navigate(Topics.ManualLeaveHandling)
)

```

### Agent-to-Agent Ingestion Schema (JSON Contract)

When calling a connected agent or delegating execution to an external Microsoft 365 / Azure Foundry endpoint:

```json
{
  "$schema": "https://copilotstudio.microsoft.com/schemas/v1/agent-to-agent-call.json",
  "caller": {
    "agentId": "a1b2c3d4-e5f6-7890-abcd-112233445566",
    "conversationId": "conv-9988-7766-5544",
    "forwardConversationHistory": false
  },
  "target": {
    "agentId": "f7e6d5c4-b3a2-1098-dcba-665544332211",
    "endpointType": "ConnectedAgent"
  },
  "inputParameters": {
    "EmployeeName": "Philip",
    "LeaveType": "Vacation",
    "TargetYear": 2029
  },
  "executionPolicy": {
    "timeoutMs": 15000,
    "fallbackOnFailure": true
  }
}

```

### Connected Agent Generative AI Setting (YAML Snippet)

The metadata configuration representing the inbound connection allowance on the target agent:

```yaml
generativeAiSettings:
  allowExternalConnections: true
  shareConversationHistory: false
  allowedCallers:
    tenantId: "8f8b3c2d-9a1e-4c7b-b3d2-8e1a5f6c4b2a"
    enforceAuthentication: true

```

---

## 4. Best Practices & Design Patterns

* **Architecture Selection Decision Matrix:**
| Architectural Vector | Inline (Child) Agent | Connected Agent |
| --- | --- | --- |
| **Scope & Complexity** | Focused task, tight integration | Complex, multi-domain capabilities |
| **Topic Architecture** | Inherits parent's topics (cannot own topics) | Maintains dedicated topic catalog |
| **ALM & Lifecycle** | Single solution, single deployment cycle | Independent solutions, disparate release cadences |
| **Team Ownership** | Single developer or tightly integrated team | Disparate teams, business units, or external vendors |
| **Reusability** | Scoped strictly to the host parent agent | Multi-agent reusable hub-and-spoke component |
| **Invocation Latency** | Low (compiled within the parent engine) | Higher (cross-boundary network serialization) |


* **Security Boundary & Confused Deputy Mitigation:**
* Maintain strict privilege symmetry across connected boundaries. If the parent agent is restricted from destructive actions (e.g., executing file deletes or modifying Dataverse security roles), **do not delegate execution to a connected agent that possesses those capabilities** without implementing explicit user authentication challenges and human-in-the-loop (HITL) authorization gates.


* **Start Simple (Refactoring Progression):**
* Begin implementation using **Inline Agents** to organize sub-tasks into modular logical units without incurring distributed system overhead. Transition to **Connected Agents** only when cross-agent reusability, independent knowledge governance, or separated ALM pipelines become hard requirements.


* **Minimizing Context Bloat:**
* Always set `forwardConversationHistory` to **False** unless the child/connected agent strictly relies on prior conversational turns for contextual disambiguation. Suppressing history reduces LLM token consumption, prevents cross-boundary data leakage, and improves response latency.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Chaining Depth Ceiling (No Nested Multi-Tier Connections):**
* Copilot Studio enforces a strict **single-level delegation limit**. An agent may connect to one or more connected agents ($\text{Agent } A \rightarrow \text{Agent } B$, $\text{Agent } A \rightarrow \text{Agent } C$). However, a connected agent **cannot invoke another connected agent** ($\text{Agent } A \rightarrow \text{Agent } B \rightarrow \text{Agent } C$ results in an execution fault).


* **No Dedicated Topics on Inline Agents:**
* *Gotcha:* Attempting to author independent topics directly within an inline agent is impossible. Inline agents rely exclusively on their assigned instructions, tools, knowledge sources, and the parent agent's global topic hierarchy.


* **The Unshared Connected Agent Flag:**
* *Gotcha:* If a target connected agent is fully published but still fails to appear or throws an invocation authorization error during execution, verify the generative AI settings. The toggle **"Let other agents connect to and use this one"** must be explicitly switched to **On**.


* **Reprompt Hard Boundary:**
* Slot-filling validation rules on child agent input parameters enforce a hard platform limit of **up to 2 reprompts**. If the input remains unvalidated or missing after two attempts, execution routes directly to the designated fallback action.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Publishing Dependencies for Connected vs. Inline:**
* *Exam Trap:* Believing both inline and connected agents require separate publishing steps before testing.
* *Fact:* **Inline agents publish automatically with the parent agent.** **Connected agents MUST be published independently** and have the connection toggle enabled before they can be consumed by external parent agents.


* **Priority Arbitration Logic:**
* *Exam Fact:* When an inline agent and a conversational topic compete for the same user intent, resolution is determined by priority: **lower numerical values execute first** (Priority $1$ executes ahead of Priority $10$).


* **Architectural Chaining Boundaries:**
* *Exam Scenario:* An architect attempts to link Customer Copilot (Parent) $\rightarrow$ Order Copilot (Connected) $\rightarrow$ Logistics Copilot (Connected).
* *Exam Fact:* This configuration **fails**. Copilot Studio does not permit multi-tier daisy-chaining of connected agents (depth limit = 1).


* **Topic Ownership Boundaries:**
* *Exam Fact:* A connected agent can have its own independent topics, whereas an **inline (child) agent CANNOT contain its own topics** and must inherit the parent agent's topic collection.