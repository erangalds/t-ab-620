## 1. Core Architecture & Concepts

* **Evolution of the Copilot Studio Experience (Classic vs. New/Modern Experience):**
* **Classic Experience:** Traditional conversational architecture structured around explicit conversational nodes, trigger-driven topics, deterministic branching paths, tools, flows, and fine-grained state management.
* **New Experience (Modern / GitHub Copilot Harness):** An instruction-driven, unified conversational canvas introduced in 2026, centering on high-level natural language prompts, agent skills, model context protocols (MCP), and orchestrators. In this harness, the navigation pane replaces "Flows" and separate configuration tabs with a streamlined **Agents** and **Workflows** paradigm.
* **Topics Dependency:** Custom topic creation, granular trigger configurations, and explicit visual node branching are native to the **Classic Experience**.


* **Agent Operational Archetypes:**
* **Assistive Agents (Copilots):** Run in a conversational loop with human operators; summarize data, provide next-best-action guidance, formulate recommendations, and present structured alternatives where the human retains final execution authority (Human-in-the-Loop).
* **Autonomous Agents:** Independent background systems capable of executing end-to-end operational decisions without per-transaction intervention (e.g., dynamically projecting stock deficiencies and initiating replenishment purchase orders based on real-time inventory signals).


* **Microsoft Release Channel Taxonomy:**
* **Frontier:** An early-access, gated preview tier reserved for select organizations enrolled in frontier validation programs to test nascent foundational capabilities.
* **Public Preview:** Broadly accessible release phase deployed across environments for testing and evaluation; fully functional but subject to behavioral shifts, operational changes, and excluded from standard production Service Level Agreements (SLAs).
* **General Availability (GA):** Fully supported enterprise-ready release tier backed by production SLAs and standard support.


* **Managed Agents Architecture:**
* Pre-packaged, domain-specific AI micro-architectures distributed as managed enterprise solutions directly within the Copilot Studio catalog.
* Packaged with pre-configured schemas, specialized prompting frameworks, ingestion pipelines, and Dataverse entity linkages.



---

## 2. Platform Features & Capabilities

* **UI Navigation & Interface Toggle:**
* **Switching from New Experience to Classic Experience:**
1. Locate the **`...` (ellipsis / More Options)** control situated in the lower-left navigation bar of the Copilot Studio interface.
2. Select **Open classic experience**.
3. Dismiss/bypass the modal feedback prompt.
4. Validate navigation: Left-hand rail transitions from *Agents / Workflows* to *Home / Agents / Flows / Tools*.




* **Pre-Built / Managed Agent Catalog:**
* **Catalog Enrichment Agent (Preview):**
* *Purpose:* Merchandising and e-commerce inventory lifecycle management.
* *Functionality:* Automates product onboarding, new variant ingestion, schema-driven catalog updates, categorization, and metadata extraction/enrichment (e.g., color, size, material specs) to accelerate go-to-market cycles and search discoverability.
* *Deployment Footprint:* Provisioning pipeline requires approximately **25 minutes** to stage underlying solution dependencies and Dataverse components.


* **Document Processor Agent:**
* *Purpose:* Intelligent Document Processing (IDP).
* *Functionality:* Extracts unstructured and semi-structured text, key-value pairs, and table arrays from digitized document images and PDFs for downstream workflow consumption.


* **Finance in M365 (Variance Analysis):**
* *Purpose:* Financial reconciliation across Microsoft 365 spreadsheet datasets.
* *Functionality:* Reconciles disparate Excel workbooks, dynamically derives mathematical matching/variance rules via generative AI, and outputs discrepancy audit reports.


* **Personalized Shopping Agent:**
* *Purpose:* Contextual consumer sales assistance and customer advisory.
* *Functionality:* Provides natural-language product discovery, real-time comparison, and contextual recommendation delivery.
* *Distribution Targets:* Deployable as an embeddable consumer web widget, integrated within customer-facing digital applications, or embedded in internal line-of-business (LOB) apps used by in-store floor associates.


* **Store Operations Agent:**
* *Purpose:* Retail operations orchestration and frontline execution management.
* *Functionality:* Aggregates operational telemetry, surfaces prioritized store-level alerts, recommends labor allocation and task scheduling, and executes routine store workflows.
* *Integration Footprint:* Connects natively to **Dynamics 365** suites (e.g., Dynamics 365 Commerce / Field Service) and enterprise third-party REST endpoints while enforcing managerial approval gates.





---

## 3. Formulas, Expressions & Schemas

### Managed Agent Installation State Machine Schema

When an administrator triggers the installation of a managed agent (e.g., Catalog Enrichment), Copilot Studio invokes the Power Platform Solution Deployment lifecycle asynchronously:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "ManagedAgentProvisioningPayload",
  "type": "object",
  "properties": {
    "agentInternalName": {
      "type": "string",
      "enum": [
        "msdyn_CatalogEnrichmentAgent",
        "msdyn_DocumentProcessorAgent",
        "msdyn_FinanceVarianceAgent",
        "msdyn_PersonalizedShoppingAgent",
        "msdyn_StoreOperationsAgent"
      ]
    },
    "provisioningState": {
      "type": "string",
      "enum": ["Queued", "InstallingDependencies", "ImportingSolution", "ConfiguringGenerativeOrchestrator", "Ready", "Failed"]
    },
    "estimatedTimeRemainingSeconds": {
      "type": "integer",
      "default": 1500
    },
    "releaseWave": {
      "type": "string",
      "enum": ["Frontier", "Preview", "GeneralAvailability"]
    }
  },
  "required": ["agentInternalName", "provisioningState"]
}

