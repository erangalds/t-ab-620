## 1. Core Architecture & Concepts

* **Platform Ecosystem Positioning:**
* **Microsoft Copilot Studio:** Belongs to the **Microsoft Power Platform** ecosystem. Optimized for low-code/no-code authoring, rapid deployment, native Microsoft 365 / Microsoft Graph integration, citizen developers, business technologists, and IT administrators.
* **Microsoft Foundry (formerly Azure AI Studio / Azure AI Foundry):** Belongs to the **Microsoft Azure** family (`ai.azure.com`). Optimized for code-first AI development, full foundation model control, hyperparameter fine-tuning, complex agent orchestration, pro-developers, and data scientists.


* **Foundry Architectural Models & Portal Flavors:**
* **Classic Foundry (Azure AI Studio Classic):** Built around Azure OpenAI projects, legacy hubs, and traditional Azure AI Service resources.
* **New Foundry Experience:** Specialized modern interface built around unified Foundry projects. Architected specifically for **multi-agent applications**, enhanced modular orchestration, flexibility, dynamic scaling, and deep integration with the open-source **Microsoft Agent Framework** / Semantic Kernel.


* **Knowledge Grounding via Vector Indexing:**
* When unstructured files (PDF, DOCX, PPTX) are ingested into Foundry, they are parsed, chunked, and embedded into a managed **Vector Index** (backed by Azure AI Search infrastructure).
* Differs fundamentally from Copilot Studio’s native Dataverse/SharePoint spreadsheet grounding: Foundry’s direct vector index does not natively process raw `.xlsx` / tabular spreadsheets directly through the file upload tool without prior tabular conversion or dedicated data connectors (e.g., Azure AI Search indexers, Fabric, or custom code).


* **Billing & Metering Abstraction:**
* *Copilot Studio:* Billed primarily via Power Platform licensing / pre-allocated tenant **Copilot Credits** consumed per message, turn, or action step.
* *Microsoft Foundry:* Direct Azure infrastructure consumption billing based on **raw tokens consumed** (Prompt/Input tokens + Completion/Output tokens), model deployment tier (pay-as-you-go vs. provisioned throughput), and underlying Azure resource consumption (storage, vector indexing compute, and search units).


* **Evaluator Guardrails & Execution Tracing:**
* Foundry captures granular execution metrics per conversational turn: total latency (seconds), total token usage, tools invoked, and integrated AI evaluation metrics (e.g., **Intent Resolution** and **Task Adherence**).



---

## 2. Platform Features & Capabilities

* **Portal Access & Navigation:**
* **Azure Portal Path:** `portal.azure.com` $\rightarrow$ Search for *Microsoft Foundry* $\rightarrow$ Create resource.
* **Standalone Portal Path:** `ai.azure.com` (dedicated studio experience).


* **Project Provisioning Parameters:**
* `Project Name`: Identifier for the developer workspace (e.g., `AB100Foundry`).
* `Subscription`: Active Microsoft Azure subscription (free tiers include $200 credit within 30 days).
* `Resource Group`: Existing or auto-created target deployment group.
* `Microsoft Foundry Resource`: Named parent resource anchor linked to the project.
* `Region Strategy`: Recommended deployment regions: **East US 2** and **Sweden Central**. These regions are the primary release rings containing the latest platform features, model availability, and multi-agent enhancements.


* **Agent Creation Canvas:**
* **Path:** Left Navigation $\rightarrow$ **Build** $\rightarrow$ **Agents** $\rightarrow$ **Create agent**.
* **Metadata:** Agent Name (e.g., *Employee Leave Tracker*), Instructions, Model Selection.


* **Model Selection & Capability Compatibility Constraints:**
* Foundry hosts diverse foundation models (OpenAI GPT-4.1 / GPT-4o, reasoning/deep models, general models, small language models).
* **Model-to-Tool Matrix:** Tools are tightly coupled to model capabilities. If an incompatible model is chosen, the UI explicitly flags: *"This tool doesn't work with the model you selected."*
* *Example:* GPT-4.1 supports standard REST/OpenAPI and function-calling tools, whereas specialized capabilities like **Computer Use** are restricted to specific vision/multimodal reasoning models.


