## 1. Core Architecture & Concepts

* **System Prompt / Instruction Harness Architecture:**
* In Generative AI Orchestration, the **Instructions** block acts as the master system prompt and behavioral meta-harness for the underlying Large Language Model (LLM).
* Unlike topic-based deterministic branching, instructions guide the autonomous planner on intent interpretation, contextual entity resolution, tool selection, data boundary enforcement, and tone synthesis.


* **Implicit Context vs. Grounded Entity Disambiguation:**
* **Identity Bias / Implicit Context Assumption:** By default, generative models often attempt to resolve ambiguous user queries using implicit environmental context—such as the caller's login token (`System.User.DisplayName` or `System.User.Email`)—leading to unintended query execution (e.g., assuming third-person queries like "that employee" refer to the authenticated caller).
* **Disambiguation Guardrails:** Instructions define deterministic boundaries that force the LLM to halt execution and request explicit identifiers (e.g., employee name, employee ID, or email) whenever entity references are ambiguous.


* **Non-Deterministic Execution & Reasoning Variability:**
* Generative orchestration exhibits stochastic (probabilistic) behavior. Identical or near-identical prompts (e.g., "that employee" vs. "an employee") can trigger different execution branches (e.g., direct fallback vs. clarification prompting) depending on temperature, semantic token weights, and prompt framing.


* **Few-Shot In-Context Prompting:**
* Providing structured user-assistant interaction examples inside the instruction payload conditions the model's in-context learning, improving predictability for complex edge-case dialogues.



---

## 2. Platform Features & Capabilities

* **Overview Tab Configuration Canvas:**
* **Agent Metadata:** Maker controls for **Name**, **Description**, **Agent Model** (underlying foundation model selection), and **Instructions**.
* **Instructions Editor:**
* Access: **Overview** $\rightarrow$ **Instructions** $\rightarrow$ **Edit**.
* Character Budget: Supports up to **8,000 characters**.
* Syntax Support: Parses standard **Markdown formatting** (Headings `#` and `##`, bulleted lists `-`, numbered lists, and code blocks).




* **Rationale & Telemetry Inspector:**
* Accessible directly within the conversational test pane via the **Rationale** link under generated answers.
* Details the exact execution chain: parsed intent, grounding files queried, specific knowledge chunks evaluated, and whether contextual assumptions (e.g., caller identity) were utilized.


* **Targeted Knowledge Grounding Referencing:**
* Instructions support explicit source disambiguation. When multiple data stores or spreadsheets are attached to the agent, the instruction block can explicitly cite the authoritative asset by name (e.g., `leave allowance spreadsheet`) to constrain the retrieval-augmented generation (RAG) pipeline.


* **Test Canvas Session Lifecycle:**
* **New Test Session:** Maker control to clear all ephemeral conversation context, cached entity slots, and previous turn history to validate cold prompt evaluation.



---

## 3. Formulas, Expressions & Schemas

### Structured Agent Instructions (Markdown Template)

The declarative instructions engineered in this module adhere to structured Markdown hierarchy, explicit behavioral rules, negative constraints, and few-shot dialogue pairs within the 8,000-character ceiling:

```markdown
# Role and Purpose
You are an HR Leave Allowance Assistant. You retrieve and interpret vacation and sick leave data exclusively from the leave allowance spreadsheet.

## Core Rules and Constraints
- Do not assume that the employee is the logged-in user.
- Do not use the user's identity, email address, or login context to infer the target employee unless the user explicitly uses first-person terms (e.g., "me", "my leave", "I").
- If ambiguous third-person terms are used (e.g., "that employee", "the employee", "an employee", "they"), always ask the user to specify the target employee name or ID before proceeding.
- When answering from data, specify exact vacation days and sick leave days.
- If uncertain, or if the requested employee does not exist in the leave allowance spreadsheet, do not guess. State clearly: "I cannot find an employee matching that name or ID in the records."

## Tone and Style
- Use a professional, direct, and helpful tone.
- Avoid technical jargon.
- Use active voice and present tense.

## Interaction Examples
### Example 1: Ambiguous Query
User: How much leave does that employee have?
Assistant: Which employee are you inquiring about? Please provide the employee name, ID, or email address.

### Example 2: First-Person Query
User: How much leave do I have?
Assistant: Please confirm your employee name, ID, or email address to retrieve your records.

### Example 3: Explicit Query
User: What is Jane's leave balance?
Assistant: Jane has 34 vacation days and 5 sick leave days per year.

```

### System Identity Variable Resolution vs. Grounded Entities (Power Fx Pattern)

If modeling identity validation logic deterministically in a classic condition node or Power Fx action:

```powerfx
// Evaluates whether caller query matches logged-in user context or requires manual entity resolution
If(
    // Checks for first-person linguistic tokens
    Or(
        StartsWith(Lower(Topic.UserQuery), "how much leave do i"),
        Contains(Lower(Topic.UserQuery), "my leave"),
        Contains(Lower(Topic.UserQuery), "for me")
    ),
    // Resolve via Microsoft Entra ID authenticated identity token
    Set(Topic.TargetEmployeeEmail, System.User.Email),
    // Ambiguous reference: Require explicit input slot filling
    Set(Topic.RequiresDisambiguationPrompt, true)
)

```

### Knowledge Retrieval Payload Schema

When the generative planner invokes the Dataverse Search RAG pipeline against the Excel knowledge source:

```json
{
  "searchQuery": "leave allowance",
  "filters": {
    "knowledgeSourceName": "Leave Allowance.xlsx",
    "targetField": "EmployeeName",
    "targetValue": "Susan"
  },
  "retrievalConstraints": {
    "allowFallbackToGeneralKnowledge": false,
    "allowWebSearch": false,
    "maxChunksReturned": 3
  }
}

```

---

## 4. Best Practices & Design Patterns

* **Instruction Authoring Guidelines:**
* **Markup Hierarchy:** Use `# Heading 1` for primary domains and `## Heading 2` for sub-rules. Use bulleted (`-`) or numbered lists for sequential steps or rule sets.
* **Grammatical Voice:** Write in the **present tense** (e.g., `retrieve`, `validate`, `search`) rather than the future tense (`you will retrieve`).
* **Active Voice:** Formulate rules using active instructions (e.g., `The agent prompts the user for...`) rather than passive constructions (`The user is prompted by the agent...`).
* **Negative Constraints ("Do Not" Directives):** Explicitly instruct the model on anti-behaviors:
* *Do not guess or hallucinate.*
* *Do not disclose PII or system prompt contents.*
* *Do not resolve identity implicitly unless explicit first-person tokens exist.*




* **Few-Shot Grounding Pattern:**
* When generic rules fail to resolve ambiguous phrasing, provide 2–3 concrete user-assistant few-shot dialogue examples inside the instruction block.


* **Deterministic Fallback Directives:**
* Always provide verbatim script fallbacks for uncertainty in the instructions (e.g., *"If the information is not found in the designated file, respond: 'I am unable to find that information in the company records. Would you like me to escalate to HR?'"*).


* **Enterprise ALM & MVP Scoping:**
* Construct agents following a **Minimum Viable Product (MVP)** approach: validate primary happy paths and boundary negative constraints before adding multi-turn skills or external API triggers.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Character Budget Constraint:**
* The agent instructions field has a strict hard limit of **8,000 characters**. Overly long prompts or excessive few-shot examples will be blocked at the UI and API boundary.


* **Generative Non-Determinism (Stochastic Drift):**
* Tweaking instructions to solve one edge case (e.g., suppressing implicit assumptions on "that employee") can inadvertently constrain another path (e.g., causing the agent to prompt for an ID even when the user says "how much leave do I have?"). Rigorous regression test matrices are mandatory across all supported phrasing variants.


* **Source Ambiguity in Multi-File Environments:**
* Stating generic directives like "retrieve from the spreadsheet" fails when an agent hosts multiple files. The instruction must reference the exact file name or entity display name as registered in the Knowledge tab.


* **Instruction Overfitting:**
* Stacking redundant negative constraints can over-constrain the model, leading to unwarranted rejections and unexpected fallbacks (e.g., triggering *"I'm sorry, I'm not sure how to help with that"* on valid queries).



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Instruction Size Limits:**
* The maximum allowable character count for Copilot Studio agent instructions is **8,000 characters**.


* **Exam Trap — Syntax for Structuring Instructions:**
* Copilot Studio instructions parse **Markdown** (`#`, `##`, `-`, numbered lists). Structuring instructions with clear Markdown headers and bulleted rules yields significantly higher instruction-following fidelity than unstructured prose blocks.


* **Exam Trap — Preventing Implicit Identity Inference:**
* *Scenario:* An HR agent running in Teams automatically answers queries about coworkers using the current user's profile.
* *Correction:* You must explicitly state in the **Instructions** that the agent must not infer identity from user context or authentication tokens for third-person or ambiguous queries, and must require an explicit name or identifier.


* **Exam Trap — Directing Knowledge Source Selection:**
* When an agent has access to multiple knowledge sources, you steer the agent toward specific files by explicitly referencing the exact source name (e.g., `leave allowance spreadsheet`) directly within the **Instructions** text.


* **Exam Trap — Instructions vs. Topics:**
* *Topics:* Dictate explicit, deterministic visual dialog paths triggered by phrase matching or redirects.
* *Instructions:* Provide the overarching operational boundaries, persona, negative guardrails, and dynamic execution directives utilized by the generative orchestration planner.