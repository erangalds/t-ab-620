## 1. Core Architecture & Concepts

* **Prompt Actions (Custom Prompts as Tools):**
* Modular, reusable AI-driven capabilities created via the Power Platform Prompt Builder (AI Builder engine) and exposed directly to Copilot Studio as tools.
* Operate independently of the main agent’s generative model. This architectural boundary allows makers to assign a specialized foundational model (e.g., Azure OpenAI models, custom models deployed via Microsoft Foundry) to a single discrete task without altering the global agent model.


* **Decoupled & Isolated Execution Lifecycle:**
* Encapsulates multi-step logic into an atomic action, preventing "instruction bleed" (where the LLM interleaves, skips, or partially executes separate instructions across turns).
* Enforces strict deterministic sequencing (e.g., executing step 4 to retrieve an employee role in its entirety before invoking subsequent actions like dispatching an email).


* **Model Temperature Mechanics:**
* Defines the degree of randomness/stochasticity in the model's token selection.
* *Scale:* Within Copilot Studio/Power Platform Prompt Builder, the temperature slider ranges strictly between **0.0 and 1.0** (unlike raw OpenAI APIs, which range from 0.0 to 2.0).
* *Low Temperature ($\approx 0.0$):* Highly focused, deterministic, and repeatable. Ideal for classification, entity lookup, and structured data extraction.
* *High Temperature ($\approx 1.0$):* Random, diverse, and expressive. Ideal for creative generation and open-ended text synthesis.


* **Dual-Layer Tool Naming Architecture:**
* *Global Asset Name (Real Name):* The underlying Dataverse prompt definition (e.g., `employee role tool`). Renaming this updates the primary AI Builder prompt asset globally across all agents, flows, and apps referencing it.
* *Local Tool Name (Alias / Display Name):* The agent-level contextual identifier (e.g., `employee role`) assigned when the tool is attached to a specific agent's toolbox.


* **Orchestration & Invocation Modes:**
* *Dynamically fill with AI:* The generative orchestrator analyzes context, trigger payloads, or conversation history to extract and map values to prompt input slots.
* *Anytime vs. Referenced Only:* Configures whether the generative engine can call the prompt autonomously or only when explicitly referenced in authoring instructions via an object badge (`/`) or within a deterministic topic canvas.



---

## 2. Platform Features & Capabilities

* **Prompt Tool Creation Path:**
* Navigate to **Tools** tab $\rightarrow$ Click **+ Add a tool** $\rightarrow$ Select **Prompt**.
* *Model Selection:* Choose the base LLM model from the dropdown (or connect custom enterprise models from **Microsoft Foundry**).


* **Natural Language Prompt Generation & Meta-Prompting:**
* Enter natural language intent (e.g., mapping employees to job roles: *Philip = IT professional, Jane = Social media manager, Maria = Secretary, Susan = HR manager*).
* Prompt Builder synthesizes structured prompt instructions along with explicit guardrails (e.g., *"Do not infer or add any additional information beyond what is given"*).
* Makers can **Keep**, **Regenerate**, or **Clear prompt**.


* **Prebuilt Prompt Library Templates:**
* Accessible via the ellipsis menu (**...**) $\rightarrow$ **Prompt library**:
* *Summarize text*
* *Extract information from text*
* *Classify text*
* *Respond to a complaint*
* *Sentiment analysis* (Positive, Negative, Neutral)
* *Detect language*




* **Advanced Prompt Settings & Hyperparameters:**
* **Temperature:** Adjustable slider from `0.0` to `1.0`.
* **Data Source Records Retrieved:** Configurable threshold governing how many grounding records are fetched from data sources; ranges from **30 (default)** up to **1,000 records**.
* **Citation Links:** Toggle to include or suppress reference citation links in responses.
* **Code Interpreter:** Sandbox environment allowing the prompt to execute Python code for advanced calculations, complex logic, file processing (Word, Excel, PowerPoint, PDF), and writing directly to Dataverse tables.
* **Content Moderation:** Adjustable safety filtering level (Default: `Medium`).


* **Input Types & Grounding Modalities:**
* *Text Input:* Dynamic slot for string variables (e.g., `EmployeeName`); accepts **Sample Data** for test simulation (e.g., `Jane`).
* *Image / Document Input:* Multimodal input allowing makers to supply sample images or documents (PDF, DOCX) directly into prompt evaluations.
* *Power Fx Formulas:* Dynamic expressions evaluated at runtime to ground prompts.
* *External SaaS Connectors:* Built-in direct integrations to **Salesforce**, **Oracle Database**, and **Zendesk**.
* *Dataverse Column Binding:* Direct grounding on specific table attributes (e.g., binding columns from the `Address` table).


* **Tool Execution & Output Settings:**
* *Input Slot Binding:* Configure parameters to `Dynamically fill with AI` or hardcode custom values.
* *Post-Execution Action:* Set completion behavior to `Don't respond` (suppresses conversational chat output, returning raw values back to the agent instructions pipeline).



---

## 3. Formulas, Expressions & Schemas

### Prompt System Instruction Template

When authoring the custom prompt, hardcoded lookup tables and instruction boundaries are compiled as follows:

```markdown
You are an HR verification assistant.
Your task is to return the exact job role for the provided employee name.

### Role Directory:
- Philip: IT professional
- Jane: Social media manager
- Maria: Secretary
- Susan: HR manager

### Instructions:
1. Identify the employee specified in the input parameter [EmployeeName].
2. Match the name against the Role Directory.
3. Return only the job role title. 
4. Guardrail: Do not infer, extrapolate, or add any additional information beyond what is explicitly given. If the employee is not found, return "Unknown".

Input:
EmployeeName: {EmployeeName}

```

