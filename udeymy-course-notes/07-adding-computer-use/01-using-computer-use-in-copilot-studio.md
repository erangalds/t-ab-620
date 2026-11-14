## 1. Core Architecture & Concepts

* **Computer Use Framework in Copilot Studio:**
* An agentic capability allowing autonomous and conversational agents to control a Windows operating system desktop environment, interact with user interfaces (UI automation, keyboard/mouse actions), and automate tasks on web applications or legacy desktop software lacking accessible APIs or direct connectors.
* Operates as a **Tool** within the agent configuration (`Tools` $\rightarrow$ `Add a tool` $\rightarrow$ `Computer use`).
* Employs vision-language models capable of parsing screen states, locating visual UI elements (text boxes, buttons, labels), dispatching OS-level inputs (clicks, keystrokes), and extracting textual/numerical state from target controls.


* **Target Machine Execution Architectures:**
* **Hosted Browser:**
* Powered by **Windows 365 for Agents**.
* Lightweight, managed virtual browser sandbox provisioned on-demand for browser-only workflows.
* Does not support native Windows desktop applications or private internal enterprise resource access (e.g., resources requiring VPN or private line routing).


* **Cloud PC Pool:**
* Powered by **Windows 365 for Agents**.
* A scalable cluster of managed Windows virtual machines joined to **Microsoft Entra ID** and enrolled in **Microsoft Intune**.
* Applies corporate mobile device management (MDM), conditional access, and group governance policies.
* Provides native integration with work/school identities (M365, SharePoint, Azure resources) using auto-scaling, pay-as-you-go compute.


* **Bring Your Own Machine (BYOM):**
* Self-hosted virtual machine or physical hardware registered to Power Automate via the Power Automate machine runtime.
* Uses **Power Automate for desktop (PAD)** host services to execute agent-driven UI interactions.
* Requires explicit environmental isolation, software baseline control, and strict credential hygiene.




* **Instruction Builder Mechanism:**
* A specialized authoring utility that compiles high-level procedural descriptions into structured, granular OS-level execution steps (e.g., launching Edge from the taskbar, navigating to an explicit URI, waiting for rendering, locating labels and adjacent input boxes, and reading text fields).



---

## 2. Platform Features & Capabilities

* **UI Path & Provisioning:**
* **Agent Tools Navigation:** `Tools` $\rightarrow$ `Add a tool` $\rightarrow$ `Computer use`.
* **Pre-built Templates:** Starter templates provided in the configuration gallery or custom description prompt.


* **Computer Use Tool Configuration Parameters:**
* `Name`: Display and logical name (e.g., *Number of years in service*).
* `Description`: Detailed functional description exposing semantic metadata to the orchestrator to decide *when* to invoke the tool.
* `Model Selection`: Dropdown allowing selection of the underlying vision/reasoning model powering screen parsing and input generation.
* `Instructions`: Multi-step operational prompt detailing exact system behavior.
* `Instruction Builder`: Modal window to refine brief requirements into expanded, structured system commands via the **Generate instructions** capability.
* `Inputs`: Typed parameter mapping. Configured with a Name (e.g., `employeeName`), Description (*"Enter the employee name here"*), and auto-populated at runtime by the agent.
* `Outputs`: Explicit typed output contracts (e.g., `number` for duration of service).


* **Identity & Credential Delegation:**
* **Maker-Provided Credentials:** Fixed author credentials used across all execution sessions.
* **End-User Credentials:** Evaluated at runtime using the active interacting user's credentials.
* **Stored Credentials:** Automated credential injection for login popups/forms; can be stored internally in **Power Platform** or externally via **Azure Key Vault**.


* **Security & Network Boundary Controls:**
* **Target Scope Whitelisting:** Restricts agent navigation to authorized desktop applications (e.g., `notepad.exe`) or domains using wildcards (e.g., `*.microsoft.com`). Prevents unauthorized process launches or outbound navigation.
* **Enforce HTTPS / Secure Websites Only:** Binary toggle that enforces strict HTTPS schemes, blocking insecure `http://` protocols.


