## 1. Core Architecture & Concepts

* **Prerequisites for Analytics Ingestion:**
* **Publishing Requirement:** Analytics and metrics dashboards remain empty (`Never` under the *Last published* column) until the agent has been published.
* **Real User Traffic Dependency:** Session and transcript telemetry require active user interactions in published channels; internal canvas test runs do not write to production analytics.


* **Session Lifecycle & Boundaries:**
* **Standard Conversation Timeout:** A conversational session automatically concludes after **30 minutes of inactivity**.
* **Telephony Channel Boundary:** When using speech/telephony channels, a session ends **3 minutes** after the `End of Conversation` event.
* **Multi-Session Conversations:** A single continuous conversation between a user and an agent can span multiple distinct analytics sessions if interaction resumes after a session timeout.


* **Session States:**
* **Unengaged Session:** Default state when a session begins. Occurs when an agent initiates a greeting or proactive message, or a user initiates contact, but the user does not progress into custom business logic.
* **Engaged Session:** The user interacts beyond the initial greeting.
* *Classic Orchestration:* Triggered when a custom non-system topic, the `Escalate` topic, the `Fallback` topic, or the `Conversational boosting` topic is invoked.
* *Generative Orchestration:* Triggered when a user directly triggers an execution plan involving a non-system topic, `Escalate`, `Fallback`, a Knowledge Source query, or a Tool.




* **Session Outcomes (Engaged Sessions):**
* **Resolved:** The interaction completes successfully.
* *Resolved Confirmed:* The `End of Conversation` system topic is triggered and the user explicitly confirms their query was answered.
* *Resolved Implied:* In generative orchestration, the session times out while no active plans or open tasks are waiting for user input, or logic completes without explicit user survey confirmation.


* **Escalated:** Triggered when the `Escalate` system topic executes or a `Transfer to Agent` (human handoff) node runs. Can be categorized as *System intended* (exceeding business rule thresholds), *System unintended* (stuck in retries/errors), or *User requested*.
* **Abandoned:** An engaged session times out after 30 minutes without reaching either a *Resolved* or *Escalated* outcome.


* **Agent Architecture Telemetry Segregation:**
* **Conversational Agents:** Focus on session engagement rates, resolution/escalation/abandonment outcomes, generated answer quality, tools, and CSAT.
* **Autonomous Agents:** Focus on trigger execution success/failure rates, run durations, trigger error identification, and tool run metrics.
* **Classic Orchestration (Topic-Level Analytics):** Provides drill-down metrics at the specific topic level via topic-specific analytics panes.



---

## 2. Platform Features & Capabilities

* **Date Range Filtering & Scope:**
* **Available Filters:** Pre-built ranges of **7 days** (default), **14 days**, **30 days**, **90 days**, **180 days**, and **Custom date range**.
* **AI Summary Card:** Surfaces generative narrative insights regarding performance shifts, available exclusively when the selected date range is **90 days or less**.
* **Data Latency:** Analytics processing typically takes between **30 minutes to 1 hour** after a session concludes to appear on dashboards.


* **Transcript & Data Export (`Download Sessions`):**
* Accessible via the **Download sessions** button above the overview pane.
* Exports raw interaction data as a comma-separated values (`.csv`) file.
* Paginates into rows representing one-day increments (up to 50,000 sessions per export row).
* Retention window for raw transcripts within native Copilot Studio storage is up to **28 days** (requires the `Bot Transcript Viewer` Dataverse security role).


* **Conversational Agent Dashboard Sections:**
* **Overview Area:**
* *Conversational Sessions:* Total session volume split into engaged vs. unengaged sessions.
* *Engagement Rate:* Formulaic percentage of sessions where users engaged $\left(\frac{\text{Engaged Sessions}}{\text{Total Sessions}} \times 100\right)$.
* *Customer Satisfaction (CSAT):* Average score measured out of 5 stars derived from customer survey prompts.
* *Average DAU:* Average daily unique active users.
* *Copilot Credits Used:* Metric tracking compute and billing consumption across generative answers, tools, and workflows.
* *Context Menu (`...`):* Direct drill-downs into detailed billing meters and active user trends.


* **Effectiveness Area:**
* Displays session outcomes over time (*Resolved*, *Escalated*, *Abandoned*, and *Unengaged*) visualizable as stacked bar charts or area charts.
* *Agents & Sub-agents Card:* Breaks down child and connected sub-agent invocations, tracking total call volumes and percentage success rates.


