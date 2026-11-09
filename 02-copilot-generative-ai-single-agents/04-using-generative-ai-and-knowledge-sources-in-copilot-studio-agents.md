## 1. Core Architecture & Concepts

* **Hybrid Orchestration Model (Generative Capabilities in Classic Mode):**
* Agents running under **Classic Orchestration** can invoke Generative AI without migrating to the full generative orchestration engine.
* This is achieved through the system topic **Conversational Boosting** or by inserting the **Create generative answers** node into any custom topic canvas (*+ Add node* $\rightarrow$ *Advanced* $\rightarrow$ *Generative answers*).


* **Knowledge Retrieval Engine (RAG Architecture in Dataverse):**
* When knowledge sources (SharePoint sites, uploaded files, Dataverse tables) are connected, content is extracted, chunked, and indexed directly in **Microsoft Dataverse**.
* Querying external sources live on every conversational turn carries excessive latency, rate-limiting risks, and computational expense. Pre-indexing into Dataverse search provides semantic search at low runtime cost.
* Initial Dataverse search configuration and indexing latency spans **5 to 30 minutes** before newly added sources can serve grounded responses.


* **Web Search vs. Enterprise Knowledge Grounding:**
* **Public Web Search:** Uses Bing-indexed public web data for broad queries. Can be toggled on/off to isolate the agent strictly to curated enterprise knowledge.
* **Knowledge Grounding:** Constrains the Large Language Model (LLM) to synthesize answers exclusively from attached enterprise sources (Dataverse, SharePoint, documents, enterprise databases).


* **Identity, Authentication & Authorization Boundaries:**
* **Microsoft Entra ID (formerly Azure AD):** Provides centralized identity and security trimming for SharePoint, OneDrive, Dataverse, and Dynamics 365. The agent inherits or executes queries using the signed-in user's credentials to guarantee that end users cannot view data they are not authorized to see in the source system.
* **Azure AI Search Authentication Models:** External indexing services like Azure AI Search do not rely solely on simple user delegation; they support four enterprise authentication models:
    1. **Access Key** (API / Admin Key).
    2. **Client Certificate Authentication**.
    3. **Service Principal** (Entra ID App Registration).
    4. **Microsoft Entra ID Integrated** (Managed Identity / Integrated User Context).




* **Database Access Architecture (Read-Only Endpoints):**
* For relational databases like **Azure SQL Database**, architects configure a dedicated **read-only endpoint / replica**.
* Routing agent queries to a read-only endpoint prevents heavy generative search queries from degrading performance or deadlocking transaction tables used by primary read-write business applications.


* **Official Source Designation:**
* Setting an attached public website as an **Official Source** updates the agent's internal system prompt instructions, telling the model to trust the source as authoritative without requiring cross-verification or secondary validation.



---

## 2. Platform Features & Capabilities

* **Knowledge Source Provisioning & UI Navigation:**
* Navigation: Copilot Studio top command bar $\rightarrow$ **Knowledge** tab (or **Overview** $\rightarrow$ scroll to **Knowledge** $\rightarrow$ **Add knowledge** / **See all**).
* **Public Web Search Configuration:** Overview $\rightarrow$ Knowledge section $\rightarrow$ toggle **Search public websites / Enable web search**.


* **Public Websites as Knowledge Sources:**
* **Scale Limits by Orchestration Mode:**
* *Classic Orchestration:* Maximum of **4 public website URLs**.
* *Generative AI Orchestration:* Maximum of **25 public website URLs**.


* **Depth Restriction:** URLs can be indexed up to a maximum of **two levels deep** (e.g., `domain.com`, `[domain.com/path1](https://domain.com/path1)`, `[domain.com/path1/path2](https://domain.com/path1/path2)`).
* **Indexing & Public Access Rules:** Must be anonymously accessible (no login/authentication screens) and actively indexed by **Bing**.
* **Description Field:** Required during ingestion to inform the generative planner when and why to target the URL for retrieval.
* **Official Source Toggle:** Access via ellipsis (`...`) on a knowledge card $\rightarrow$ toggle **Official source** to *Yes* (modifies system instructions to prioritize content).


