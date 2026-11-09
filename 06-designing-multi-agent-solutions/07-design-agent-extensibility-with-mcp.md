## 1. Core Architecture & Concepts

* **Model Context Protocol (MCP):**
* An open, standardized protocol enabling AI models and orchestrators to securely discover, expose, and invoke external data sources, contexts, and executable tool systems.
* Serves as Microsoft’s recommended architectural standard to surface enterprise data and actions to agents with centralized enterprise-grade security, granular authentication, and full operational auditing.
* **Single Tool vs. System of Tools Architecture:** Unlike traditional discrete plugins, flows, or REST APIs that map to a single atomic action (1:1 binding), an MCP server functions as a centralized gateway exposing a dynamic collection/system of multiple tools (1:N binding) under a single consolidated endpoint and authentication context.


* **Orchestration Dependency:**
* MCP integration strictly requires the agent to be configured with **Generative Orchestration** (Dynamic Chaining). Classic deterministic intent recognition cannot evaluate or invoke dynamically introspected MCP tool sets.


* **Agent-to-Agent (A2A) vs. Model Context Protocol (MCP):**
* **MCP (Model Context Protocol):** Designed for *tool augmentation and context retrieval*. The primary agent consumes tools and resources directly from a client-server endpoint to execute tasks. Configured under **Tools**.
* **A2A (Agent to Agent):** An open standard backed by the Linux Foundation and defined within the Microsoft Cloud ecosystem designed for *peer-level multi-agent collaboration, delegation, and negotiation*. Configured under **Agents** (External Agents).


* **Work IQ:**
* Microsoft’s branded ecosystem and operational framework encompassing enterprise-tailored, first-party MCP servers designed to integrate deeply with organizational data and Microsoft 365 services.


* **Authentication & Delegation Models:**
* **Maker-Provided Credentials:** Actions execute using the connection identity and authorization permissions of the author/developer who configured the agent. Users running the agent share this pooled service context.
* **End-User Credentials (User Delegation):** Actions execute strictly under the identity, security permissions, and enterprise access boundaries of the active user interacting with the agent.



---

## 2. Platform Features & Capabilities

* **Extensibility Paths in Copilot Studio Canvas:**
* **Tools Navigation:** **Tools** $\rightarrow$ **Add a tool** $\rightarrow$ select from:
* Power Platform Pre-built Connectors.
* Custom Prompts (AI Builder).
* Agent Flows (Power Automate).
* REST APIs / Custom Connectors (created via the Power Apps Maker Portal or linked to Azure Functions).
* **Model Context Protocol (MCP)**.


* **Agents Navigation (for Multi-Agent Systems):**
* **Agents** $\rightarrow$ **Add** $\rightarrow$ **Connect to an external agent** $\rightarrow$ **Agent to Agent (A2A)**.




* **Connecting to an MCP Server:**
* **Custom MCP Server Onboarding Parameters:**
* `Server Name`: Display and logical name of the target MCP server.
* `Description`: Functional explanation leveraged by the generative orchestrator for discoverability.
* `URL / Website`: Fully Qualified Domain Name (FQDN) or service endpoint hosting the protocol server.
* `Authentication Type`: Supports `None`, `API Key`, or `OAuth 2.0`.


* **Catalog of Pre-Built / Out-of-the-Box MCP Servers:**
* Microsoft Dataverse MCP
* Dynamics 365 MCP
* Microsoft Fabric MCP
* Office 365 Outlook MCP
* Kusto Query (Azure Data Explorer) MCP
* GitHub MCP
* LearnDocs MCP
* Box.com MCP
* Work IQ MCP Servers (e.g., *Email Management*)
* Jenny Actions




* **Tool Filtering, Auto-Opt-In & Scoping Controls:**
* **Allow All Tools Toggle (Default = On):** Automatically exposes every action offered by the MCP server to the agent.
* **Dynamic Auto-Inclusion:** When `Allow All` is enabled, any future tools, schemas, or actions deployed to that MCP server by the server administrator automatically become callable by the agent without re-authoring or manual intervention.
* **Granular Tool Filtering (Allow All = Off):** Allows developers to manually inspect and toggle specific tools on or off (e.g., disabling high-risk operational tools such as `Delete Email` while retaining `Get Email`, `List Email`, `Flag Email`, `Reply to Email`, `Send Email`, `Forward Email`).


* **Security & Execution Controls:**
* **Prompt for User Confirmation (Human-In-The-Loop):** Configurable under **Additional Details** to force the agent to request explicit user confirmation/approval before executing sensitive MCP tool actions.
* **Credential Source Configuration:** Toggle between `Maker provided credentials` and `End user credentials`.


* **Lifecycle Actions:**
* `Add and configure`: Commits the server registration.
* Context Menu (`...`): Exposes server removal via **Delete**.



---

## 3. Formulas, Expressions & Schemas

### MCP Tool Configuration & Delegation Schema (Conceptual Copilot Studio JSON)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "McpServerConfiguration",
  "type": "object",
  "properties": {
    "serverName": { "type": "string", "example": "WorkIQ-EmailManagement" },
    "endpointUrl": { "type": "string", "format": "uri", "example": "https://mcp.workiq.microsoft.com/v1/email" },
    "orchestrationEngine": { "type": "string", "enum": ["GenerativeOrchestration"], "default": "GenerativeOrchestration" },
    "authentication": {
      "type": "object",
      "properties": {
        "authType": { "type": "string", "enum": ["None", "ApiKey", "OAuth2"] },
        "credentialDelegation": { "type": "string", "enum": ["MakerProvided", "EndUser"] }
      },
      "required": ["authType", "credentialDelegation"]
    },
    "toolManagement": {
      "type": "object",
      "properties": {
        "allowAllTools": { "type": "boolean", "default": true },
        "toolOverrides": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "toolName": { "type": "string" },
              "isEnabled": { "type": "boolean" },
              "requireConfirmation": { "type": "boolean" }
            },
            "required": ["toolName", "isEnabled"]
          }
        }
      },
      "required": ["allowAllTools"]
    }
  },
  "required": ["serverName", "endpointUrl", "orchestrationEngine", "authentication", "toolManagement"]
}

