## 1. Core Architecture & Concepts

* **Agent Provisioning Modes:**
* **Natural Language Copilot Builder:** An AI-assisted conversational onboarding route where the maker enters an initial statement of purpose, and the platform automatically generates an initial name, persona description, starter topics, and trigger phrasing.
* **Blank Agent Provisioning:** Direct runtime instantiation bypassing generative scaffolding. Creates an empty agent shell with base system topics and default conversational runtime handlers.
* **Advanced Create:** Explicit architectural configuration provisioning. Exposes core foundational settings before tenant deployment:
* *Primary Language:* Determines tokenization, stemming, natural language understanding (NLU) models, and entity recognition pipelines.
* *Target Solution:* Binds the agent directly to a specific Dataverse unmanaged solution boundary rather than defaulting to system/common solutions.
* *Internal Schema Name (`SchemaName`):* Sets the immutable programmatic identifier prefixed with the solution publisher namespace (e.g., `cr123_`, `ab100_`).


* **Solution Import:** Instantiates an agent via importing pre-packaged managed or unmanaged Dataverse solution `.zip` files containing the bot definition and sub-components.


* **Orchestration Paradigms:**
* **Classic Orchestration (Deterministic / Topic-Based):**
* Dialogue flow is governed strictly by hardcoded topic trigger phrases, deterministic condition logic, explicit branching nodes, manual entity collection, and fixed question-response pipelines.
* Generative execution is restricted; conversational traversal relies on exact or fuzzy intent matching across trigger lists.


* **Generative AI Orchestration (Dynamic Chaining / Generative Mode):**
* Uses large language models (LLMs) to dynamically determine user intent, chain disparate topics, select actions/tools, dynamically fill slot parameters, and synthesize natural language answers from grounded enterprise knowledge sources.




* **Agent Lifecycle Architecture in Dataverse:**
* When an agent is provisioned, Dataverse instantiates records across underlying core tables: `bot` (stores high-level configuration, schema name, orchestration state) and `botcomponent` (stores individual topics, system triggers, dialog components, and environment variables).
* **Studio-to-Canvas Sync:** Metadata updates (e.g., changing the display name) update the underlying Dataverse schema asynchronously; the active authoring canvas and local test canvas runtime reflect changes only after a browser refresh or local test session re-initialization.



---

## 2. Platform Features & Capabilities

* **UI Navigation Paths:**
* **Create Agent:** Left-hand navigation $\rightarrow$ **Agents** $\rightarrow$ **Create blank agent** (or dropdown caret $\rightarrow$ **Advanced create** / **Import agent**).
* **Agent Metadata Configuration:** Agent Overview canvas $\rightarrow$ **Edit** (pencil icon next to agent title):
* *Name:* Display name presented across communication channels and system greeting headers (e.g., `A.VacationAllowanceClassic`).
* *Icon:* Custom image asset representing the agent persona across client endpoints.
* *Description / Instructions:* Meta-description instructing users or host orchestrators on the agent's specific domain expertise.


* **Orchestration Settings Navigation:** Top-right studio bar $\rightarrow$ **Settings** $\rightarrow$ **Generative AI** tab:
* *How should your agent decide how to respond?* toggle between **Generative (default)** and **Classic**.
* Saving the selection updates the agent runtime profile to execute either dynamic planner routing or rigid trigger-phrase topic resolution.


* **Integrated Test Canvas:**
* Toggle access via the **Test** button in the upper-right or collapsible side-dock.
* Displays dynamic system greeting (e.g., *"Hello, I am [AgentName], how can I help?"*).
* Test environment requires page refresh or reset button click (`Restart conversation`) to pull modified Dataverse bot display attributes.





---

## 3. Formulas, Expressions & Schemas

### Dataverse Agent (`bot`) Entity Provisioning Payload

When creating an agent via **Advanced Create**, the platform submits a provisioning payload to the Dataverse Web API. The schema name, publisher prefix, and orchestration mode map directly into the entity record:

```json
{
  "name": "A.VacationAllowanceClassic",
  "schemaname": "ab100_vacationAllowanceClassic",
  "language": 1033,
  "solutionid": "3367448e-d1a7-40fb-abcb-184045f233cf",
  "authenticationmode": "Integrated",
  "configuration": {
    "orchestrationsettings": {
      "useDynamicChaining": false,
      "generativeOrchestrationEnabled": false,
      "orchestrationMode": "Classic"
    },
    "defaultLanguage": "en-US",
    "supportedLanguages": ["en-US"]
  }
}

```

### System Variable Evaluation in Default Greeting

