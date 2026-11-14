## 1. Core Architecture & Concepts

* **Data Grounding in Generative Architectures:**
* Grounding acts as the runtime boundary anchoring generative models to enterprise truth.
* Governed by the **"Garbage In, Garbage Out" (GIGO)** principle: LLM response fidelity is directly constrained by the syntactic and semantic quality of ingested data.


* **Data Typology & Structural Organization:**
* **Structured Data:** Tabular schemas (relational databases, Dataverse tables, spreadsheets) where columns represent discrete, strongly typed attributes across consistent entities (e.g., dedicated `Condition` columns vs. numerical `WindSpeed` metrics).
* **Unstructured Data:** Freeform narrative documents (Microsoft Word `.docx`, PowerPoint `.pptx`, PDFs) lacking fixed database schemas.
* **Semi-Structured Document Formatting:** Using semantic markup styles (e.g., Word **Heading 1**, **Heading 2**) imposes structural hierarchies, assisting the ingestion parser during semantic chunking and indexation.


* **Core Dimensions of Data Quality:**
* **Consistency:** Standardized casing, terminology, and categorizations (e.g., deduplicating near-synonyms like "snow" vs. "snow showers").
* **Accuracy:** Factually verified source data; vital when running multi-table joins or semantic correlations across disparate enterprise data sets.
* **Precision:** Matching numeric, temporal, or spatial resolution to business needs. Over-precise values introduce noise, whereas under-precise data breaks operational lookups (e.g., unique telephone numbers or transactional timestamps).
* **Validity:** Conformance to programmatic constraints and domain rules (e.g., regex formats for email addresses containing `@`, strict non-negative constraints for purchase quantities, or bounds such as UV indices from $0$ to $9$).
* **Completeness & Granularity:**
* *Tolerance Variation:* High-frequency time-series streams (IoT sensors reporting every 10 seconds) tolerate packet loss and missing rows without impacting macro analysis.
* *Zero-Tolerance Data:* Mission-critical domains (e.g., clinical patient allergy registries) demand 100% completeness; omissions lead to catastrophic hallucinations or dangerous decisions.
* *Granularity Adjustment:* Rolling up high-frequency, incomplete raw records into two-hour or daily statistical aggregations (averages, counts) stabilizes retrieval and mitigates token bloat.


* **Timeliness & Availability:** Data freshness prevents an agent from surfacing obsolete facts; multi-system ingestion requires reliable endpoints with predictable uptime.
* **Non-Duplication:** Redundant records skew semantic ranking weights in vector/hybrid search and create non-deterministic output when conflicting versions coexist.


* **Atomic Decomposition of Compound Attributes:**
* Multi-attribute, unstructured text fields (e.g., `Description` containing `"large blue t-shirt"`) must be decomposed into atomic, single-responsibility dimensions (`Size`, `Color`, `ProductType`) to support deterministic filtering and precise grounding.


* **Data Representativeness & Bias Mitigation (Foundry Vision / Multimodal Analogy):**
* Training data and grounding sets must represent true operational distributions rather than spurious environmental correlations (e.g., training a Custom Vision model where apples appear exclusively on plates and oranges appear exclusively on trees causes the classifier to correlate background context rather than target fruit morphology).


* **Cross-System AI Data Sharing & RBAC:**
* Enterprise architectures favor shared, centralized grounding repositories across Copilot Studio, Microsoft 365 Copilot, and external AI agents over fragmented data silos.
* Access boundaries must be enforced via **Role-Based Access Control (RBAC)**, connection credentials, and Microsoft Entra ID to ensure agents only expose data permitted for a given user context.



---

## 2. Platform Features & Capabilities

* **Knowledge Ingestion & File Location Constraints:**
* **Microsoft OneDrive / SharePoint Online:** Primary unstructured file repository accessible via Microsoft 365 (`m365.cloud.microsoft`), housing `.docx`, `.pptx`, `.xlsx`, and `.pdf` files.
* **Dataverse:** The native relational and search backbone for Copilot Studio, automatically handling storage, vectorization, and metadata tagging.
* **Azure AI Search:** Dedicated external indexing platform used to host custom enterprise indexes, supporting semantic ranking, vector search, and complex multi-modal knowledge.