* **Tool Ecosystem Comparison:**
* *Copilot Studio:* Out-of-the-box access to 1,000+ pre-built Power Platform certified connectors (Microsoft 365, Teams, Outlook, Dataverse, SQL, Salesforce).
* *Microsoft Foundry:* Geared for pro-code extensibility:
* Azure Functions & Logic Apps
* OpenAI standard tools (compliant with OpenAPI 3.0 specification)
* Model Context Protocol (MCP) servers
* Code Interpreter (sandboxed Python execution)
* Azure AI Search, Bing Search, Microsoft Fabric, and SharePoint knowledge tools




* **Knowledge Ingestion & File Type Limits:**
* Supported file formats for direct vector index ingestion: `.pdf`, `.docx`, `.pptx`, `.txt`.
* **Unsupported for Direct Document Upload:** Spreadsheets (`.xlsx`, `.csv` without specialized tabular connectors). Tabular data must be converted to structured documents (e.g., formatted PDF) or routed through a structured data source (Azure SQL, Fabric, or custom vector index).


* **Testing Canvas & Runtime Observability:**
* Surfaces live multi-turn chat testing.
* Displays runtime telemetry bar beneath responses:
* Active Model Name.
* Execution Duration (latency).
* Total Tokens Billed.
* Tools Executed.
* Built-in Evaluation Verdicts (e.g., *Passed Intent Resolution*, *Failed Task Adherence*).
* Raw JSON Execution Logs, Copy Response, and Session Reset (*New Chat*).





---

## 3. Formulas, Expressions & Schemas

### System Instructions Template (Foundry Agent System Metaprompt)

Unlike Copilot Studio's topic nodes, Foundry agents rely heavily on explicit, markdown-delimited system prompt engineering to enforce entity gathering and authorization checks:

```markdown
# Role & Identity
You are the Employee Leave Tracker agent. Your purpose is to assist authenticated employees in retrieving their remaining vacation and sick leave balances.

# Authentication & Guardrails
- Before disclosing any confidential leave records, you MUST obtain and verify the user's Employee ID or confirmed full name.
- If the user asks "How much leave do I have?" without providing credentials, respond: "Can you please provide your employee ID or another authentication detail?"
- Never fabricate leave balances. Ground all numerical figures strictly on the retrieved knowledge index.

# Output Formatting
- State vacation leave and sick leave distinctly:
  * Vacation Leave: [X] days
  * Sick Leave: [Y] days

```

### JSON Schema: Agent Execution Telemetry Record

The telemetry bar in Microsoft Foundry outputs structured execution traces for each turn:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "FoundryAgentTurnTelemetry",
  "type": "object",
  "properties": {
    "session_id": { "type": "string", "format": "uuid" },
    "turn_id": { "type": "string", "format": "uuid" },
    "model": { "type": "string", "example": "gpt-4.1" },
    "latency_seconds": { "type": "number", "example": 2.14 },
    "usage": {
      "type": "object",
      "properties": {
        "prompt_tokens": { "type": "integer", "example": 842 },
        "completion_tokens": { "type": "integer", "example": 68 },
        "total_tokens": { "type": "integer", "example": 910 }
      },
      "required": ["prompt_tokens", "completion_tokens", "total_tokens"]
    },
    "tools_invoked": {
      "type": "array",
      "items": { "type": "string" },
      "example": ["vector_index_retrieval"]
    },
    "evaluations": {
      "type": "object",
      "properties": {
        "intent_resolution": { "type": "string", "enum": ["Passed", "Failed"], "example": "Passed" },
        "task_adherence": { "type": "string", "enum": ["Passed", "Failed"], "example": "Failed" }
      },
      "required": ["intent_resolution", "task_adherence"]
    }
  },
  "required": ["session_id", "model", "latency_seconds", "usage", "evaluations"]
}

```

### Power Fx Equivalent (Copilot Studio Comparison)

To replicate Foundry’s explicit identity-check prompt inside a Copilot Studio deterministic Topic node:

```powerfx
// Evaluate if employee identity exists in Topic or Global context
If(
    IsBlank(Topic.EmployeeName) And IsBlank(Topic.EmployeeId),
    Set(Topic.PromptMessage, "Can you please provide your employee ID or another authentication detail?"),
    Set(Topic.PromptMessage, "Retrieving leave details for " & Coalesce(Topic.EmployeeName, Topic.EmployeeId))
)

