## 1. Core Architecture & Concepts

* **Primary Agent Model Architecture:**
* The primary model assigned to a Copilot Studio agent serves as the core cognitive engine responsible for conversational intent recognition, dynamic planning, tool calling, knowledge retrieval synthesis, and natural language generation.
* Native direct custom model uploads were deprecated/removed from Copilot Studio circa mid-2025; custom fine-tuned or specialized models must now be integrated externally as **Tools** via **Microsoft Foundry** (the evolution/rebranding of Azure AI Foundry).


* **Model Workload Categories:**
* **General Models:** Optimized for high-throughput, low-latency, and cost efficiency. Best suited for everyday conversational interactions, light grounding, drafting, text transformation, summarization, language translation, grounded FAQ retrieval, and simple single-step action automation.
* **Auto Models:** Dynamic, adaptive routing models optimized for mixed-complexity enterprise workloads. Best suited for IT service desks, internal employee assistance, and query streams exhibiting unpredictable cognitive depth.
* **Deep Reasoning Models:** Advanced reasoning models optimized for complex, multi-step deterministic workflows, multi-system orchestration, tool-supported logic flows, and deep document synthesis requiring precise citations. Characterized by higher latency and higher per-message/token billing consumption.


* **Instruction-Conditioned Deep Reasoning Triggers:**
* Deep reasoning is not automatically applied across every conversational turn. The orchestrator selectively invokes deep reasoning capabilities only when explicit directive keywords (specifically the action keyword `reason`) are embedded within the agent's system instructions for designated steps.


* **Model Release Lifecycle & Maturity Tiers:**
* **Default (GA):** Fully supported, generally available baseline model providing optimal stability, performance, and regional availability for production workloads.
* **Preview:** Publicly accessible models deployed for functional evaluation. Subject to performance variability, latency shifts, and capacity fluctuations; not recommended for SLA-bound production environments.
* **Experimental:** Pre-preview release tier representing cutting-edge architectures. Subject to severe variance in response quality, token consumption, latency, and regional uptime.
* **Retired (Superseded):** A former default model phased out in favor of a newer release. Can be extended via administrator opt-in for an operational grace period of up to **30 days** post-deprecation before hard cutover to the new default model.


* **Third-Party Sub-Processor Ecosystem (Anthropic & xAI):**
* Support for non-native foundation models (such as **Anthropic Claude** or **xAI**) operates under Microsoft's external sub-processor governance framework.
* Multi-tier tenant governance requires explicit administrator enablement across both the Microsoft 365 Admin Center (licensing and sub-processor consent boundary) and the Power Platform Admin Center (runtime capability enablement).



---

## 2. Platform Features & Capabilities

* **Primary Model Selection & UI Navigation:**
* Navigation: Agent Canvas $\rightarrow$ **Overview** tab $\rightarrow$ **Model** dropdown.
* Allows toggling between available runtime model tiers (General, Auto, Deep, Preview, GA).


* **Deep Reasoning Configuration:**
* Path: Top command bar $\rightarrow$ **Settings** $\rightarrow$ **Generative AI** $\rightarrow$ Enable **Deep reasoning** toggle $\rightarrow$ Save.
* Telemetry Inspection: Deep reasoning execution steps can be audited during runtime inside the test canvas via the **Activity map** located in the left tracking pane.


* **Retired Model Retention Toggle:**
* Path: **Settings** $\rightarrow$ **Generative AI** $\rightarrow$ Toggle **Continue using retired models** (grants up to 30 days of execution on deprecated models).


* **Custom Models via Microsoft Foundry Tools:**
* Custom models cannot be selected as the primary agent reasoning engine directly from the main dropdown; they must be bound as callable **Tools** inside topics or prompt actions.
* Navigation: **Tools** tab (or Agent Overview $\rightarrow$ scroll to **Tools** $\rightarrow$ **+ Add a tool**) $\rightarrow$ Select **New prompt** $\rightarrow$ Change Model $\rightarrow$ Select **Microsoft Foundry models (+)**.
* Insertion within Topics: Visual Canvas $\rightarrow$ Click node insert (`+`) $\rightarrow$ **Add a tool** $\rightarrow$ **New prompt** $\rightarrow$ Bind Foundry model.