* **Uploaded Documents (Direct File Ingestion):**
* Ingested from local disk, OneDrive, or SharePoint into Dataverse file storage.
* Bounded only by the environment's total **Dataverse file storage quota**.
* Maximum file size: **512 MB per file**.


* **SharePoint & OneDrive Repository Integration:**
* **Scale Limits by Orchestration Mode:**
* *Classic Orchestration:* Maximum of **4 SharePoint URLs**.
* *Generative AI Orchestration:* Maximum of **25 SharePoint URLs**.


* **Catalog Ingestion Quotas (per source):**
* Maximum of **1,000 files**.
* Maximum of **50 folders**.
* Maximum traversal depth of **10 levels of subfolders**.


* **Unstructured Knowledge Ceiling:** Up to **500 knowledge objects** per agent (files, folders, websites, knowledge articles).
* **Indexing State Lifecycle:** Displays **Ready** briefly on upload $\rightarrow$ transitions asynchronously to **In Progress** while chunking $\rightarrow$ returns to **Ready** once indexing completes and responses can be served.


* **Microsoft Dataverse Integration:**
* *Classic Orchestration:* Maximum of **2 Dataverse knowledge sources**, up to **15 tables per knowledge source**.
* *Generative AI Orchestration:* No strict 2-source cap on Dataverse connections.


* **Connected Enterprise Knowledge Sources:**
* Standard: Dynamics 365, Salesforce, ServiceNow, Azure SQL Database.
* Advanced: Snowflake, Databricks / Azure Databricks, Confluence, Oracle Database, SAP OData, Zendesk, Bing Custom Search.
* **Knowledge Source Type Cap:** An agent can connect to a maximum of **5 different types of knowledge sources** simultaneously (e.g., SharePoint counts as 1 type, Dataverse counts as 1 type regardless of table count).



---

## 3. Formulas, Expressions & Schemas

### Generative Answers Node (Bot Component YAML Representation)

When adding generative search capabilities into a topic (such as the default `Conversational boosting` topic or custom topic), the declarative YAML schema evaluates as follows:

```yaml
kind: AdaptiveDialog
beginDialog:
  kind: OnRecognizedIntent
  id: main
  actions:
    - kind: SearchAndGenerateContent
      id: searchAndGenerate_leaveInfo
      userInput: =Topic.UserQuery
      inputs:
        searchPublicWebsites: false
        webSearchSources:
          - https://www.microsoft.com
        sharePointSources:
          - https://tenant.sharepoint.com/sites/HumanResources
        dataverseSources:
          - entityLogicalName: cr123_leaveallowance
      variable: Topic.GenerativeAnswerOutput

```

### Power Fx: Dynamic Knowledge Source Query & Variable Binding

Passing dynamic context and evaluating generative search payloads programmatically:

```powerfx
// Evaluates user query and maps directly to the generative search input property
Set(Topic.UserQuery, System.Activity.Text);

// Evaluates whether the generated response is blank before triggering manual fallback
If(
    IsBlank(Topic.GenerativeAnswerOutput.Text),
    Redirect(Fallback),
    Set(Topic.FinalMessage, Topic.GenerativeAnswerOutput.Text)
)

```

### Azure AI Search Knowledge Connector Connection Payload Schema

The JSON structure required when provisioning a secure Azure AI Search connection via a Service Principal:

```json
{
  "connectionParameters": {
    "azureSearchEndpoint": "https://exam-search-service.search.windows.net",
    "authenticationType": "ServicePrincipal",
    "tenantId": "00000000-0000-0000-0000-000000000000",
    "clientId": "11111111-1111-1111-1111-111111111111",
    "clientSecret": "[SECURE_VAULT_REFERENCE]",
    "indexName": "enterprise-knowledge-index"
  }
}

```

---

## 4. Best Practices & Design Patterns

* **Mitigating Indirect Prompt Injection via Untrusted Public Sites:**
* Public websites, user-editable community forums, and social networks are prone to **prompt manipulation / indirect prompt injection** (hidden instructions embedded in web markup designed to hijack agent persona or exfiltrate data).
* *Best Practice:* Never ingest open social networks or unvetted web forums as knowledge sources. Restrict public website indexing to authoritative, corporate-owned, or vetted regulatory domains.