```

---

## 4. Best Practices & Design Patterns

* **Hybrid Coexistence Architecture:**
* Use **Copilot Studio** as the conversational front-end / digital assistant layer to take advantage of out-of-the-box Microsoft 365, Teams, and Power Platform connectors.
* Use **Microsoft Foundry** as the backend intelligence engine when complex model routing, fine-tuned domain models, Python sandboxed code execution (Code Interpreter), or advanced multi-agent orchestration (via Microsoft Agent Framework / Semantic Kernel) are required.


* **Regional Deployment Alignment:**
* When provisioning new Foundry projects, select feature-rich regions such as **East US 2** or **Sweden Central** to guarantee access to the latest generative model updates, multi-agent frameworks, and vector search features.


* **Knowledge Pre-Processing for Foundry Ingestion:**
* Do not upload raw tabular `.xlsx` files directly to a Foundry Vector Index. Tabular data should be converted into clear semi-structured documents (e.g., Markdown, structured text, or text-heavy PDF tables with explicit column headers) or hosted inside an indexed database / Azure AI Search schema to prevent chunking fragmentation.


* **Prompt-Driven Verification Guardrails:**
* Because Foundry agents lack deterministic topic canvas branches by default, developers must write defensive metaprompt rules that force the agent to request authentication parameters (e.g., Employee ID) before executing knowledge retrieval tools.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Spreadsheet Parsing Limitation:**
* Copilot Studio natively connects to and parses Excel spreadsheets stored in SharePoint/OneDrive via generative answers or Excel connectors. In contrast, Microsoft Foundry's direct file upload indexer **rejects `.xlsx` files** and requires `.pdf`, `.docx`, `.pptx`, or `.txt`.


* **Model-Tool Coupling Traps:**
* Tools in Foundry cannot be mapped universally to any model. Attempting to attach specialized tools (such as *Computer Use* or specific function-calling schemas) to unsupported foundation models will fail validation, rendering the tool inactive.


* **Cost & Consumption Model Shift:**
* Moving an agent workload from Copilot Studio to Microsoft Foundry changes the commercial model: operations transition from fixed Power Platform license / Copilot Credit allocations to pay-as-you-go, per-token, consumption-based Azure infrastructure billing.


* **Classic vs. New Foundry Project Boundary:**
* Projects created under Classic Foundry cannot seamlessly leverage the modern multi-agent application runtime. Multi-agent workflows strictly require the **New Foundry** project interface.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Copilot Studio vs. Microsoft Foundry Positioning:**
* *Question:* "An enterprise requires an AI agent that connects to 50+ pre-built third-party business systems with minimal code and integrates directly into Microsoft Teams and Microsoft 365. Which tool should you recommend?"
* *Correct Answer:* **Microsoft Copilot Studio** (Low-code, Power Platform ecosystem, native M365 channels, 1,000+ connectors).
* *Question:* "A data science team needs full control over foundation model weights, custom Python code interpreters, fine-tuning, and direct token-level billing in Azure. Which tool should you recommend?"
* *Correct Answer:* **Microsoft Foundry** (Azure ecosystem, pro-code, token-based, model catalog).


* **Exam Trap — Portal URLs:**
* Copilot Studio Portal: `copilotstudio.microsoft.com`
* Microsoft Foundry Portal: `ai.azure.com`
* Power Platform Admin Center: `admin.powerplatform.microsoft.com`


* **Exam Trap — Knowledge File Upload Support:**
* If an exam question asks why an Excel workbook (`.xlsx`) containing employee data cannot be uploaded directly to an agent's knowledge index via the Foundry studio UI, the correct rationale is that **Foundry's direct vector index upload supports PDF, DOCX, PPTX, and TXT, but does not support direct spreadsheet ingestion**.


* **Exam Trap — Evaluation Metrics Terminology:**
* Foundry provides built-in evaluators that score responses during testing:
* **Intent Resolution:** Measures whether the model correctly identified what the user was trying to achieve.
* **Task Adherence:** Measures whether the model strictly executed the assigned task instructions and adhered to all constraints without hallucinating or dropping mandatory steps.