### Dataverse AI Model Storage & Invocation Schema

Custom prompts are stored as records in the Dataverse **`msdyn_AIModel`** table. When invoked at runtime by Copilot Studio or the Predict API, the execution request follows this open-type schema:

```json
{
  "operationId": "Predict",
  "modelId": "a1b2c3d4-e5f6-7890-abcd-1234567890ab",
  "version": "2.0",
  "requestv2": {
    "EmployeeName": "Jane"
  },
  "settings": {
    "temperature": 0.0,
    "maxRecordsRetrieved": 30,
    "contentModeration": "Medium",
    "enableCodeInterpreter": false
  }
}

```

### Output Response Payload Schema (Internal Copilot Studio Tool Binding)

The raw response payload emitted from the prompt action and ingested back into the agent context:

```json
{
  "status": "Succeeded",
  "responsev2": {
    "text": "Social media manager",
    "finishReason": "stop",
    "executionTimeMs": 340,
    "billingCreditsUsed": 1
  }
}

```

---

## 4. Best Practices & Design Patterns

* **Isolation of Concerns via Prompt Tools:**
* Extract complex, multi-branch, or fragile extraction tasks from the main agent instructions into dedicated prompt tools. This enforces atomic execution and eliminates prompt drift across long reasoning chains.


* **Instruction Audit Verification:**
* *Verification Antipattern:* Do not assume AI-generated prompt instructions captured all your initial sample inputs.
* *Remediation:* Always review generated prompt instructions before saving. Meta-prompt generators often drop entity lists or specific hardcoded mappings during draft creation; manually insert bulleted/hyphenated lists to ensure exact retrieval.


* **Deterministic Temperature Calibration:**
* For entity extraction, role lookups, classification, or strict data transformations, always set temperature to **0.0** (or near zero) to minimize variance and hallucination.


* **Descriptive Semantic Metadata:**
* Provide explicit, functional tool names and descriptions (e.g., *"This tool returns the employee role. It requires the name of the employee"*). The generative orchestrator relies strictly on tool names and descriptions to assess invocation relevance.


* **Global vs. Local Name Alignment:**
* Keep global prompt asset names and local agent tool names closely aligned to prevent administrative confusion during cross-environment ALM deployments and solution packaging.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Temperature Range Boundary:**
* Copilot Studio / Power Platform Prompt Builder restricts temperature to **0.0 – 1.0**. Entering or expecting raw OpenAI limits ($0.0 - 2.0$) is invalid in this UI.


* **Data Source Record Retrieval Ceilings:**
* Data source record retrieval limits are constrained to a minimum of **30** and an upper platform ceiling of **1,000 records**.


* **Global Impact of Prompt Renaming / Editing:**
* Prompts are shared Power Platform / Dataverse environment assets. Modifying instructions, inputs, or names on the global prompt asset applies changes everywhere the prompt is used (including Power Automate flows, Power Apps, and other Copilot Studio agents), which can break downstream dependencies.


* **Local vs. Global Tool Name Sync Glitch:**
* Renaming a prompt globally does not always reflect instantaneously in the agent's local tools pane. A manual page refresh is required to reconcile local UI cache states with Dataverse.


* **Code Interpreter Prerequisites:**
* Enabling the Code Interpreter toggle requires the feature to be enabled at the Power Platform environment level by an administrator via the Power Platform Admin Center or Environment Management Settings APIs.


* **Billing & Capacity Consumption:**
* Every prompt action execution incurs runtime latency (visible in test metrics) and consumes tenant **AI Builder / Copilot capacity credits**.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Model Independence (Per-Action Models):**
* *Exam Fact:* An agent can use different LLMs within the same orchestration pipeline. The main agent can run the standard orchestrator, while specific Prompt Actions can be configured with distinct base models or custom models from **Microsoft Foundry**.


* **Grounding Options for Prompts vs. Agents:**
* *Exam Trap:* Confusing prompt grounding with agent-level knowledge.
* *Fact:* Prompts support grounding via **Dataverse table columns**, **Power Fx expressions**, **Salesforce**, **Oracle Database**, and **Zendesk** directly within the prompt builder, as well as multimodal document/image inputs. Uploaded SharePoint sites and broad unstructured websites belong to agent/copilot knowledge sources, not prompt grounding.


* **Code Interpreter Capabilities:**
* *Exam Fact:* Turning on the **Code Interpreter** inside a prompt allows it to generate and run Python code, process complex Excel/Word/PDF files, perform mathematical computation, and write data into Dataverse tables.


* **Global Scope of Prompts (ALM Impact):**
* *Exam Trap:* Assuming editing a prompt inside Copilot Studio only affects that specific agent.
* *Fact:* Prompts are independent Dataverse artifacts (`msdyn_AIModel`). Any edit made to a prompt applies across all agents, flows, and canvas apps referencing that prompt.


* **Input Binding for Autonomous Execution:**
* *Exam Scenario:* An autonomous agent running without user interaction needs to invoke a prompt. How should prompt inputs be bound?
* *Answer:* Inputs must either be **Dynamically filled with AI** (extracted by the orchestrator from context/trigger payloads) or mapped to specific variables/custom values. They cannot prompt an end user for interactive input.