## 1. Core Architecture & Concepts

* **Computer Use Runtime Execution Lifecycle:**
* **Orchestration Boundary:** In Copilot Studio, **Computer Use** functions as a specialized, vision-language model (VLM)-driven tooling extension integrated under the broader Model Context Protocol (MCP) tool-calling abstraction.
* **Virtual Machine Provisioning & Sandboxing:** When invoked, the system dynamically provisions or binds a cloud desktop environment (e.g., a Hosted Browser powered by Windows 365 for Agents). The session queues with a status of *"computer use will begin as soon as a computer is available"* before transitioning to *"computer is now ready"*.
* **Vision-Guided Closed-Loop Automation (Self-Correction / Retry Mechanism):**
* Unlike brittle DOM-selector-based RPA or static coordinates, the vision agent operates in an iterative visual feedback loop: **Observe Screen $\rightarrow$ Determine Plan $\rightarrow$ Dispatch OS Input $\rightarrow$ Re-Observe Screen to Verify State**.
* If an action fails (e.g., text fails to inject into an input field or a suggestion flyout fails to open), the agent visually detects that the UI state has not transitioned, automatically retries the keystrokes/clicks, locates the suggestion overlay, clicks the dynamic suggestion dropdown, and extracts the target textual output.




* **Autonomous vs. Conversational Agent Fit:**
* **Latency Asymmetry:** Computer Use operations incur substantial round-trip execution latencies (e.g., **1 minute 40 seconds (100 seconds)** for a basic sequence involving launching Edge, URL navigation, text injection, dynamic autocomplete selection, and optical data extraction).
* **Conversational Agent Anti-Pattern:** Synchronous chat interactions experience severe UX degradation and potential channel timeouts (such as Microsoft Teams or Web Chat 15-second response expectations) when waiting on virtual machine spinning and vision processing.
* **Autonomous Agent Alignment:** Ideal for asynchronous, event-driven autonomous agents (e.g., an email-triggered background workflow where an inbound email is processed and a draft or confirmation reply is generated without an active human blocking on the conversational channel).



---

## 2. Platform Features & Capabilities

* **Tool Testing Canvas (`Computer Use` Interface):**
* **State Dependency:** The UI disables test execution with an explicit banner warning: *"Please save your changes before testing"*. The configuration must be saved to Dataverse before the **Test** button becomes active.
* **Live VM Streaming & Control:** Provides a sandboxed test viewer allowing makers to monitor OS-level interactions live on the remote Microsoft Windows desktop (launching Microsoft Edge, address bar navigation, typing, drop-down selection, and data reading).
* **Manual Input Parameter Testing:** Enables direct manual payload binding (e.g., testing `employeeName = "Jane"`) via a modal prompt prior to clicking **Test now**.
* **Session Termination:** Makers must explicitly click **Finish testing** to release the remote session resources, disconnect the stream, and unlock configuration fields.


* **System Instructions Authoring & Forward-Slash (`/`) Binding:**
* Under the agent **Overview** tab $\rightarrow$ **Instructions**, tools are bound directly into natural language execution steps using the forward slash (`/`) symbol (e.g., `/number of years in service`).
* **Step Construction Standard:** Instructions require explicit step definitions:
* **Goal:** High-level objective of the task.
* **Action:** Direct delegation statement invoking the `/ToolName` with contextual entity parameters.
* **Transition:** Downstream flow directives telling the agent which step to proceed to once output variables are bound.




* **Autonomous Agent Trigger Testing:**
* Because autonomous agents lack a conversational user input box, testing is initiated via the **Test trigger** button in the canvas header.
* Clicking **Start testing** injects a simulated trigger event (e.g., an inbound email trigger with mock metadata: `Name: Maria`, `Role: Secretary`).
* **Rationale Inspection:** During test execution, selecting the **Rationale** button surfaces the orchestrator’s reasoning trace, displaying why the agent selected the tool, how it resolved parameter entities, and how it routed outputs to subsequent steps.



---

## 3. Formulas, Expressions & Schemas

### Autonomous Agent Instruction Template (Prompt Markdown)

When integrating an event-driven computer use tool into a multi-step autonomous workflow:

```markdown
# Role & Workflow Instructions
You are an autonomous HR Vacation Leave Checker agent. When an inbound leave request email is detected, process the steps sequentially:

Step 1: Extract Inbound Metadata
- Goal: Parse sender name, leave dates, and job role from the incoming email trigger.
- Action: Store sender full name into Topic.EmployeeName and job role into Topic.EmployeeRole.
- Transition: Proceed to Step 2.

[... Intermediate Allowance and Balance Checks ...]

Step 5: Retrieve Number of Years in Service
- Goal: Retrieve the number of years in service for the employee.
- Action: Use the /number of years in service tool by passing Topic.EmployeeName to return the numeric years in service.
- Transition: Once data is retrieved, proceed to Step 6.

Step 6: Prepare and Dispatch Notification
- Goal: Construct and send the vacation acknowledgment email.
- Action: Draft an email using the Office 365 Outlook connector including:
  * Employee Name (Topic.EmployeeName)
  * Employee Job Role (Topic.EmployeeRole)
  * Number of Years in Service (Topic.YearsInService)
- Transition: Proceed to Step 7 (End conversation / Log activity).

Step 7: Finalize Run
- Goal: Close execution trace and record completion metrics.

```

