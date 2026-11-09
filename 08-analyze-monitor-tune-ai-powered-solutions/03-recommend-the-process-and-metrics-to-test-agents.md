## 1. Core Architecture & Concepts

* **Copilot Studio Agent Evaluation Framework:** A built-in, systematic testing and benchmarking harness designed to assess agent answer quality, accuracy, safety, and orchestration behavior using reproducible test sets rather than subjective ad-hoc manual testing.
* **Test Set Architecture:**
* A container aggregating multiple deterministic or synthetic **test cases** (questions/prompts).
* Evaluates responses against configured baseline expectations, scoring thresholds, and semantic matching models.


* **Execution & Evaluation Engine:**
* Executes asynchronous test batches using the agent’s active or draft runtime configuration.
* Uses LLM-as-a-judge evaluators (for general quality, text similarity, and semantic comparison) alongside deterministic assertion engines (for keyword, exact string, and tool utilization checks).


* **Testing Method Taxonomies:**
* **General Quality:** A multi-dimensional evaluative metric scored as a percentage ($0\text{--}100\%$) assessing four core criteria:
* *Relevance:* Assesses whether the response directly addresses the question and remains on-topic without conversational drift.
* *Groundedness:* Verifies whether the output strictly derives from provided knowledge sources and context without hallucinations or ungrounded claims.
* *Completeness:* Evaluates whether all facets and sub-questions of the prompt are fully resolved.
* *Abstention:* Validates whether the agent appropriately declined to answer or adhered to guardrails when information was missing or out of scope.


* **Compare Meaning:** Semantic evaluation measuring whether the generated answer conveys the same core meaning and intent as the reference expected response. Scored from $0\text{--}100\%$ with a **default passing score threshold of 50%**.
* **Keyword Match:** Deterministic lexical search verifying the presence of target words or phrases in the response. Scored as a binary **Pass/Fail**. Supports matching **All** specified keywords or **Any** keyword.
* **Text Similarity:** Algorithmic similarity comparison measuring character/token overlap and textual closeness against the expected response. Scored from $0\text{--}100\%$.
* **Exact Match:** Strict, deterministic, verbatim comparison (character-for-character, word-for-word). Scored as a binary **Pass/Fail**.
* **Capability Use (Tool Use):** Verifies whether the agent correctly recognized intent and invoked specific designated tools, connectors, agent flows, or topics to produce the output. Scored as **Pass/Fail**.
* **Classification (Custom):** Applies custom user-defined instructions, criteria, and labels to categorize and evaluate the agent's response (e.g., policy compliance, tone, format adherence).



---

## 2. Platform Features & Capabilities

* **Navigation & Workspace:**
* Located on the agent canvas under the dedicated **Evaluation** tab.
* Houses test sets, past test runs, aggregate scores, and detailed drill-down execution telemetry.


* **Test Set Creation Workflows:**
1. **Manual Authoring:** Selecting *Write some questions yourself* allows manual entry of prompt strings, expected responses, and test method assignments per test case.
2. **File Import:**
* Supported file formats: Comma-Separated Values (`.csv`) or plain text (`.txt`).
* Mandatory header structure: `Question`, `Expected response`, and `Testing method` (or configured per-row during review).


3. **Copilot-Assisted Generation:**
* *Quick question set:* Automatically synthesizes **10 test questions** derived from the agent’s description, instructions, and configured capabilities.
* *Full question set:* Generates **up to 100 test questions** referencing configured **Knowledge Sources** (optimal for Generative Orchestration) or **Topics** (optimal for Classic Orchestration).
* *From Test Chat Conversation:* Converts the interactive multi-turn dialogue history from the current active test pane session directly into a reusable evaluation test set.




* **Execution Configuration & Security Context:**
* **User Profile Selection:** Allows selecting a specific test user identity and connection credentials prior to running the test.
* **Security Boundary Testing:** Enables validation of role-based access control (RBAC), Entra ID permissions, and data loss prevention (DLP) by executing evaluations under restricted versus privileged personas.


* **Execution Actions:**
* **Save:** Persists the configured test set, questions, and test methods without triggering an execution run.
* **Evaluate / Run:** Commits the test set and immediately queues asynchronous batch evaluation execution (typically takes several minutes depending on test set volume).


* **Results & Diagnostics Viewer:**
* Displays an aggregate agent score (e.g., *Good quality agent at 90%*) alongside per-question Pass/Fail status.
* Provides diagnostic rationales explaining failures (e.g., flagging responses that are *relevant but incomplete* due to hanging confirmation questions like *"Would you like me to also include the upcoming approved leave?"*).



---

## 3. Formulas, Expressions & Schemas

### Test Set Import File Schema (`.csv`)

When importing external test suites into the Copilot Studio Evaluation workspace, the file must adhere to the standard tabular schema:

```csv
Question,Expected response,Testing method
"How much vacation leave do I have left?","You have 15 days of vacation leave remaining.","Compare meaning"
"What is the company sick leave policy?","Employees receive 5 paid sick days annually.","General quality"
"Who is the CTO?","Marcus Vance","Exact match"
"Apply for 2 days leave starting tomorrow","Your leave request has been submitted for approval.","Capability use"

```

### JSON Test Case Data Contract

