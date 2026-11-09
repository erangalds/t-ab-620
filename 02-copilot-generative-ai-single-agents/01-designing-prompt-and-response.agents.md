## 1. Core Architecture & Concepts

* **Generative AI Orchestration Paradigm:**
* **Dynamic Orchestration Engine:** Unlike classic orchestration—which relies on deterministic dialog trees and static `User says a phrase` triggers—generative orchestration uses the underlying Large Language Model (LLM) to dynamically plan, route, and synthesize conversational turns.
* **Instruction & Context-Driven Execution:** The agent evaluates user utterances against high-level agent instructions, tool descriptions, and connected knowledge sources to autonomously determine the execution plan (e.g., selecting relevant data chunks or prompting the user for missing slot parameters).


* **Knowledge Retrieval & Grounding (Retrieval-Augmented Generation / RAG):**
* **Knowledge Grounding:** Unstructured and semi-structured documents (e.g., `.xlsx`, `.csv`, `.docx`, `.pdf`) uploaded directly to the agent are parsed, chunked, and stored in Dataverse search indexes.
* **Dynamic Slot Filling & Ambiguity Resolution:** When an utterance is ambiguous (e.g., asking for an allowance without specifying a subject), the generative model evaluates whether the entity exists in session context or proactively prompts for identifying keys (e.g., employee name, employee ID, or email address) before completing the query.


* **Knowledge Base Layering & Answering Sources:**
1. **Domain Knowledge Sources (Uploaded Files / Enterprise Stores):** Curated enterprise data uploaded directly to the agent or integrated via connectors (Dataverse, SharePoint, Azure AI Search).
2. **General Knowledge (LLM Parametric Memory):** Foundational world knowledge encoded directly into the base weights during model training (e.g., general geography like "In what state is New York City?").
3. **Web Search / Information from the Web:** Real-time web retrieval via Bing search integration, allowing the agent to answer questions requiring live, dynamic information outside its training cutoff or uploaded files (e.g., current news events).


* **Generative Rationale & Attribution Telemetry:**
* The runtime exposes a **Rationale** inspector in the test canvas, providing transparent telemetry into which specific chunks, files, or external web URLs the orchestrator cited to formulate its final grounded response.



---

## 2. Platform Features & Capabilities

* **Agent Provisioning Routes:**
* **Blank Agent Creation:** Generates an unconfigured agent container defaulting directly to **Generative AI Orchestration**.
* **Natural Language Agent Scaffolding:** Allows makers to specify business intent in natural language (e.g., *"create an agent which shows how much allowance for leave an employee has... uploaded via spreadsheet"*). The platform parses this intent into default metadata, initial instructions, and knowledge hooks.
* **Advanced Configuration Modal (Gear Icon):** Allows developers to explicitly set the agent **Language**, target **Solution**, and **Schema Name** (prefix bound to the solution publisher) prior to deployment.


* **Knowledge Management Hub (`Knowledge` / `Sources`):**
* Located on the left-hand navigation pane and the agent overview dashboard.
* **Add Knowledge:** Allows importing data from files, websites, Dataverse tables, or external enterprise repositories.
* **File Upload Workflow:**
* Click **Select to browse** $\rightarrow$ select file (e.g., `Leave Allowance.xlsx`) $\rightarrow$ select **Add to agent**.
* Upload executes immediately, followed by an asynchronous background indexing phase (status transitions from processing/spinner to **Ready** with a checkmark).




* **Generative AI Settings & Guardrails:**
* Navigation: Top command bar $\rightarrow$ **Settings** $\rightarrow$ **Generative AI**.
* **Orchestration Mode:** Option to select *Generative (dynamic)* vs. *Classic (deterministic)*.
* **General Knowledge Toggle:** Controls whether the agent can answer using the LLM's pre-trained parametric knowledge when no connected knowledge source contains the answer.
* **Use information from the web (Web Search):** Controls whether the agent performs live Bing search grounding for external, real-time public domain data.


* **Test Canvas Diagnostics:**
* **Search Sources Telemetry:** Real-time visual indicator in the test pane displaying active retrieval queries against attached knowledge repositories.
* **Citation & Table Rendering:** The agent automatically synthesizes tabular source data into Markdown tables in the chat response, citing specific reference sources.
* **Session Reset:** Using **New test session** drops all conversational short-term context and slot-filled variables to validate cold queries.



---

## 3. Formulas, Expressions & Schemas

### Agent Provisioning Payload (Generative Mode)

When creating an agent via natural language scaffolding with solution binding, the underlying Dataverse `bot` record configures the generative orchestrator and knowledge settings:

```json
{
  "name": "Employee Leave Allowance Agent",
  "schemaname": "ab100_leaveAllowanceAgent",
  "language": 1033,
  "solutionid": "3367448e-d1a7-40fb-abcb-184045f233cf",
  "configuration": {
    "orchestrationsettings": {
      "orchestrationMode": "Generative",
      "useDynamicChaining": true,
      "allowGeneralKnowledge": false,
      "allowWebSearch": false
    },
    "defaultLanguage": "en-US"
  }
}

```

### Knowledge File Ingestion Metadata Schema