```

### Tool Invocation Payload Example (Email Management MCP Server)

When an agent dynamically selects the `Send Email` action exposed by the MCP server:

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "sendEmail",
    "arguments": {
      "to": ["compliance@enterprise.com"],
      "subject": "Q4 Escalation Notice",
      "body": "The report has been approved and flagged for audit.",
      "importance": "High"
    }
  },
  "id": "req-mcp-00192"
}

```

### Power Fx: Conditional Human-in-the-Loop Confirmation Evaluation

When evaluating whether a destructive or privileged MCP tool requires interactive confirmation:

```powerfx
// Evaluates execution mode prior to delegating to an external tool
If(
    Topic.SelectedToolName = "deleteEmail" Or Topic.SelectedToolName = "forwardEmail",
    Set(Topic.RequiresExplicitConfirmation, true);
    Set(Topic.ConfirmationPrompt, "The agent is attempting to run: " & Topic.SelectedToolName & ". Do you authorize this action?"),
    Set(Topic.RequiresExplicitConfirmation, false)
)

```

---

## 4. Best Practices & Design Patterns

* **Principle of Least Privilege in Tool Registration:**
* Disable the default `Allow all tools` toggle on enterprise or multi-purpose MCP servers. Manually enable only the specific operational tools required for the agent's defined scope.
* Always deactivate destructive actions (e.g., `deleteEmail`, `dropTable`, `purgeRecord`) unless an explicit operational requirement exists and Human-in-the-Loop approval is enforced.


* **Orchestration Noise & Latency Minimization:**
* Every tool exposed via an MCP server adds its metadata schema to the generative model’s runtime context window. Exposing excessive, unused tools dilutes orchestrator reasoning, increases token consumption, increases response latency, and introduces tool-selection hallucinations.


* **Credential Scoping Strategy:**
* Use **End User Credentials** for any MCP server interacting with user-scoped resources (e.g., Office 365 Outlook, Teams, OneDrive) to enforce tenant boundary isolation and prevent privilege escalation.
* Use **Maker Provided Credentials** exclusively for shared system utilities or read-only reference stores (e.g., LearnDocs MCP, central knowledge databases) where individual end-user Entra ID licensing is not applicable.


* **Enterprise Action Auditing:**
* Route internal microservices through authenticated MCP servers rather than ad-hoc custom endpoints to capture standardized execution traces, identity contexts, and audit logs.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Incompatible with Classic Orchestration:**
* MCP tools cannot be bound or evaluated if the copilot is set to Classic Orchestration. The agent **must** have generative orchestration enabled.


* **Unintended Tool Injection via "Allow All":**
* If `Allow all tools` is left enabled, upstream modifications by the MCP server maintainer (such as publishing a newly developed diagnostic or administrative tool) instantly introduce that tool into the production agent's execution graph without pipeline promotion, review, or testing.


* **Authentication Mismatches:**
* When configuring OAuth 2.0 on custom MCP servers, missing redirect URIs or mismatched client/tenant ID scopes within Microsoft Entra ID will prevent connection hydration and cause silent execution failures during generative tool routing.


* **Architectural Misplacement (MCP vs. A2A):**
* Attempting to configure peer agent collaboration via the **Tools $\rightarrow$ MCP** menu will fail. Autonomous agent collaboration requires the **Agent to Agent (A2A)** framework configured under the **Agents** hub.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Architectural Distinction (MCP vs. A2A):**
* *Question:* "You need to integrate an external system of operational capabilities and actions into your copilot using an open standard protocol. Which feature should you configure?"
* **Correct:** **Model Context Protocol (MCP)** accessed via **Tools**.


* *Question:* "You need to enable a Copilot Studio agent to collaborate, negotiate, and delegate sub-tasks to an external, independently governed AI agent. Which framework must you use?"
* **Correct:** **Agent to Agent (A2A)** accessed via **Agents $\rightarrow$ Add $\rightarrow$ Connect to an external agent**.




* **Exam Trap — Prerequisites for MCP Usage:**
* MCP servers **cannot** be called from classic trigger-based topics using deterministic conditions alone. The agent must use **Generative Orchestration**.


* **Exam Trap — Automatic Tool Enrollment:**
* *Scenario:* A developer connects an MCP server with default settings. The administrator of the MCP server later deploys a new endpoint `archiveRecord`.
* *Exam Fact:* If `Allow all tools` is active, the agent **immediately gains access** to `archiveRecord` without requiring the agent maker to manually re-publish or re-add the tool.


* **Security Configuration Matrix:**
* *Maker-Provided Credentials:* Executes as the author; ignores the current conversational user's permission boundaries.
* *End-User Credentials:* Executes as the active conversational user; enforces per-user Entra ID Access Control Lists (ACLs).
* *Supported Auth Protocols for Custom MCP:* `None`, `API Key`, and `OAuth 2.0`.