In classic orchestration mode, the agent initializes the conversation using the default system topic `Greeting`, resolving system-level identity variables dynamically via Power Fx:

```powerfx
// Default runtime greeting resolution evaluated on conversation start
Concatenate(
    "Hello, I am ", 
    System.Bot.Name, 
    ". How can I help you today?"
)

```

### Business Metric Definition Schema (Goal Alignment)

To ground agent development in measurable ROI metrics, the goal definitions described in the transcript follow structured quantifiable target schemas:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "AgentProjectMetrics",
  "type": "object",
  "properties": {
    "agentName": { "type": "string", "example": "A.VacationAllowanceClassic" },
    "primaryMetric": { 
      "type": "string", 
      "enum": ["AverageResponseTimeReduction", "InquiryResolutionRate", "CostPerInquiryReduction", "DeflectionRate"] 
    },
    "baselineValue": { "type": "number", "example": 14.5 },
    "targetValue": { "type": "number", "example": 2.0 },
    "unitOfMeasure": { "type": "string", "example": "Minutes" }
  },
  "required": ["agentName", "primaryMetric", "targetValue", "unitOfMeasure"]
}

```

---

## 4. Best Practices & Design Patterns

* **Targeted Agent Scoping (Low Complexity, High Impact):**
* Start agent modernization initiatives on deterministic, high-volume, low-risk corporate policies (e.g., Vacation/Sick Time Allowance lookup) before introducing multi-step autonomous write transactions.


* **Deterministic Baseline Architecture:**
* When regulatory precision, strict auditing, or uniform verbatim compliance responses are mandatory, disable generative orchestration and enforce **Classic Orchestration**. This prevents generative model hallucinations and guarantees execution follows strictly tested topic nodes.


* **Naming Conventions for Multi-Agent Workflows:**
* Adopt systematic prefixing (e.g., `A.`, `B.`, `C.` or domain codes `HR_`, `IT_`, `FIN_`) across agent display names and solution components to maintain clarity across enterprise testing environments and multi-agent catalogs.


* **Cross-Functional Team Governance:**
* Implement an enterprise AI Center of Excellence (CoE) triad:
* *Business Subject Matter Experts (SMEs):* Define business rules, measurable KPIs, and approval policies.
* *Data Engineers / Scientists:* Curate, sanitize, and validate AI-ready structured/unstructured knowledge stores.
* *Professional Developers / IT Admins:* Govern Power Platform ALM, environment variables, Entra ID security scopes, and Dataverse solution packaging.





---

## 5. Technical Limitations, Gotchas & Guardrails

* **Immutable Language Configuration:**
* Primary language selection made during **Advanced Create** or blank creation is **permanent**. You cannot alter the primary base language of an agent once the Dataverse `bot` record is generated; switching base languages requires deleting and recreating the agent or utilizing multi-language localized secondary packs.


* **Client Cache Stale Metadata Gotcha:**
* Modifying an agent's display name or avatar in the Overview tab updates Dataverse asynchronously. The built-in emulator/test pane frequently caches the prior session context (`Hello, I am agent`). A hard browser reload (`Ctrl+F5` / `Cmd+Shift+R`) or resetting the test session via the test canvas refresh control is required to reload the updated metadata payload.


* **Generative vs. Classic Orchestration Lockout:**
* Disabling generative orchestration disables dynamic slot filling, plugin actions, and generative knowledge fallback actions across the agent. All conversational pathways must be backed by explicit trigger phrases and topic nodes; unmapped user inputs will immediately fall through to the system `Fallback` topic.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Blank Create vs. Advanced Create:**
* *Trap:* A scenario requires an agent with a specific Dataverse internal schema name (using a custom publisher prefix) and an isolated unmanaged solution from inception.
* *Fact:* Using **Create blank agent** immediately provisions the agent inside the *Preferred Solution* (or Default Solution if unset) using an autogenerated logical schema name. To explicitly define the internal schema name, solution container, and base language before provisioning begins, the developer must select **Advanced create**.


* **Exam Trap — Orchestration Mode Impact:**
* *Classic Orchestration:* Requires manual entry of **trigger phrases** (typically 5–10 per topic) and deterministic branching logic.
* *Generative AI Orchestration:* Employs the generative orchestration planner to analyze user utterances against topic titles and descriptions, dynamically dispatching actions without relying exclusively on matched trigger phrases.


* **Exam Trap — System Variable Scope:**
* Variables like `System.Bot.Name` and `System.User.DisplayName` are globally readable, read-only system variables managed by the platform runtime. Changing the display name in the studio settings directly alters the string resolved by `System.Bot.Name` during runtime conversation greeting loops.