When tabular files (e.g., Excel/CSV) are ingested into the agent's Dataverse Knowledge repository, an indexing record is generated:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "AgentKnowledgeSourceRecord",
  "type": "object",
  "properties": {
    "knowledgeSourceId": { "type": "string", "format": "uuid" },
    "displayName": { "type": "string", "example": "Leave Allowance.xlsx" },
    "sourceType": { 
      "type": "string", 
      "enum": ["File", "SharePoint", "Dataverse", "PublicWeb", "AzureAISearch"] 
    },
    "fileType": { "type": "string", "enum": ["xlsx", "csv", "docx", "pdf"] },
    "indexingStatus": { 
      "type": "string", 
      "enum": ["Uploading", "Processing", "Ready", "Failed"] 
    },
    "lastIndexedDateTime": { "type": "string", "format": "date-time" }
  },
  "required": ["displayName", "sourceType", "indexingStatus"]
}

```

### Target Data Model in Spreadsheet Knowledge Source

The unstructured/semi-structured tabular schema referenced during ground-truth retrieval:

| EmployeeID | EmployeeName | VacationDays | SickLeaveDays | Department |
| --- | --- | --- | --- | --- |
| `EMP001` | Philip | `30` | `5` | Executive |
| `EMP002` | Jane | `34` | `5` | Engineering |
| `EMP003` | Maria | `35` | `5` | Human Resources |
| `EMP004` | Susan | `29` | `4` | Operations |

### Generative Filter Policy (Power Fx Evaluation Equivalent)

The internal filtering rule enforced when restricting responses strictly to uploaded knowledge:

```powerfx
// Evaluates whether the generated response is strictly grounded in designated source files
If(
    And(
        AgentSettings.AllowGeneralKnowledge = false,
        AgentSettings.AllowWebSearch = false
    ),
    Filter(
        KnowledgeSources, 
        SourceType = "File" And IndexingStatus = "Ready"
    ),
    KnowledgeSources
)

```

---

## 4. Best Practices & Design Patterns

* **Decoupling Data from Logic (Dynamic Knowledge vs. Hardcoded Topics):**
* Avoid maintaining hardcoded values inside condition branches or switch statements. Storing changing operational data (e.g., leave allocations, product pricing, tier benefits) in external knowledge files or Dataverse tables allows data updates without modifying or re-publishing conversational topics.


* **Knowledge Scoping & Grounding Guardrails:**
* For enterprise and compliance-focused copilots, **disable both "General Knowledge" and "Use information from the web"**. This eliminates the risk of LLM hallucinations, prevents generic internet answers, and restricts the agent's responses exclusively to verified internal company sources.


* **Explicit Agent Persona Instructions:**
* Write clear, declarative instructions defining how the agent should handle ambiguity (e.g., *"When an employee allowance is requested without a specific name or ID, always prompt the user to provide their Employee ID or Full Name before retrieving data"*).


* **Telemetry & Rationale Auditing:**
* Regularly inspect the **Rationale** trace during testing to ensure the model extracts information from the intended authoritative document and does not blend data with extraneous public sources.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Asynchronous Indexing Latency:**
* Uploading a knowledge file is not instantaneous. While the initial upload completes in seconds, data extraction, chunking, and Dataverse search index synchronization take several minutes. Queries executed before status reaches **Ready** will return fallback or null responses.


* **Stale Conversational Context Trap:**
* Testing subsequent queries in the same test window causes the LLM to retain conversational context from prior turns. If the agent previously resolved "Jane", asking a general follow-up may evaluate using Jane's context unless **New test session** is explicitly clicked.


* **Global Web Search Latency & Variance:**
* Toggling *Use information from the web* requires saving settings and can take several minutes to propagate through tenant routing layers. Responses grounded in web searches also exhibit higher response latency and non-deterministic variations compared to local file searches.


* **Dataverse Search Prerequisite:**
* File-based knowledge ingestion relies directly on Dataverse Search. If Dataverse Search is disabled in the Power Platform Admin Center for that specific environment, file uploads and document grounding will fail to index.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Classic vs. Generative Triggering:**
* *Classic Orchestration:* Requires explicit trigger phrases (5–10 phrases) mapped directly to specific conversational topics.
* *Generative AI Orchestration:* Operates without phrase-based triggers; the agent autonomously selects actions, tools, and knowledge sources based on intent evaluation and system instructions.


* **Exam Trap — General Knowledge vs. Web Search:**
* **General Knowledge:** The internal parametric knowledge of the LLM learned during training. Useful for static facts, coding syntax, or definitions, but possesses a fixed training cutoff and no real-time awareness.
* **Use Information from the Web:** Uses Grounding with Bing Search to query public internet sites at runtime. Required for current news, live stock data, or dynamic public facts.


* **Exam Trap — Restricting Hallucinations to Enterprise Data:**
* If an exam scenario asks how to ensure an internal HR agent **only** answers questions based on company policy documents and never provides external or generic answers, the required action is: **Turn off General Knowledge AND turn off Use information from the web** in Generative AI settings.


* **Exam Trap — Dynamic Clarification Prompting:**
* In Generative mode, if required information is missing from an utterance to satisfy a knowledge query, the LLM will automatically formulate a clarifying prompt (e.g., asking for an ID or name) without requiring an explicit *Ask a question* node.