* **Microsoft Foundry Connection Parameters:**
* **Model Deployment Name:** The specific deployment identifier provisioned in Microsoft Foundry.
* **Base Model Name:** The foundational model architecture (e.g., fine-tuned Llama, Mistral, or custom weights).
* **Azure Model Endpoint URL:** The HTTPS REST endpoint hosting the model inference target.
* **API Key:** The encrypted authentication key granting authorization to the endpoint.
* **Model Description:** Semantic instructions detailing for the Copilot Studio orchestrator when and why this custom model tool should be invoked.


* **Environment-Level Governance (Power Platform Admin Center):**
* URL: `admin.powerplatform.microsoft.com`
* Path: **Manage** $\rightarrow$ **Environments** $\rightarrow$ Select Environment $\rightarrow$ **Settings** (top bar) $\rightarrow$ **Product** $\rightarrow$ **Features** $\rightarrow$ Scroll to **AI Builder / Copilot Studio** $\rightarrow$ Toggle **Enable preview and experimental AI models in Copilot Studio**.


* **Cross-Tenant Sub-Processor Governance (Microsoft 365 Admin Center):**
* URL: `admin.cloud.microsoft`
* Path: Navigation menu $\rightarrow$ **Settings** $\rightarrow$ **Copilot settings** $\rightarrow$ **Data access** tab $\rightarrow$ **AI providers operating as Microsoft sub-processors** $\rightarrow$ Toggle **Anthropic** / **xAI** from *Disabled* to *Enabled*.
* *Prerequisite Flow:* Must be enabled in M365 Admin Center first before the toggle becomes actionable in the Power Platform Admin Center.



---

## 3. Formulas, Expressions & Schemas

### Microsoft Foundry Model Tool Binding Schema

When registering a Microsoft Foundry custom model within a Copilot Studio Prompt tool, the underlying action definition consumes the following JSON schema:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "MicrosoftFoundryToolConnection",
  "type": "object",
  "properties": {
    "modelDeploymentName": {
      "type": "string",
      "example": "custom-risk-scoring-deployment"
    },
    "baseModelName": {
      "type": "string",
      "example": "meta-llama-3.1-70b-instruct"
    },
    "azureModelEndpointUri": {
      "type": "string",
      "format": "uri",
      "example": "https://foundry-custom-model.services.ai.azure.com/models"
    },
    "apiKey": {
      "type": "string",
      "format": "password",
      "description": "Secure vault or connection reference token"
    },
    "modelDescription": {
      "type": "string",
      "example": "Invoked strictly for scoring financial underwriting risk when tabular balance sheets are supplied."
    }
  },
  "required": ["modelDeploymentName", "baseModelName", "azureModelEndpointUri", "apiKey"]
}

```

### Prompt Node with Deep Reasoning Directive

Instruction syntax configured within the Agent Instructions or within a prompt node to invoke the deep reasoning engine:

```markdown
# Agent Instructions & Execution Directives

## Processing Protocol
1. For simple policy queries (vacation allowances, office locations), retrieve grounded answers directly using General synthesis.
2. For cross-border tax compliance calculations and multi-system dispute analysis, you must **reason** step-by-step through the statutory rules, cite external regulatory sources, and output a validated risk determination.

```

### Power Fx: Dynamic Tool Invocation and Fallback Routing

When orchestrating a custom prompt tool backed by a Microsoft Foundry model within a topic:

```powerfx
// Evaluates user query complexity; conditionally routes to custom prompt tool
If(
    Topic.TransactionRiskScore > 75,
    Set(
        Topic.CustomModelOutput,
        'ab100_RiskAnalysisPrompt'.Predict(
            {
                CustomerProfile: Topic.CustomerJsonPayload,
                EvaluationCriteria: "Strict Compliance"
            }
        ).GeneratedText
    ),
    Set(Topic.CustomModelOutput, "Standard risk threshold met. No escalation required.")
)