* **Official Source Verification Protocol:**
* Reserve the **Official source = Yes** setting strictly for verified internal repositories or signed policy documentation. Enabling this instructs the orchestrator to bypass cross-verification logic, which will propagate unverified data if set on third-party domains.


* **Relational Database Decoupling:**
* Always bind Copilot Studio agents to an **Azure SQL read-only replica / endpoint**. This protects online transaction processing (OLTP) workloads from thread exhaustion or latency spikes caused by analytical and semantic search queries.


* **Knowledge Indexing Staging Lifecycle:**
* When configuring new SharePoint sites or Dataverse tables in an enterprise deployment, allow a staging buffer of **30 minutes** before executing acceptance tests. Never run user testing while status indicates **In Progress**.


* **Minimizing Source Type Proliferation:**
* Because agents are bounded by a maximum of **5 different knowledge source types**, consolidate enterprise knowledge (e.g., standardizing unstructured documents into SharePoint rather than splitting across Azure SQL, Zendesk, Salesforce, and file uploads simultaneously).



---

## 5. Technical Limitations, Gotchas & Guardrails

* **File Format & MIME Restrictions:**
* **Supported Formats:** `.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx`, and `.pdf`.
* **Strictly Unsupported Formats:** Standalone images (`.png`, `.jpg`), raw audio (`.mp3`, `.wav`), video (`.mp4`), and executable binaries (`.exe`, `.sh`).
* *Exception:* Images embedded directly inside `.pdf` files are parsed and supported.


* **Sensitivity Label Blacklist:**
* Documents tagged with Microsoft Purview Information Protection (MIP) sensitivity labels of **Confidential** or **Highly Confidential** will **not** be indexed or served by generative answers.
* *The Gotcha:* These files may display a status of **Ready** in the UI, but the model will quietly drop them during search retrieval and fail to return answers.


* **Public URL Deep Linking Limit:**
* URLs must not exceed **2 levels of path depth** past the domain root. Attempting to ingest `[microsoft.com/en-us/solutions/power-platform/copilot](https://microsoft.com/en-us/solutions/power-platform/copilot)` will fail or truncate indexing beyond level 2.


* **Bing Indexing Prerequisite:**
* Public web scraping requires that the site is actively crawled and indexed by **Bing**. Newly launched or intranet-bound public sites blocked by `robots.txt` cannot be indexed as public website knowledge sources.


* **Transient "Ready" State Trap:**
* When a SharePoint or OneDrive source is attached, the UI displays **Ready** for a few seconds before changing to **In Progress**. Testing the agent during this initial window will yield false-negative fallback responses.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — URL Scaling Limits by Orchestration Mode:**
* **Classic Orchestration:** Strictly capped at **4 Public Website URLs** and **4 SharePoint URLs**.
* **Generative AI Orchestration:** Supports up to **25 Public Website URLs** and **25 SharePoint URLs**.


* **Exam Trap — Maximum Source Types vs. Total Sources:**
* You can have up to **500 knowledge objects** (files, folders, sites), but you can connect to a maximum of **5 distinct types of knowledge sources** (e.g., 1. SharePoint + 2. Dataverse + 3. Azure SQL + 4. Public Web + 5. Salesforce). Exceeding 5 types is blocked by the platform.


* **Exam Trap — SharePoint Ingestion Capacity Limits:**
* Remember the exact triad: **1,000 files, 50 folders, 10 subfolder levels** per individual SharePoint/OneDrive knowledge source.


* **Exam Trap — Maximum File Size:**
* The maximum supported file size for direct document uploads is **512 MB per file**, constrained overall only by available Dataverse storage capacity.


* **Exam Trap — Dataverse Table Caps in Classic Mode:**
* Classic agents are restricted to **2 Dataverse sources** and a maximum of **15 tables per source**. Generative orchestration removes this specific source cap.


* **Exam Trap — Azure AI Search Authentication:**
* If a question asks which authentication methods are valid for connecting Azure AI Search as a knowledge source, the four valid options are: **Access Key, Client Certificate, Service Principal (Entra ID), and Entra ID Integrated**.