* **Power Platform Admin Center (PPAC) Governance & Logging:**
* **Navigation:** PPAC $\rightarrow$ `Manage environments` $\rightarrow$ Select Environment $\rightarrow$ `Settings` $\rightarrow$ `Product` $\rightarrow$ `Features` $\rightarrow$ Scroll to **Computer Use**.
* **Environment Toggle:** Global enable/disable switch for the entire Dataverse environment.
* **Dataverse Logging Verbosity Levels:**
* `All data including screenshots`: Full execution capture comprising screen frames, inputs, coordinates, and diagnostic logs.
* `Data without screenshots`: Captures action types, bounding coordinates, execution timestamps, user content, inputs, and descriptions; excludes screen captures.
* `Summary`: Lightweight execution record detailing instructional text, run inputs, total duration, step counts, average action duration, human escalation count, machine metadata (machine name, URL, user login), visited apps/sites, and credential references.
* `Minimum`: Functionally identical to *Data without screenshots*.


* **Log Retention Policy:** Configurable retention duration for Dataverse log persistence.
* **Purview Integration:** Optional toggle to stream structured audit logs into **Microsoft Purview** for unified enterprise compliance, eDiscovery, and security auditing.



---

## 3. Formulas, Expressions & Schemas

### Structured Instruction Schema for Vision-Guided Execution

Computer Use instructions compiled by the Instruction Builder follow an imperative sequential operational contract:

```yaml
version: 1.0
toolType: ComputerUse
targetTask: RetrieveEmployeeTenure
executionSequence:
  - step: 1
    action: LaunchApplication
    target: "Microsoft Edge"
    locator: "Taskbar > Icon[Name='Microsoft Edge']"
  - step: 2
    action: NavigateToUrl
    url: "https://idodata.com/computer-use-example"
    enforceProtocol: HTTPS
  - step: 3
    action: LocateElement
    description: "Locate input field adjacent to label"
    selectorCriteria:
      targetType: TextBox
      relativeTo: "Label[Text='Employee Name']"
      position: Right
  - step: 4
    action: SendKeys
    value: "{Topic.EmployeeName}"
    clearExisting: true
  - step: 5
    action: ExtractValue
    selectorCriteria:
      targetType: TextBox
      relativeTo: "Label[Text='Years in Service']"
      position: Right
    outputVariable: "Topic.YearsInService"
    targetType: Float

```

### JSON Tool Input / Output Contract Definition

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "ComputerUseServiceTenureLookup",
  "type": "object",
  "properties": {
    "inputs": {
      "type": "object",
      "properties": {
        "employeeName": {
          "type": "string",
          "description": "Full name of the employee to look up in the legacy web portal"
        }
      },
      "required": ["employeeName"]
    },
    "outputs": {
      "type": "object",
      "properties": {
        "yearsInService": {
          "type": "number",
          "description": "The numeric value representing the total years the employee has served at the company"
        }
      },
      "required": ["yearsInService"]
    }
  }
}

```

### Power Fx: Calling Tool & Handling Output in Dynamic Conversational Topics

```powerfx
// Step 1: Validate input extraction before invoking computer use tool
If(
    IsBlank(Topic.ExtractedEmployeeName),
    Set(Topic.PromptMessage, "Please specify an employee name."),
    // Step 2: Invoke tool and map response to Topic variable
    Set(Topic.YearsInServiceResult, 
        ActionNode_ComputerUseTenure.Execute(
            { employeeName: Topic.ExtractedEmployeeName }
        ).yearsInService
    )
);

// Step 3: Validate numeric payload received from automation
If(
    Not(IsBlank(Topic.YearsInServiceResult)) And Topic.YearsInServiceResult >= 0,
    Set(Topic.FormattedServiceString, Topic.ExtractedEmployeeName & " has been with the company for " & Text(Topic.YearsInServiceResult) & " years."),
    Set(Topic.FormattedServiceString, "Could not determine years in service from the legacy application.")
)

```

---

## 4. Best Practices & Design Patterns

* **Hierarchy of Integration (Exhaustion Principle):**
* Computer Use is an architectural **pattern of last resort**. Always evaluate alternatives in this strict order:
1. Direct Power Platform Connectors (Dataverse, SharePoint, SQL, Graph).
2. REST APIs / OpenAPI Custom Connectors / Azure Functions.
3. Headless Power Automate Cloud Flows / Graph API requests.
4. Traditional Power Automate for desktop (API/DOM selectors).
5. **Computer Use (Vision-Driven Screen Interaction).**


* *Reasoning:* Computer Use incurs higher latency, nondeterministic visual dependencies, and increased compute/licensing costs compared to programmatic APIs.


* **Prompt Specificity & URL Formatting:**
* Always supply Fully Qualified Domain Names (FQDN) including protocol prefixes (`https://...`). Avoid vague natural language instructions like "go to the website."
* Explicitly define spatial layout references for the vision model: *"locate the text box to the right of the label 'Years in Service'"*.