* **Use (Answer Rate & Quality) Area:**
* *Generated Answer Rate:* Ratio of questions answered versus unanswered/declined.
* *Answer Quality:* Evaluates responses into **Good** vs. **Poor** ratings based on *Completeness*, *Relevance*, and *Groundedness*.
* *Poor Answer Breakdown:* Categorized into **Incomplete**, **Irrelevant**, or **Knowledge Not Used**.
* *Knowledge Source Use ("See details"):* Surfaces reasons for non-answers (e.g., *Decline to answer*), percentage of answers referencing specific knowledge sources over time, and knowledge retrieval error rates.


* **Tool Use Area (Conditional Rendering):**
* Dynamically surfaces only when tools/plugins (e.g., Power Automate flows, Power Platform connectors, MCP servers) are configured.
* Metrics: Total tool triggers, tool execution success rate (%), and run rankings for the top 5 tools over the period.


* **Satisfaction Area:**
* *Reactions:* Percentage distribution of positive (thumbs up) vs. negative (thumbs down) inline message responses.
* *Survey Results:* Aggregated CSAT score (1–5) and satisfaction categorization (*Dissatisfied*, *Neutral*, *Satisfied*).
* *Sentiment Analysis:* Natural Language Processing analysis rating conversational sentiment across user turns.




* **Autonomous Agent Dashboard Sections:**
* **Run Outcomes:** Metric charts showing total, successful, and failed execution runs alongside duration distribution maps over time.
* **Trigger Use:** Surfaces top 5 most frequently triggered workflows and top 5 triggers resulting in failed runs.
* **Tool Use:** Total tools triggered, percentage success rates, and volume charts for the top 5 tools over time.
* **Knowledge Source Use:** Ingestion frequency, error rates, overall answer rate, and top 5 knowledge sources over time.


* **Classic Orchestration Topic Analytics:**
* **Path:** **Topics** tab $\rightarrow$ select individual Topic $\rightarrow$ context menu (`...`) in top right corner $\rightarrow$ **Analytics**.
* **Filters:** Last 7, 14, 30, and 90 days.
* **Visualizations:** Total sessions associated with the topic (% of all agent topics), outcome distribution pie chart (*Resolved*, *Escalated*, *Abandoned*), outcome trends over time (bar/area charts), and topic-specific CSAT.



---

## 3. Formulas, Expressions & Schemas

### Downloaded Session Transcript Schema (`.csv`)

Exported session records from the Copilot Studio analytics canvas adhere to the following schema contract:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "CopilotStudioSessionTranscriptRecord",
  "type": "object",
  "properties": {
    "SessionID": { "type": "string", "description": "Unique GUID for the analytics session." },
    "StartDateTime": { "type": "string", "format": "date-time", "description": "Session initiation timestamp (UTC)." },
    "SessionOutcome": { "type": "string", "enum": ["Resolved", "Escalated", "Abandoned", "Unengaged"] },
    "OutcomeReason": { "type": "string", "description": "Diagnostic reason behind session outcome." },
    "IsResolvedImplied": { "type": "boolean", "description": "True if resolved via timeout/logic without user CSAT survey; False if confirmed via End of Conversation." },
    "Turns": { "type": "integer", "description": "Total count of user inputs and agent responses." },
    "ChatTranscript": { "type": "string", "description": "Serialized interaction formatted as 'User says: <msg>; Agent says: <resp>;'." },
    "InitialUserMessage": { "type": "string", "description": "First message string submitted by the user." },
    "TopicName": { "type": "string", "description": "Name of the last authored topic triggered in this session." },
    "TopicId": { "type": "string", "description": "Schema identifier/GUID of the last triggered topic." },
    "ChannelId": { "type": "string", "enum": ["msteams", "directline", "facebook", "telephony", "webchat"] },
    "CSAT": { "type": "integer", "minimum": 1, "maximum": 5, "description": "Survey response score." },
    "Comments": { "type": "string", "description": "User comments accompanying thumbs up/down reactions." }
  },
  "required": ["SessionID", "StartDateTime", "SessionOutcome", "Turns", "ChatTranscript"]
}

```

### Deterministic Topic Outcome Manipulation (YAML)

To explicitly flag a custom topic branch as resolved or escalated rather than allowing it to default to *Abandoned* upon session timeout, configure the `conversationOutcome` parameter in the topic's YAML definition:

```yaml
kind: AdaptiveDialog
beginDialog:
  kind: Sequence
  id: resolveIssueSequence
  operations:
    - kind: SendActivity
      id: sendSuccessMessage
      activity: "Your request has been processed successfully."

    # Explicitly setting confirmed resolution
    - kind: SetConversationOutcome
      id: setOutcomeResolved
      conversationOutcome: ResolvedConfirmed

    # Alternatively, invoking the End of Conversation System Topic
    - kind: BeginDialog
      id: triggerEndOfConversation
      dialog: cr_system_endOfConversation

