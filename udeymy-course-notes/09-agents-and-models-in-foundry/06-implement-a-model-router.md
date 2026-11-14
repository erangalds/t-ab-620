## 1. Core Architecture & Concepts

* **Model Router (`model-router`) Architecture:**
* A trained routing and dispatch model deployed as a single unified inference endpoint in Microsoft Foundry.
* Dynamically evaluates inbound user prompts in real time based on linguistic complexity, intent, reasoning depth, and task type, automatically directing the prompt to the most suitable underlying Large Language Model (LLM) or Small Language Model (SLM).
* Eliminates the architectural overhead of provisioning, managing, and governing separate endpoints for diverse model families.


* **Operational Trade-Off Optimization:**
* Balances cost, throughput/latency, and output quality across heterogeneous workloads without requiring client-side branching code.
* Trivial queries (e.g., greetings, basic classifications) are assigned to lightweight, cost-efficient models (e.g., GPT-4.1-nano / GPT-5-nano / SLMs), while computationally intensive reasoning tasks (e.g., complex algorithm implementation, mathematical synthesis) route to high-capability reasoning models (e.g., DeepSeek, frontier GPT models).


* **Decoupled Underlying Model Provisioning:**
* **Zero-Deployment Infrastructure:** Users do not need to individually instantiate, deploy, or maintain underlying models in the Foundry project to include them in the router pool. The Model Router provides native passthrough access to supported models under its single deployment envelope.
* **The Anthropic Claude Prerequisite Exception:** Third-party proprietary models such as **Anthropic Claude** are not automatically provisioned. To route traffic to Claude, the Claude model must already be explicitly deployed within the Azure environment/project before selecting it in the router subset.


* **Data Residency & Deployment Tiers:**
* **Global Standard:** Inbound prompts and completions can be processed and dynamically routed across any Microsoft Azure data center globally. Offers the highest capacity allocation and lowest deployment latency.
* **Data Zone Standard:** Restricts dynamic model processing to data centers within a defined Microsoft geographic data zone (e.g., European Union / Data Zone EUR). Customer data storage at rest remains strictly within the primary geographic region assigned to the Foundry resource.


* **Context Window Asymmetry:**
* The effective context window supported by an underlying model when invoked through the Model Router may be smaller than the native context window available when that model is deployed as a standalone endpoint.



---

## 2. Platform Features & Capabilities

* **Portal Navigation & Discovery:**
* **UI Path:** Microsoft Foundry (`ai.azure.com`) $\rightarrow$ Left Navigation: **Discover** $\rightarrow$ **Models** $\rightarrow$ Search for `model-router`.


* **Deployment Configuration Modes:**
* **Default Settings:** Deploys using the **Balanced** routing strategy across all globally supported underlying models.
* **Custom Settings:** Exposes fine-grained routing parameters, underlying model subset filters, quota limits, and guardrails.


* **Custom Deployment Parameters:**
* `Deployment Name`: Unique identifier for the endpoint (e.g., `model-router-prod`).
* `Deployment Type`: Radio selection between **Global Standard** and **Data Zone Standard**.


* **Model Router Specific Settings:**
* **Routing Mode:**
* *Quality Mode:* Routes exclusively to the highest-performing, most capable model for the prompt, prioritizing reasoning accuracy and depth irrespective of token pricing.
* *Cost Mode:* Employs a broader quality tolerance band (e.g., within 5–6% quality variance), selecting the most cost-effective underlying model.
* *Balanced Mode (Default):* Achieves an operational equilibrium between inference token expenditure and completion quality.


* **Model Pool Selection:**
* *All Supported Models:* Includes all verified foundation models available to the router version (e.g., OpenAI GPT-4/GPT-5 series, DeepSeek, Meta Llama, Mistral).
* *Route to a Subset of Models:* Allows makers to select an explicit allowlist of models (e.g., pairing a lightweight nano model like `gpt-4.1-nano` with a reasoning engine like `DeepSeek-V3` / `DeepSeek-R1`).


* **Model Version Upgrade Policy:**
* Configures runtime behavior when Microsoft updates underlying models.
* Default/Standard: *Upgrade once new default versions become available*.


* **Tokens Per Minute (TPM) Rate Limit:**
* Configures maximum allowed token consumption per minute at the root router deployment level, enforcing unified throttling across all dispatched underlying models.


* **Guardrails & Content Filters:**
* Assigns unified **Azure AI Content Safety** policies (Prompt Shields, Protected Material, Hate/Violence/Sexual/Self-Harm filters) centrally at the router endpoint, covering all downstream models automatically.




* **Runtime Observability in Foundry Playground:**
* Submitting a simple prompt (`"hello"`) routes to lightweight models (e.g., `gpt-4.1-nano`).
* Submitting complex algorithmic prompts (`"implement a Monte Carlo tree search algorithm from scratch in Python to PlayConnect4"`) dynamically routes to specialized high-reasoning models (e.g., `DeepSeek`).
* Response metadata surfaces the exact model selected by the routing engine for that specific turn.



---

## 3. Formulas, Expressions & Schemas

### Model Router REST API Request Payload (OpenAI-Compatible Chat Completion)

The Model Router exposes an OpenAI-compliant Chat Completion / Responses API endpoint. Client applications specify the router deployment name as the `model` parameter:

```json
{
  "model": "model-router-prod",
  "messages": [
    {
      "role": "system",
      "content": "You are an enterprise software engineering assistant."
    },
    {
      "role": "user",
      "content": "Implement a Monte Carlo tree search algorithm from scratch in Python to PlayConnect4."
    }
  ],
  "temperature": 0.2,
  "max_tokens": 4096
}

```