* **Principle of Least Privilege on BYOM Hosts:**
* Dedicated host machines registered for Computer Use must run under service accounts configured with the absolute minimum administrative and network permissions required to execute the targeted task.
* Machines must be strictly isolated from personal or administrative workloads.


* **Dynamic Domain Guardrails:**
* Explicitly configure domain allowlists (`*.company.com`) and process filters (`winword.exe`, `excel.exe`) to block accidental data exfiltration, lateral navigation, or prompt-injection-driven browsing.


* **Production Telemetry Management:**
* Avoid capturing screenshots (`All data including screenshots`) in production environments handling sensitive business data, payment details, or PII. Use `Data without screenshots` or `Summary` to prevent unintended storage of regulated visual data in Dataverse.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Geographic Tenant Boundaries:**
* Computer Use features and Windows 365 for Agents compute endpoints are restricted to Dataverse environments deployed in the **United States (US) region**. Tenants in Europe, APAC, or Sovereign Clouds cannot provision or run these capabilities.


* **Hosted Browser Capability Boundary:**
* The Hosted Browser profile **cannot**:
* Access local or line-of-business (LOB) Windows desktop executables (`.exe`, `.msc`).
* Route through corporate on-premises data gateways or local subnet firewalls.
* Access internal intranets not exposed to public DNS without corporate VPN integration.




* **Logging Storage Overhead:**
* Setting logging verbosity to `All data including screenshots` creates heavy Dataverse database and file storage consumption. High transaction rates can exhaust tenant Dataverse storage capacity.


* **Execution Latency:**
* Vision-based execution involves capturing virtual frames, sending them to the model, generating bounding-box coordinates, and dispatching virtual inputs. Round-trip latency is significantly higher than API execution (often requiring 10–30+ seconds per interaction step).


* **State Drift & Transient Popups:**
* Unanticipated OS notifications, browser updates, cookie consent dialogs, or CAPTCHA screens will disrupt vision pipelines, resulting in failed runs or human-escalation events unless explicitly anticipated in the instructions.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Best Tool for Legacy Web Applications:**
* *Question:* "An agent needs to extract data from a legacy internal web app that has no REST API or Power Platform connector. How should you automate this with minimal infrastructural footprint?"
* *Answer:* Use **Computer Use** with a **Hosted Browser** target machine (if browser-only) or a **Cloud PC pool** (if enterprise domain/intranet authentication is required).


* **Exam Trap — Selecting Compute Infrastructure:**
* *Hosted Browser:* Ideal for quick, public web-only interactions without internal network requirements.
* *Cloud PC Pool:* Best for auto-scaling, Intune-enrolled, Entra ID-governed virtual machines accessing internal M365/Azure enterprise resources.
* *Bring Your Own Machine (BYOM):* Required when using custom on-premises physical/virtual setups utilizing Power Automate for desktop runtimes.


* **Regional Availability Trap:**
* If a case study states that an organization is hosted entirely in the **Europe (EUR)** or **United Kingdom (UK)** region and wants to enable Computer Use natively in Copilot Studio, this is **unsupported** (restricted to US-hosted environments).


* **Governance & Compliance Logging Configuration:**
* Computer Use audit and telemetry logging is governed in the **Power Platform Admin Center (PPAC)** under Environment Features, NOT within the Copilot Studio topic authoring canvas.
* Logs can be stored natively in **Dataverse** and streamed directly to **Microsoft Purview** for audit compliance.


* **Verbosity Level Differences:**
* `All data including screenshots` $\rightarrow$ captures full screen frames.
* `Data without screenshots` $\rightarrow$ logs coordinates, inputs, action types, timestamps, but strips visuals.
* `Summary` $\rightarrow$ logs run metrics, durations, machine name, and human escalation counts.
* `Minimum` $\rightarrow$ functionally identical to *Data without screenshots*.