### JSON Schema: Computer Use Dynamic Tool Execution Trace

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "ComputerUseToolExecutionTrace",
  "type": "object",
  "properties": {
    "executionId": { "type": "string", "example": "exec-vm-20261109-8832" },
    "toolType": { "type": "string", "enum": ["ModelContextProtocol/ComputerUse"] },
    "runtimeEnvironment": { "type": "string", "enum": ["HostedBrowser", "CloudPCPool", "BYOM"] },
    "durationSeconds": { "type": "integer", "example": 100 },
    "stepsExecuted": { "type": "integer", "example": 6 },
    "billedCopilotCredits": { "type": "integer", "example": 30 },
    "inputParameters": {
      "type": "object",
      "properties": {
        "employeeName": { "type": "string", "example": "Maria" }
      },
      "required": ["employeeName"]
    },
    "outputParameters": {
      "type": "object",
      "properties": {
        "yearsInService": { "type": "integer", "example": 8 }
      },
      "required": ["yearsInService"]
    },
    "selfCorrectionEvents": [
      {
        "step": 3,
        "action": "SendKeys",
        "status": "FailedVerification",
        "retryCount": 1,
        "resolvedStatus": "Success"
      }
    ]
  },
  "required": ["executionId", "toolType", "durationSeconds", "billedCopilotCredits", "outputParameters"]
}

```

### Power Fx: Handling Tool Output in Response Generation

```powerfx
// Step 1: Format dynamic email body string incorporating Computer Use output
Set(Topic.EmailBodyContent,
    $"We received your vacation request for {Topic.EmployeeName}. " &
    $"Our records confirm that they are currently assigned as a {Topic.EmployeeRole} " &
    $"with {Text(Topic.YearsInService)} years of service at the company."
);

// Step 2: Guardrail check for null or invalid extraction before sending
If(
    IsBlank(Topic.YearsInService) Or Topic.YearsInService < 0,
    Set(Topic.FinalEmailPayload, Substitute(Topic.EmailBodyContent, "with " & Text(Topic.YearsInService) & " years of service", "with an unconfirmed tenure")),
    Set(Topic.FinalEmailPayload, Topic.EmailBodyContent)
);

```

---

## 4. Best Practices & Design Patterns

* **Model Selection Resiliency & Cache Clearing:**
* If testing halts with a *"specified model is invalid"* runtime error, re-select the target vision-reasoning model from the tool's dropdown menu, re-save the configuration, and relaunch the test to force re-binding of the underlying model endpoint.


* **Concurrency Management & Connection Throttling:**
* Do not trigger consecutive test runs in rapid succession. Rapid execution while a preceding remote VM session or hosted browser teardown is still active causes connection collisions, locking, or execution failures due to unreleased remote desktop threads.


* **Step Explicit Transition Design:**
* When authoring instructions that invoke asynchronous or external UI tools, always specify explicit **Goal**, **Action**, and **Transition** statements. If intermediate steps are added (e.g., adding a new Step 5), re-index all subsequent steps (updating Step 5 to Step 6, and Step 6 to Step 7) and explicitly update prerequisite pointers (*"In the previous step, once data is retrieved, go to step 5"*).


* **Architectural Fallback Hierarchy:**
* Always verify whether an API, native Power Platform Connector, or Dataverse entity exists before committing to Computer Use. Computer Use should be strictly reserved for legacy applications, SaaS portals without API/webhooks, or intranet web pages lacking structured programmatic interfaces.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Billing & Consumption Metrics:**
* **Credit Burn Rate:** Each discrete operational step executed by Computer Use consumes **5 Copilot credits** (approximately **$0.05 USD / 5 cents** per action step). Complex UI interactions with multiple keystrokes, retries, and navigation steps scale linearly in consumption costs.


* **Execution Time (Latency Floor):**
* Even trivial sequences (navigating to a single webpage, entering text, clicking an autocomplete box, and scraping a number) take on the order of **1 to 2 minutes** (100 seconds observed). Real-time synchronous chatbot scenarios will degrade user experience or hit gateway timeouts.


* **Unsaved Changes Block Testing:**
* The test canvas strictly prohibits execution if unsaved modifications exist in the tool editor. Any instruction, parameter, or model update must be committed via **Save** before clicking **Test**.


* **Dynamic Autocomplete Race Conditions:**
* Legacy web applications that use asynchronous suggestion flyouts require the agent to pause, observe the rendering of the dropdown list, and explicitly click the suggestion item rather than simply pressing enter or tabbing out.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Architectural Positioning (Computer Use vs. Connectors):**
* *Question:* "An architect needs to retrieve data from an internal website for a customer-facing conversational bot. Turnaround time must be under 5 seconds. How should this be implemented?"
* *Trap Answer:* "Configure a Computer Use tool using a Hosted Browser." (Incorrect: High latency of ~60–100+ seconds will fail the latency requirement).
* *Correct Answer:* "Develop a custom connector or REST API / Azure Function wrapper to query the data programmatically."


* **Exam Trap — Agent Model Suitability:**
* Computer Use is functionally supported on both conversational and autonomous agents, but is **architecturally best suited for autonomous agents** where human users are not actively waiting on an interactive conversational thread.


* **Exam Trap — Licensing & Metering:**
* Computer Use actions are billed against tenant Copilot capacity at a rate of **5 Copilot credits per action step**.


* **Exam Trap — Tool Categorization in Traces:**
* In execution telemetry and orchestrator rationale traces, Computer Use is classified as a specialized implementation under the **Model Context Protocol (MCP)** tool ecosystem.


* **Prerequisite Testing Rule:**
* A maker **cannot** test a Computer Use tool without first explicitly clicking **Save**. Unsaved configurations will disable testing functionality.