```

### Deterministic Human-in-the-Loop Approval (Power Fx Pattern)

Autonomous agents handling operational tasks (e.g., ordering inventory or critical merchandising adjustments) must enforce deterministic guardrails before invoking transactional APIs:

```powerfx
// Evaluates stock-to-projection delta and gates transactional execution
Set(
    varSuggestedReorderQty, 
    Value(Topic.ProjectedSalesDemand) - Value(Topic.CurrentStockLevel)
);

If(
    varSuggestedReorderQty > 0,
    If(
        // Human-in-the-loop threshold check: Orders exceeding $5,000 require manager approval
        (varSuggestedReorderQty * Topic.UnitCost) > 5000,
        Set(Topic.RequiresManagerApproval, true),
        Set(Topic.RequiresManagerApproval, false)
    ),
    Set(Topic.RequiresManagerApproval, false)
);

```

### Structured Output Extraction Schema for Catalog Enrichment

```json
{
  "type": "object",
  "description": "Extracted and enriched catalog item attributes generated by Catalog Enrichment Agent",
  "properties": {
    "sku": { "type": "string" },
    "displayName": { "type": "string" },
    "category": { "type": "string" },
    "attributes": {
      "type": "object",
      "properties": {
        "color": { "type": "string" },
        "size": { "type": "string" },
        "material": { "type": "string" }
      },
      "required": ["color", "size"]
    },
    "confidenceScore": {
      "type": "number",
      "minimum": 0.0,
      "maximum": 1.0
    }
  },
  "required": ["sku", "displayName", "category", "attributes"]
}

```

---

## 4. Best Practices & Design Patterns

* **Human-in-the-Loop (HITL) Governance:**
* When architecting decision-support agents, design the conversation pattern so generative AI handles analysis, synthesis, and recommendation generation, but routes transactional side-effects (e.g., approving budgets, issuing refunds, modifying core catalog tables) through mandatory human confirmation steps or Power Automate approval nodes.


* **Hybrid Deterministic/Generative Architecture:**
* Use **Workflows / Agent Flows** for deterministic tasks (compliance auditing, financial calculation, relational database writes) where rule adherence must be 100% predictable.
* Route unstructured, ambiguous, and natural-language tasks (sentiment analysis, intent classification, catalog attribute parsing) to **Generative Agents / Topics**.


* **Managed Agent Environment Isolation:**
* Because managed agents (e.g., Store Operations, Catalog Enrichment) install managed solution layers and Dataverse tables requiring up to 25 minutes to provision, install them in dedicated sandbox/development environments first to assess tenant capacity impact and dependency chains before promoting upstream via ALM pipelines.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Interface Parity Discrepancy:** The modern (2026 / GitHub harness) interface does not support standard custom conversational topic authoring directly in the same manner as the Classic studio. Developers authoring deterministic dialog trees must toggle back to the **Classic Experience** via the left-hand rail navigation.
* **Managed Agent Provisioning Latency:** Provisioning pre-built agents is not instantaneous. Managed solution dependencies across Dataverse, AI models, and orchestration pipelines take up to **25 minutes** to fully deploy per environment.
* **Preview / Frontier Release Guardrails:**
* Features flagged as **Preview** are not supported for high-governance production workloads and lack full regional/compliance commitments.
* Features flagged as **Frontier** represent bleeding-edge releases accessible strictly via organizational opt-in programs, subject to breaking schema modifications.


* **No Direct Migration Path:** Agents instantiated under the new agent experience cannot be natively down-converted into classic topic-based bots, and vice-versa; architectural selection must occur prior to foundational asset building.

---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Interface Navigation:** If an exam question asks where to configure explicit conversational **Topics**, trigger phrases, and visual canvas logic, and the UI displays *Agents & Workflows* without topic authoring options, the developer must open the **Classic Experience** via the bottom-left `...` menu.
* **Exam Trap — Assistive vs. Autonomous:**
* *Assistive (Copilot):* Augments the human operator, provides contextual analytics, draft responses, and leaves execution approval to the user.
* *Autonomous:* Evaluates telemetry against logic models and initiates external actions/transactions independently (e.g., auto-reordering stock when below safety levels).


* **Managed Agents Lifecycle:** Pre-built managed agents deploy as managed Dataverse solutions; configuring them requires appropriate Power Platform environment Maker/Admin privileges, available tenant storage capacity, and up to 25 minutes of solution import runtime.
* **Ecosystem Terminology Mapping:**
* *PVA (Power Virtual Agents):* Legacy name for the platform.
* *Copilot Studio (Classic):* Canvas-centric topic and flow development environment.
* *Copilot Studio (Modern / New Experience):* Instruction-, skill-, and workflow-centric agent harness.