* **Fabric & Azure SQL Storage Boundaries:**
* Copilot Studio native knowledge connectors interface directly with Azure SQL Database (using read-only endpoints), Dataverse, and SharePoint.
* Direct ingestion from newly introduced database tiers (such as Microsoft Fabric SQL Database) may not be natively exposed as a standard one-click knowledge source in earlier release waves, requiring automated pipeline synchronization (e.g., via Power Automate or Azure Data Factory) to stage data into supported storage (Dataverse or Azure AI Search).


* **Azure AI Search Security Configuration:**
* Connection requires:
* Search Service HTTPS Endpoint URL (e.g., `https://<service-name>.search.windows.net`).
* Index Name.
* Authentication Credentials: API Admin/Query Key, Client Secret/Certificate, or Microsoft Entra ID Integrated Managed Identity.




* **Unstructured Data Ingestion Limits:**
* **Microsoft Copilot Studio:** Ingestion bounded by tenant Dataverse storage capacity, file size ceilings (up to 512 MB per file), and a limit of 500 knowledge objects (up to 25 SharePoint/Web URLs in generative mode). Standalone raw image files (`.png`, `.jpg`) are unsupported for direct document grounding unless embedded inside PDFs.
* **Microsoft 365 Copilot Agent Constraints:** Strict ceiling of **up to 20 knowledge sources / embedded files**.
* *Consolidation Pattern:* To bypass the 20-file cap, store hundreds of files inside a single SharePoint folder/document library and link that **SharePoint folder URL as one single knowledge source**.





---

## 3. Formulas, Expressions & Schemas

### Data Cleansing & Granularity Rollup (Power Fx Pattern)

When aggregating high-frequency sensor or weather readings into summary rows before committing to Dataverse:

```powerfx
// Power Fx aggregation: Computes 2-hour rolling averages from raw readings table
ClearCollect(
    AggregatedTelemetry,
    AddColumns(
        GroupBy(RawReadings, "ReadingIntervalBlock", "GroupedIntervals"),
        "AverageTemperature", Round(Average(GroupedIntervals, TemperatureF), 1),
        "AverageApparentTemp", Round(Average(GroupedIntervals, ApparentTemperatureF), 1),
        "SampleCount", CountRows(GroupedIntervals)
    )
);

// Enforces business validity rules on incoming ground data rows
If(
    Or(
        IsBlank(Topic.IncomingEmail) || Not(IsMatch(Topic.IncomingEmail, "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")),
        Topic.PurchaseQuantity <= 0,
        Topic.UVIndex < 0 || Topic.UVIndex > 9
    ),
    Set(Topic.IsValidDataRow, false),
    Set(Topic.IsValidDataRow, true)
)

```

### Atomic Decomposition Schema (JSON Data Preprocessing)

Transformation schema decomposing compound descriptions into atomic entity properties:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "ProductAttributeDecomposition",
  "type": "object",
  "properties": {
    "rawDescription": {
      "type": "string",
      "example": "large blue t-shirt"
    },
    "decomposedAttributes": {
      "type": "object",
      "properties": {
        "size": {
          "type": "string",
          "enum": ["Small", "Medium", "Large", "X-Large"],
          "example": "Large"
        },
        "color": {
          "type": "string",
          "example": "Blue"
        },
        "itemType": {
          "type": "string",
          "example": "T-Shirt"
        }
      },
      "required": ["size", "color", "itemType"]
    }
  }
}

```

### Azure AI Search Index Schema for Normalized Grounding

Declarative JSON payload for an Azure AI Search index enforcing clean metadata fields and semantic configuration:

```json
{
  "name": "enterprise-policies-index",
  "fields": [
    { "name": "id", "type": "Edm.String", "key": true, "searchable": false },
    { "name": "documentTitle", "type": "Edm.String", "searchable": true, "filterable": true },
    { "name": "headingLevel1", "type": "Edm.String", "searchable": true, "filterable": true },
    { "name": "content", "type": "Edm.String", "searchable": true },
    { "name": "lastUpdated", "type": "Edm.DateTimeOffset", "filterable": true, "sortable": true },
    { "name": "accessRoleRequired", "type": "Edm.String", "filterable": true }
  ],
  "semantic": {
    "configurations": [
      {
        "name": "default-policy-semantic-config",
        "prioritizedFields": {
          "titleField": { "fieldName": "documentTitle" },
          "contentFields": [{ "fieldName": "content" }],
          "keywordsFields": [{ "fieldName": "headingLevel1" }]
        }
      }
    ]
  }
}