```

### Power Fx: Dynamic Sentiment Branching Guardrail

When monitoring user dissatisfaction in real time within topics using system variables:

```powerfx
// Evaluate turn-level satisfaction score and route dynamically
If(
    System.Conversation.Sentiment.Score < 0.25,
    Redirect(Topic.Escalate),
    Set(Topic.ContinueProcessing, true)
)

```

---

## 4. Best Practices & Design Patterns

* **Mitigating High Abandonment Rates:**
* Sessions that end abruptly without routing to an `End of Conversation` or `Escalate` node default to an **Abandoned** outcome after 30 minutes of inactivity.
* *Design Pattern:* Always terminate custom conversational flows by explicitly redirecting to the `End of Conversation` system topic to trigger user confirmation and record a *Resolved Confirmed* state.


* **Knowledge Base Tuning from Quality Telemetry:**
* Monitor the **Answer Quality** dashboard. If high rates of *Knowledge Not Used* or *Incomplete* are recorded:
* Review semantic chunking and clean formatting within the source documents.
* Add explicit trigger instructions or conversational boosting examples to guide prompt context.
* Remove redundant, conflicting, or out-of-date files in SharePoint or Dataverse tables.




* **Tool Performance Auditing:**
* Regularly inspect the **Tool Use** pane. If a custom connector, MCP tool, or Power Automate flow shows a degrading success rate, inspect authentication token expirations, API throttling limits, or missing required parameters in the tool schema.


* **Handling Unengaged Sessions:**
* A high percentage of *Unengaged* sessions typically indicates that greeting messages do not clearly set user expectations. Refactor the `Conversation Start` topic to include actionable prompt buttons or suggested starters.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Export Retention Limits:**
* Downloaded session transcripts are stored within Copilot Studio infrastructure for a maximum of **28 days** before permanent purging. Organizations requiring multi-year audit compliance must implement pipeline exports from the Dataverse `ConversationTranscript` table to an Azure Data Lake or Log Analytics workspace.


* **SharePoint Knowledge Source Redaction:**
* When an agent grounds answers using **SharePoint** as a knowledge source, the generated answer is **redacted** in downloaded session transcript files (`search_results` contains the retrieved snippets, but the final output is labeled `REDACTED` due to enterprise tenant data compliance boundaries).


* **AI Summary Date Range Cap:**
* The AI-generated summary of key analytics does not render if the selected analytics filter exceeds **90 days**.


* **Security Role Prerequisites:**
* Makers cannot download session transcripts unless they are explicitly assigned the **Bot Transcript Viewer** security role in the target Microsoft Dataverse environment.


* **Dataverse for Teams & Microsoft Copilot Channel Limitations:**
* Transcripts are **not written** for agents deployed inside Microsoft Dataverse for Teams environments.
* Reactions collected from users interacting via the Microsoft Copilot channel are excluded from the *Reactions* column in the Themes and Knowledge source analytics panels.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Abandoned vs. Unengaged:**
* *Unengaged:* The user never submitted a message or triggered a custom topic/plan (only system greeting triggered).
* *Abandoned:* The user engaged with a topic/plan, but the conversation ceased for 30 minutes without hitting a resolution or escalation trigger.


* **Exam Trap — Inactivity Timeout Thresholds:**
* Standard messaging/chat channels: **30 minutes of inactivity** marks a session end.
* Telephony channels: **3 minutes** following an end-of-conversation event.


* **Exam Trap — Resolved Confirmed vs. Resolved Implied:**
* *Resolved Confirmed:* Requires the user to explicitly indicate "Yes" to the End of Conversation resolution survey.
* *Resolved Implied:* Occurs in Generative AI orchestration when a session times out with no pending plans awaiting input, or completes without explicit user survey submission.


* **Exam Trap — Topic Analytics Navigation Path:**
* Topic-specific analytics (session count, resolution rate per topic) are accessible for **Classic Orchestration** by opening the specific topic and clicking the **`...` (ellipsis)** in the top right corner, selecting **Analytics** (requires the topic and agent to be published).


* **Security Role Trap for Transcripts:**
* To export session files via the `Download sessions` button, having the *System Customizer* or *Environment Maker* role is insufficient; the user must hold the **Bot Transcript Viewer** security role.