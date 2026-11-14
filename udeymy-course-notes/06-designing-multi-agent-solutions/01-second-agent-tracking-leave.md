## 1. Core Architecture & Concepts

* **Multi-Agent Solution Patterns (Separation of Concerns):**
* Instead of overloading a single monolithic agent with disparate domains, responsibilities are segmented into specialized micro-agents:
* *Allowance Checker Agent (Agent C):* Tracks global/yearly entitlements and balance baselines from static/annual knowledge sources.
* *Leave Tracker / Request Agent (Agent D):* Queries dynamic, operational transaction records (historical, submitted, or in-flight leave bookings).


* This modular pattern simplifies prompt engineering, minimizes hallucination risk across overlapping instruction sets, avoids exceeding per-agent tool/knowledge performance limits, and allows independent lifecycle management (ALM) for each capability.


* **Knowledge Ingestion Models: Static Upload vs. Cloud Storage Reference:**
* *Direct File Upload (Local Disk):* Uploads a snapshot directly to Dataverse file storage. This is static; content updates require manual re-uploading and re-indexing.
* *Cloud Storage Reference (OneDrive for Business / SharePoint):* Points to a living file in Microsoft 365 cloud storage. This establishes a dynamic source where the file can be updated directly via Office 365 apps or Power Platform connectors, serving as a read/write data store.


* **Structured Tabular Data Binding vs. Unstructured Cell Ranges:**
* Ingesting an entire unstructured spreadsheet range forces the RAG vector indexing engine to chunk data arbitrarily.
* Converting a spreadsheet range into an explicit **Excel Table (`ListObject`)** with defined headers provides structural boundaries. This preserves schema fidelity (column name to value mappings), enabling the semantic indexer to accurately resolve multi-row transactions per entity.


* **Deterministic Scope Pruning in System Prompts:**
* AI meta-prompting defaults often attempt full end-to-end processing (e.g., automatically inserting instructions to "calculate remaining balances" or "connect to an HR system").
* Architects must prune unwanted operational instructions to strictly bound the agent's domain, delegating cross-agent calculations (e.g., subtracting requested leave from annual allowances) to an orchestrating supervisor or separate workflow.



---

## 2. Platform Features & Capabilities

* **Excel Source Preparation (Microsoft 365 OneDrive for Business):**
* *Path:* Microsoft 365 App Launcher $\rightarrow$ **OneDrive** $\rightarrow$ **New / Upload** $\rightarrow$ **Excel workbook**.
* *Schema Definition:* Columns defined as `Name`, `Date From`, `Date To`, `Leave Type`, and `Vacation Days`.
* *Table Conversion:* Insert $\rightarrow$ **Table** $\rightarrow$ Check **My table has headers**. Table named `leave requests`; worksheet tab renamed to `leave requests`.


* **Adding Cloud-Hosted Knowledge to Copilot Studio:**
* *Path:* Agent Overview $\rightarrow$ **Knowledge** $\rightarrow$ Click **+ Add knowledge**.
* *Selection:* Choose **OneDrive** (or SharePoint) $\rightarrow$ **Browse items** $\rightarrow$ Select `leave requests.xlsx` $\rightarrow$ Click **Confirm selection** $\rightarrow$ Click **Add to agent**.


* **System Instructions Tuning:**
* *Path:* Agent Overview $\rightarrow$ **Instructions** $\rightarrow$ **Edit**.
* *Instruction Refactoring:*
* Replace generic generated placeholders (e.g., *"use the HR system or database connector"*) with specific source references (*"use the leave requests spreadsheet to fetch leave records for the user"*).
* Explicitly instruct aggregation: *"Sum up the number of days for each category"*.
* Explicitly exclude out-of-scope tasks: Remove any instructions attempting to calculate remaining balances.




* **General Knowledge / Public Web Search Deactivation:**
* *Path:* **Settings** $\rightarrow$ **Generative AI** $\rightarrow$ Scroll to **Knowledge sources** / **Web search**.
* *Configuration:* Uncheck/toggle off **Search public websites / Use general knowledge** and click **Save**. This prevents the orchestrator from grounding answers on public internet data or general LLM world knowledge.


* **Testing & Authentication Runtime Prompts:**
* *Draft Execution:* Testing in the canvas can occur immediately without publishing the agent.
* *Consent Dialogue:* When querying a personal OneDrive for Business source during testing, the runtime prompts: *"Allow the agent to run with these credentials"*. The maker/user must grant delegated OAuth consent to execute the Microsoft Graph / OneDrive connector call against the target account.



---

## 3. Formulas, Expressions & Schemas

### Excel Tabular Schema (`leave requests.xlsx`)

Structured table configuration stored in OneDrive for Business:

```markdown
| Name   | Date From   | Date To     | Leave Type | Vacation Days |
|--------|-------------|-------------|------------|---------------|
| Philip | 01/01/2029  | 02/01/2029  | Vacation   | 2             |
| Philip | 15/01/2029  | 17/01/2029  | Vacation   | 3             |
| Philip | 25/01/2029  | 26/01/2029  | Sick Leave | 2             |
| Jane   | 01/01/2029  | 07/01/2029  | Vacation   | 5             |

```

### Agent System Instructions Template

The refined system prompt governing retrieval and mathematical aggregation:

```markdown
You are an specialized Leave Tracking Agent. Your sole purpose is to retrieve and summarize historical and requested leave records.

### Instructions:
1. Retrieval: Use the "leave requests" spreadsheet to fetch leave records matching the specified employee name.
2. Aggregation: 
   - Sum up the total number of days requested for each distinct category (e.g., Vacation, Sick Leave).
   - If an employee has multiple entries for the same category, add the days together and present the breakdown by date range along with the total.
3. Negative Matching: If no records match the requested employee name, explicitly state that there are no leave requests listed for that employee. Do not assume zero or infer missing values.
4. Scope Constraints: 
   - Do NOT attempt to calculate remaining leave balances.
   - Do NOT access or reference external HR databases or general knowledge.

```

### Evaluation Logic (Power Fx Equivalent)

When an agent or Power Automate action processes records from this tabular data source:

```powerfx
// Filter rows matching employee entity and sum vacation days
Set(
    varTotalVacationRequested,
    Sum(
        Filter(
            'leave requests',
            Lower(Name) = Lower(varExtractedEmployeeName) && Lower('Leave Type') = "vacation"
        ),
        'Vacation Days'
    )
);

// Fallback handling when zero rows return
If(
    IsEmpty(Filter('leave requests', Lower(Name) = Lower(varExtractedEmployeeName))),
    Set(varResponseText, "There is no leave request for " & varExtractedEmployeeName & " listed in the employee spreadsheet.")
);

```

---

## 4. Best Practices & Design Patterns

* **Micro-Agent Specialization (Single Responsibility Principle):**
* Keep individual agent scopes narrow. Agent C calculates baseline allowances; Agent D aggregates historical/requested transactional entries. A third supervisor/parent agent will coordinate between them to perform cross-domain reconciliation (e.g., Net Balance = Allowance - Taken).


* **Format Data as Excel Tables (`Ctrl + T`):**
* Never leave raw tabular data as an unformatted range (`A1:E10`) when using spreadsheets as knowledge sources. Always wrap them in an Excel Table (`ListObject`). Tables inject structured column-name metadata into the semantic search pipeline, preventing row cross-talk and attribute misalignments during semantic retrieval.


* **Explicit Fallback Directives for Missing Records:**
* Instruct the agent explicitly on how to respond when an entity search returns empty (e.g., *"If an employee has no rows, state that no leave requests are listed"*). Without this directive, the LLM may hallucinate that an unlisted user has zero requests, or assume a general baseline.


* **Knowledge Isolation:**
* Always deactivate public web search and general knowledge toggles for transactional, internal-only HR agents. Grounding must remain 100% deterministic against authenticated enterprise files.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Cloud Storage Indexing Latency ("Ready" vs. Usable):**
* Attaching a cloud-hosted file from OneDrive or SharePoint does not guarantee instantaneous indexing.
* *The Trap:* The knowledge UI may show the file status as **Ready**, but it internally flips to **In Progress** while parsing chunks and generating vector embeddings. Querying the agent immediately after attachment will result in fallback responses or an inability to find rows until indexing completes.


* **OneDrive/SharePoint Refresh Intervals:**
* Unstructured/file knowledge sources synchronized from OneDrive for Business and SharePoint refresh every **4 to 6 hours**.
* *Gotcha:* Direct edits made in Excel Online do not reflect in Copilot Studio's generative retrieval instantly unless the index is manually refreshed or synchronization completes.


* **OneDrive Personal vs. Business Account Context:**
* Connecting to personal OneDrive accounts causes enterprise governance and credential delegation failures. Enterprise multi-agent architectures require **OneDrive for Business** backed by Microsoft Entra ID.


* **Mathematical Aggregation Ceilings in RAG:**
* While modern LLM generative orchestrators can sum a few rows (e.g., $2 + 3 = 5$ days for Philip across three rows), large datasets (hundreds of rows) cannot be reliably summed via unstructured semantic retrieval chunking. High-volume aggregation requires Power Automate, the Code Interpreter tool, or direct Dataverse/SQL aggregation connectors.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Draft Testing vs. Publishing State:**
* *Exam Fact:* You **can** test an agent with newly attached knowledge sources and modified instructions in the test canvas **without publishing**. Publishing is strictly required only to push changes to live channels (Teams, web chat, autonomous triggers).


* **Credential Delegation for Cloud Knowledge:**
* *Exam Trap:* Assuming OneDrive knowledge sources automatically run under a shared background system account.
* *Fact:* When interacting with OneDrive for Business and SharePoint knowledge sources, the user/tester must authenticate and grant permissions under their **delegated user credentials** (Microsoft Entra ID).


* **Direct Upload vs. OneDrive Knowledge Source Capabilities:**
* *Exam Distinction:*
* *Direct Upload:* Read-only static snapshot stored in Dataverse file storage. Cannot be modified externally by other workflows.
* *OneDrive for Business / SharePoint:* Cloud-hosted live document. Supports read/write data operations via external cloud flows, apps, or connectors, enabling downstream dynamic updates.




* **Topic Hygiene in Task-Specific Agents:**
* *Exam Scenario:* An agent designed strictly for data retrieval incorrectly triggers the conversational "Greeting" or "Goodbye" system topics when users provide input.
* *Remediation:* Unneeded system or default topics can and should be individually **disabled** or **deleted** to prevent intent-matching conflicts with generative orchestration.