The underlying object structure representing an authored test case inside an evaluation suite:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "AgentEvaluationTestCase",
  "type": "object",
  "properties": {
    "testCaseId": { "type": "string" },
    "question": {
      "type": "string",
      "maxLength": 1000,
      "description": "User prompt submitted to the agent."
    },
    "expectedResponse": {
      "type": "string",
      "description": "Ground truth reference answer (required for Compare meaning, Text similarity, Exact match)."
    },
    "testMethod": {
      "type": "string",
      "enum": [
        "General quality",
        "Compare meaning",
        "Keyword match",
        "Text similarity",
        "Exact match",
        "Capability use",
        "Classification"
      ]
    },
    "configuration": {
      "type": "object",
      "properties": {
        "passingScoreThreshold": {
          "type": "number",
          "default": 50,
          "description": "Passing percentage for Compare meaning and Text similarity."
        },
        "keywordMatchMode": {
          "type": "string",
          "enum": ["All", "Any"],
          "default": "All"
        },
        "expectedKeywords": {
          "type": "array",
          "items": { "type": "string" }
        },
        "expectedCapabilities": {
          "type": "array",
          "items": { "type": "string" }
        }
      }
    }
  },
  "required": ["question", "testMethod"]
}

```

### Power Fx: Evaluation Threshold Assertion Logic

Equivalent expression used by the evaluation engine to evaluate passing grades for semantic similarity and keyword tests:

```powerfx
// Compare Meaning / Text Similarity Evaluation Check
Set(Topic.IsPassed,
    If(
        Topic.TestMethod = "Compare meaning" Or Topic.TestMethod = "Text similarity",
        Topic.EvaluatedScore >= Topic.ConfiguredPassScoreThreshold, // Defaults to 50%
        If(
            Topic.TestMethod = "Exact match",
            Topic.ActualResponse = Topic.ExpectedResponse,
            If(
                Topic.TestMethod = "Keyword match" And Topic.KeywordMatchMode = "All",
                CountRows(Filter(Topic.ExpectedKeywords, Not(Value in Topic.ActualResponse))) = 0,
                // Keyword match mode: Any
                CountRows(Filter(Topic.ExpectedKeywords, Value in Topic.ActualResponse)) > 0
            )
        )
    )
)

```

---

## 4. Best Practices & Design Patterns

* **Regression Testing in ALM Pipelines:**
* Maintain a permanent golden test set of up to 100 test cases per agent.
* Re-run evaluation suites whenever:
* Knowledge sources are added, updated, or re-indexed.
* System instructions or agent descriptions are modified.
* Underlying foundation models are changed or updated.
* New tools, connectors, or topics are deployed.




* **Method Selection Strategy:**
* *Generative Answers from Knowledge Sources:* Use **General Quality** or **Compare Meaning** (allows syntactic flexibility while ensuring substance and groundedness).
* *Deterministic Topics with Static Text / Formulas:* Use **Exact Match** or **Text Similarity**.
* *Compliance & Policy Verification:* Use **Keyword Match** with mode set to `All` to enforce mandatory legal disclosures or disclaimers.
* *Action / Plugin Triggering:* Use **Capability Use** to guarantee that user intent routes to the appropriate tool rather than hallucinating an answer.


* **Handling Conversational Proactivity / Clarification Pitfalls:**
* When an agent ends an answer by proactively asking a follow-up question (e.g., *"Would you like me to check upcoming leave?"*), **General Quality** evaluators may score the response as *Incomplete* because the turn is interpreted as waiting on an unfulfilled action.
* *Remedy:* Ensure system instructions distinguish between final answers and conversational follow-ups, or refine the expected response criteria to account for proactive prompts.


* **Role-Based Security Evaluation:**
* Run test sets twice across different personas: once signed in as an unauthorized user (asserting abstention/refusal) and once as an authorized manager (asserting complete data retrieval) to catch permission leaks.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Test Set Capacity Limit:**
* A single test set is hard-capped at **100 test cases (questions)**.


* **Question Character Constraint:**
* The `Question` field in any test case cannot exceed **1,000 characters**.


* **Missing Expected Response Invalidation:**
* If a test case assigns **Compare meaning**, **Text similarity**, or **Exact match**, but the *Expected response* column is left blank, the evaluation engine flags the result as **Invalid** rather than evaluating it.


* **Execution Duration & Concurrency:**
* Evaluation batches run asynchronously and can take **several minutes** to process a 10-to-100 question batch depending on platform load and model latency.
* Only **one evaluation run** can execute at a time per agent.


* **Historical Retention Window:**
* Evaluation test run histories and granular results are retained in Copilot Studio for **89 days**. Historical evidence required for long-term compliance audits must be exported manually.


* **Default Baseline Behavior for Unknown Entities:**
* When queries include unnamed or unrecognized entities (e.g., *"How much leave does Emily Johnson have?"* where Emily is not in the database), generative agents may default to providing company-wide baseline policies (e.g., standard 10 vacation days) rather than abstaining, unless strict negative-path guardrails are instructed.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — General Quality Four Pillars:**
* General Quality does NOT rely on a user-provided expected response string. It evaluates the agent's response using four specific LLM-judged criteria: **Relevance**, **Groundedness**, **Completeness**, and **Abstention**.


* **Exam Trap — Default Passing Score:**
* For the **Compare meaning** evaluation method, the default platform passing score threshold is **50%** (configurable between 0% and 100%).


* **Exam Trap — Question Set Generation Source Alignment:**
* *Quick question set:* Generates exactly **10 questions** based on agent description and instructions.
* *Full question set:* Generates up to **100 questions** based on **Knowledge sources** (for Generative Orchestration) or **Topics** (for Classic Orchestration).
* *From conversation:* Converts manual chat sessions into test sets directly.


* **Exam Trap — Supported Import Headers:**
* Import files must use CSV or TXT format with the required headers: `Question`, `Expected response`, and `Testing method`.


* **Exam Trap — Metric Scoring Types:**
* *Percentage Scored ($0\text{--}100\%$):* **General quality**, **Compare meaning**, and **Text similarity**.
* *Binary Scored (Pass/Fail):* **Keyword match**, **Exact match**, **Capability use**, and **Content safety**.