```

---

## 4. Best Practices & Design Patterns

* **Targeted Model Tier Allocation:**
* **Cost & Latency Minimization:** Do not assign Deep reasoning models as the default engine for routine, transactional, or standard informational copilots. Reserve Deep reasoning for analytical agents requiring complex synthetic logic, long-form multi-document comparison, and statutory verification.
* **General Models for High Concurrency:** Deploy General models for customer-facing public web chat or internal Tier-1 helpdesks to minimize message latency and prevent rapid capacity pack exhaustion.


* **Instruction Precision for Deep Reasoning:**
* Because enabling Deep reasoning in settings does not universally force all actions to run through high-latency reasoning chains, explicitly structure prompt workflows with the trigger verb `reason` on complex tasks.


* **Two-Step Cross-Tenant Enablement Workflow:**
* When configuring third-party LLMs (Anthropic Claude, xAI), coordinate administrative access beforehand: the global tenant administrator must first approve the sub-processor agreement in `admin.cloud.microsoft` before Power Platform environment administrators can activate the models in `admin.powerplatform.microsoft.com`.


* **Graceful Model Deprecation Planning:**
* When Microsoft issues a deprecation announcement for an active default model, immediately enable **Continue using retired models** in sandbox environments. Use the 30-day grace period to run regression suites against the upcoming GA default model before production auto-switches.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Custom Models Cannot Be the Primary Orchestration Engine:**
* Custom models (fine-tuned models, open-source weights hosted in Foundry) **cannot** be selected directly in the agent's primary model dropdown. They can **only** be consumed downstream via tools, actions, or custom prompts.


* **Cross-Geo Data Residency Implications:**
* Enabling Deep reasoning, preview models, or third-party models (Anthropic, xAI) frequently routes data across geographic regions. Runtime payloads and prompt text may be transmitted and stored in the United States or other regional datacenters where Microsoft sub-processors operate, bypassing strict local geo-boundaries.


* **The 30-Day Retirement Hard Wall:**
* Enabling *Continue using retired models* extends execution on deprecated foundation models for a **maximum of 30 days**. Once this window elapses, the platform automatically forces the agent onto the current default model; there are no exceptions or permanent pinning options.


* **Deep Reasoning Latency Penalty:**
* Deep reasoning involves multi-pass internal chain-of-thought processing. Response times are significantly slower than General or Auto models; incorporating this tier into synchronous interactive phone/telephony channels will trigger silence or utterance timeouts unless latency messaging is configured.


* **Admin Disablement False Diagnoses:**
* Models appearing with the status *"Disabled by your admin"* in the model picker often do not reflect a policy restriction inside Copilot Studio itself, but rather an unconfigured prerequisite toggle in the Microsoft 365 Admin Center's sub-processor settings.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Custom Model Ingestion:**
* *Trap:* A scenario asks how to set an Azure AI / Microsoft Foundry custom-trained model as the agent's primary brain in the Overview settings.
* *Fact:* You cannot assign custom models as the primary agent model. You must register the custom model deployment as an external **Tool / Prompt** and invoke it within topics or instructions.


* **Exam Trap — Model Workload Characteristics:**
* **General:** Fastest response time, lowest cost, best for translation, summarization, simple FAQs.
* **Auto:** Dynamic balancing, optimal for unpredictable IT/HR helpdesk query distributions.
* **Deep:** High latency, highest cost, multi-step logical reasoning, long document synthesis with strict citations.


* **Exam Trap — Activating Deep Reasoning:**
* Merely turning on the toggle in **Settings $\rightarrow$ Generative AI** is insufficient. The agent requires the specific keyword **`reason`** inside its instructions or prompt nodes to trigger the deep reasoning path.


* **Exam Trap — Retired Model Policy:**
* If an organization needs to temporarily maintain compatibility after a model deprecation wave, they can toggle **Continue using retired models** to prolong usage for exactly **30 days**.


* **Exam Trap — Enabling Anthropic / Third-Party Models:**
* *Question Scenario:* A developer attempts to select Anthropic Claude in Copilot Studio, but the option is disabled. Which administrative action is required first?
* *Correct Sequence:* The tenant administrator must enable Anthropic as an approved sub-processor in the **Microsoft 365 Admin Center** (`admin.cloud.microsoft`) under Copilot Data Access settings *before* it can be enabled in the Power Platform Admin Center or selected in Copilot Studio.