### Model Router Response Payload with Resolution Attribution

The response payload returns standard completion data and indicates the underlying model chosen in the `model` attribution property:

```json
{
  "id": "chatcmpl-router-99201a88",
  "object": "chat.completion",
  "created": 1794235800,
  "model": "DeepSeek-V3",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Here is a complete, self-contained implementation of the Monte Carlo Tree Search (MCTS) algorithm designed to play Connect4 in Python:\n\n```python\nimport math\nimport random\nimport copy\n\nclass Connect4Board:\n    # Board state and transition mechanics\n    pass\n```"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 42,
    "completion_tokens": 1280,
    "total_tokens": 1322
  }
}

```

### Power Fx: Dynamic Dispatch Verification & Attribution in Copilot Studio

When invoking a deployed Model Router custom connector from a Copilot Studio topic, capture and inspect the underlying model selected:

```powerfx
// Step 1: Dispatch conversational payload to the Model Router deployment
Set(Topic.RouterResponse,
    FoundryModelRouterConnector.CreateChatCompletion({
        model: "model-router-prod",
        messages: Table(
            { role: "system", content: "You are a customer service assistant." },
            { role: "user", content: Topic.UserInquiry }
        ),
        temperature: 0.0
    })
);

// Step 2: Extract generated completion text
Set(Topic.FinalOutputText, First(Topic.RouterResponse.choices).message.content);

// Step 3: Extract and log the underlying model selected by the router for audit telemetry
Set(Topic.ResolvedModelName, Topic.RouterResponse.model);

// Step 4: Emit diagnostic telemetry to Azure Application Insights
Trace("ModelRouterResolutionTelemetry", TraceSeverity.Information, {
    UserQueryLength: Len(Topic.UserInquiry),
    ResolvedModel: Topic.ResolvedModelName,
    TotalTokensBilled: Topic.RouterResponse.usage.total_tokens
});

```

---

## 4. Best Practices & Design Patterns

* **Prompt-Cost Arbitrage Pattern:**
* Deploy the Model Router in front of multi-turn conversational agents with varied user inputs. Let the router offload greetings, entity clarifications, and binary confirmations to nano/mini models while reserving premium reasoning models for complex, domain-heavy requests.


* **Centralized Guardrail Governance:**
* Apply Azure AI Content Safety filters and TPM rate limits directly at the **Model Router deployment level**. Avoid configuring redundant, fragmented content filters on individual base models.


* **Model Subset Curation:**
* Instead of enabling all supported models across multiple vendors, configure a curated **subset** (e.g., pairing one fast/cheap model with one deep reasoning model). This simplifies billing predictability, context length planning, and enterprise vendor compliance.


* **Auditing and Cost Attribution:**
* Always log the `response.model` attribute in client applications or Copilot Studio topics. Because Model Router bills based on the rates of the underlying model that serviced the request, logging the resolved model is required for reconciling Azure Cost Management invoices.


* **Claude Deployment Staging:**
* When planning a model subset that includes Anthropic Claude, verify that Claude is already deployed and active in the project. Attempting to save a router configuration referencing an undeployed Claude model will halt deployment with a configuration error.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **The Claude Pre-Deployment Blocker:**
* While OpenAI, DeepSeek, and other open-weight models in the router pool do not require standalone deployments, **Claude requires a pre-existing, active deployment**. Saving a custom router configuration containing Claude without this prerequisite fails validation immediately.


* **Context Window Truncation Trap:**
* Do not assume underlying models support their maximum native context limits (e.g., 128K or 1M tokens) through the Model Router. Context windows through the router may be restricted to a lower ceiling to standardize prompt evaluation across the pool. Long documents must be pre-chunked or summarized before ingestion.


* **Data Residency Compliance Trade-Off:**
* Selecting **Global Standard** allows prompts to be processed dynamically outside the tenant’s home geography. For tenants subject to GDPR, HIPAA, or strict data sovereignty, select **Data Zone Standard** to ensure data processing remains confined to the designated regional boundary.


* **Preview Model Volatility:**
* Underlying models within the router pool may operate in **Preview**. Model version updates, availability fluctuations, or deprecations can alter routing behavior over time.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Model Router Deployment Requirements:**
* *Question:* "You are configuring a Model Router deployment in Microsoft Foundry with a subset containing GPT-4.1-nano, DeepSeek, and Anthropic Claude. Which model requires a pre-existing, standalone deployment before the router can be created?"
* *Correct Answer:* **Anthropic Claude** (OpenAI and open-source models do not require separate deployments).


* **Exam Trap — Routing Modes:**
* **Quality:** Optimizes purely for the best possible reasoning and completion accuracy, ignoring cost differences.
* **Cost:** Optimizes for lower token pricing within an acceptable quality margin.
* **Balanced (Default):** Establishes an operational compromise between quality and expense.


* **Exam Trap — Global Standard vs. Data Zone Standard:**
* *Global Standard:* Prompt inferencing executes anywhere across Microsoft's worldwide data center infrastructure.
* *Data Zone Standard:* Restricts dynamic inferencing to Microsoft-specified data zone boundaries (e.g., EU) while retaining storage at rest in the resource's Azure geography.


* **Exam Trap — Context Window Expectations:**
* *True or False:* "A model accessed via the Model Router always supports the exact same context window size as when deployed independently."
* *Answer:* **False**. Context window limits for models accessed through the Model Router may be lower than their standalone counterparts.


* **Single Unified Interface:**
* The primary purpose of the Model Router is to provide a **single endpoint and connection** that dynamically dispatches traffic across multiple LLMs/SLMs based on prompt complexity, without requiring custom client-side triage logic.