```

---

## 4. Best Practices & Design Patterns

* **Semantic Document Formatting (Styling over Plain Text):**
* When preparing `.docx` or `.pptx` files for unstructured RAG, always use built-in Microsoft Word heading styles (**Heading 1**, **Heading 2**). Copilot Studio chunking heuristics use heading tags as structural boundaries, preserving hierarchical context across extracted text chunks.


* **Controlled Vocabulary & Pre-Processing Categorization:**
* Run automated data-cleaning scripts prior to ingestion to merge fragmented categories (e.g., standardizing "snow", "snow showers", and "light snow" into a canonical category like "Snowfall"). This prevents the RAG pipeline from dispersing vector weights across multiple redundant terms.


* **Mitigating "Plate vs. Tree" Contextual Bias:**
* Validate that image and text training/grounding datasets decouple the target entity from accidental background attributes. Audit training data to confirm labels represent the object itself, not correlated peripheral markers.


* **SharePoint Folder Consolidation Pattern (M365 Copilot Workaround):**
* To circumvent strict file quotas (such as the 20-source limit in M365 Copilot), consolidate documents into a structured SharePoint document library or folder. Add the single root SharePoint URL as a knowledge source rather than uploading individual files.


* **Centralized Knowledge Fabric (Single Source of Truth):**
* Where multiple agents (e.g., an internal Copilot Studio HR bot and a Microsoft 365 Copilot declarative agent) consume identical business policies, point both systems to the same authoritative SharePoint repository or Azure AI Search index rather than maintaining duplicate copies.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Direct File Format Restrictions:**
* Copilot Studio knowledge uploads **strictly reject standalone image formats** (`.png`, `.jpg`, `.jpeg`, `.svg`), raw audio files, and video streams. Image-based text must undergo optical character recognition (OCR) or be embedded inside PDF documents before upload.


* **M365 Copilot vs. Copilot Studio Knowledge Quotas:**
* **Microsoft 365 Copilot Agents:** Constrained to a maximum of **20 knowledge sources / embedded files**.
* **Copilot Studio Standalone Agents:** Supports up to **500 knowledge objects**, bounded by Dataverse capacity and a maximum of 5 distinct source types.


* **Data Cleansing Antipattern: Ignoring Compound Attributes:**
* Storing multi-variable strings in a single unstructured database column prevents precise query filtering. Searching for a "large blue shirt" across a raw string column degrades retrieval precision compared to filtering on structured attributes (`Size = 'Large'`, `Color = 'Blue'`).


* **Unfiltered Duplication Pitfall:**
* Ingesting duplicate documents with minor revisions or conflicting versions causes non-deterministic generative behavior; the agent will alternate between outdated and current facts unpredictably across identical user prompts.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Word Styles & RAG Chunking:**
* *Trap:* Assuming that large text chunks are split purely by fixed character counts.
* *Fact:* Formatting documents with structural styles like **Heading 1** and **Heading 2** provides semantic anchors that help the indexing engine chunk documents accurately.


* **Exam Trap — Bypassing the 20-File Limit in Microsoft 365 Copilot:**
* *Question Scenario:* An organization needs an agent in Microsoft 365 Copilot to reference 80 corporate HR policy documents, but hits the 20-source limit.
* *Correct Solution:* Move all 80 documents into a dedicated SharePoint Online folder or document library, then register the **single SharePoint folder URL** as the knowledge source.


* **Exam Trap — Completeness vs. Granularity Trade-offs:**
* *High-Frequency Sensor Data:* High granularity (every 10 seconds) with occasional dropped packets does *not* require complete remediation; it should be aggregated into summary averages.
* *Critical Enterprise Data:* Transactional registries, security clearances, and patient allergy tables demand 100% completeness and zero tolerance for missing records.


* **Exam Trap — Direct Ingestion of Image Files:**
* You cannot upload raw `.jpg` or `.png` files directly into Copilot Studio as knowledge sources. They must be converted/OCRed into text or compiled inside a `.pdf`.


* **Exam Trap — External AI System Permissions:**
* Pointing multiple AI agents to Azure AI Search requires configuring explicit authentication credentials (API Keys, Service Principals, or Entra ID) and verifying that Azure Role-Based Access Control (**Azure RBAC**) permissions do not leak restricted data across different user roles.