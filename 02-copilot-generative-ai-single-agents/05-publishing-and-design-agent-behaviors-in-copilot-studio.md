## 1. Core Architecture & Concepts

* **Channel Distribution Boundary:** Copilot Studio agents are authored centrally in Dataverse environments but published to independent client endpoints known as **Channels**. Channels dictate the rendering capabilities (plain text, rich markdown, adaptive cards, quick replies, or voice/audio streams).
* **Channel-to-Authentication Coupling:**
* The selected authentication profile acts as a hard security gate for channel availability.
* Setting authentication to **Authenticate with Microsoft** limits channel publication strictly to internal Microsoft ecosystem endpoints (**Microsoft Teams**, **Microsoft 365 Copilot**, **SharePoint**, and **Power Apps**).
* Exposing agents to external consumer channels (Facebook, WhatsApp, public websites, custom mobile apps, Twilio) requires changing the authentication profile to **No authentication** or **Authenticate manually**.


* **Authentication Models:**
* **No Authentication:** Anonymous access. Anyone possessing the endpoint URL or embed snippet can interact with the agent.
* **Authenticate with Microsoft:** Uses tenant-level Microsoft Entra ID (Azure AD). Enforces organizational single sign-on (SSO), grants access to Entra ID user identity claims, and restricts bot interactions to authorized tenant users.
* **Authenticate Manually:** OAuth 2.0 configuration connecting to external identity providers (Entra ID custom app registration, generic OAuth2, OpenID Connect, GitHub, Google, etc.).


* **Suggested Prompts / Conversation Starters:** Dynamic sample prompts displayed to users inside Microsoft Teams and Microsoft 365 Copilot chat interfaces upon initiating a session, priming user intent before typing.
* **Telephony & Speech (Voice-First Architecture):**
* **Speech Synthesis & Recognition Modality:** Allows decoupling visual text output from spoken responses (SSML/voice prompts) within the same node structure.
* **DTMF (Dual-Tone Multi-Frequency):** Standardized telephone keypad signaling allowing touch-tone interaction (0–9, *, #) alongside or in place of voice input.
* **Acoustic & Session Timing Engines:** Built-in speech recognition timers governing barge-in, background silence, keypad pause tolerance, and latency masking during long-running backend processes.



---

## 2. Platform Features & Capabilities

* **Security & Authentication Navigation:**
* Path: **Settings** $\rightarrow$ **Security** $\rightarrow$ **Authentication** (or direct hyperlink from blocked channels view).
* Modes available:
* *No authentication*
* *Authenticate with Microsoft* (Default for Teams/M365)
* *Authenticate manually*




* **System Identity Variables (Resolved via Authentication):**
* When authentication is configured (*Authenticate with Microsoft* or *Authenticate manually*), runtime user identity claims populate system variables:
* `System.User.DisplayName`
* `System.User.Email`
* `System.User.FirstName`
* `System.User.Id`
* `System.User.IsLoggedIn`




* **Supported Channel Ecosystem:**
* **Microsoft-First Channels:** Microsoft Teams, Microsoft 365 Copilot (unified toggle under *Teams and Microsoft 365 Copilot*), SharePoint, Power Apps.
* **Testing / Staging Channels:** Demo website (provides a shareable, password-free URL with custom welcome text and conversation starters for stakeholder validation; strictly non-production).
* **External & Consumer Messaging Channels:** Web app (custom iframe / web chat control), Native app (Direct Line SDK / mobile wrapper), Facebook Messenger (requires Facebook App ID, App Secret, and Page Token), WhatsApp.
* **Azure Bot Service & Telephony Channels:** Slack, Telegram, Twilio, Line, GroupMe, Direct Line Speech, Email, Customer Engagement Hubs (e.g., Dynamics 365 Omnichannel for Customer Service).


* **Suggested Prompts Configuration:**
* Path: **Overview** $\rightarrow$ scroll to **Suggested prompts** $\rightarrow$ **+ Add suggested prompts**.
* Up to multiple prompt pairings configured with a title and full natural language question:
* *Total allowance:* "What is my total leave allowance?"
* *Sick leave allowance:* "How much sick leave allowance do I have?"
* *Vacation allowance:* "What vacation allowance do I have over a year?"




* **Voice & Telephony Configuration:**
* Path: **Settings** $\rightarrow$ **Voice** $\rightarrow$ toggle **Optimize for voice**.
* Configurable parameters:
* *Primary authoring mode:* Switch default authoring focus to voice/speech design.
* *Increase accuracy with agent data:* Feeds domain entities, trigger phrases, and vocabulary directly into the speech-to-text recognition language model to boost phoneme accuracy.
* *DTMF controls:*
* **Interdigit timeout:** Maximum allowable pause (in seconds/milliseconds) between dialed keypad numbers before input evaluation triggers.
* **Termination timeout:** Duration to wait for the designated termination key (e.g., `#`).


* *Silence detection timeout:* Duration of audio silence before the agent assumes user input is complete or absent.
* *Speech collection timers:*
* **Utterance end timeout:** Time elapsed following user vocal pause before speech-to-text stops listening and submits the utterance.
* **Speech recognition timeout:** Threshold for processing vocal input; if exceeded without speech recognition, triggers reprompt logic.


* *Latency messaging:* Automated interim audio messages (e.g., "Please wait while I check that...") played during long-running API, flow, or search executions.




* **Canvas Node Modality Toggle:**
* Inside *Send a message* or *Ask a question* nodes, switch between **Text** and **Speech** tabs to provide alternate audio scripts for IVR channels.



---

## 3. Formulas, Expressions & Schemas

### Accessing Authenticated Identity Claims (Power Fx)

When an agent runs under Entra ID (*Authenticate with Microsoft*) or manual OAuth, system identity variables are accessed directly in message nodes, conditions, or expressions:

```powerfx
// Greeting referencing authenticated system variables
Concatenate(
    "Hello ", 
    If(IsBlank(System.User.FirstName), System.User.DisplayName, System.User.FirstName), 
    "! Your authenticated account email is: ", 
    System.User.Email
)

```

### Channel State Detection (Power Fx)

To conditionally adapt message formatting or branch logic depending on whether the conversation runs on Microsoft Teams, Web Chat, or Telephony:

```powerfx
// Evaluate active channel host
Switch(
    System.Activity.ChannelId,
    "msteams", "You are chatting via Microsoft Teams.",
    "directline", "You are accessing via Web Chat / Native App.",
    "telephony", "You are connected via telephone voice line.",
    "Connected via other channel."
)

```

### Telephony Dual Modality & DTMF Schema (Bot Component YAML)

Declarative YAML showing dual-mode text/speech rendering and voice settings:

```yaml
kind: AdaptiveDialog
beginDialog:
  kind: OnRecognizedIntent
  id: main
  actions:
    - kind: SendMessage
      id: send_allowance_message
      message:
        # Standard chat visual output
        text: You have **29 vacation days** and **5 sick days** remaining.
        # Spoken audio script override (SSML-compliant)
        speak: You have twenty-nine vacation days and five sick days remaining this calendar year.

    - kind: Question
      id: ask_menu_selection
      prompt:
        text: Press 1 for Vacation Leave, or Press 2 for Sick Leave.
        speak: To check vacation leave, press one. To check sick leave, press two.
      entity: StringPrebuiltEntity
      variable: Topic.UserSelection
      dtmfSettings:
        interDigitTimeoutInSeconds: 3
        terminationTimeoutInSeconds: 5
        terminationCharacter: "#"

```

### External Channel Webhook Binding Payload (Facebook Messenger Example)

```json
{
  "channelName": "Facebook",
  "appId": "123456789012345",
  "appSecret": "[ENCRYPTED_VAULT_SECRET]",
  "pageAccessToken": "[EXTENDED_PAGE_ACCESS_TOKEN]",
  "validationToken": "RandomSecurityToken123!"
}

```

---

## 4. Best Practices & Design Patterns

* **Data Leakage Mitigation with Public Channels:**
* When configuring **No authentication**, audit all attached knowledge sources (SharePoint, internal files, enterprise Dataverse tables). If public users access the agent, enterprise-internal data indexed by generative knowledge becomes queryable by anyone on the internet.
* Never connect internal SharePoint document libraries to an agent published on an unauthenticated public channel.


* **Dual-Modality Scripting (Voice vs. Chat):**
* Avoid raw markdown tables, URLs, complex abbreviations, or dense bulleted lists in speech output.
* When configuring voice/telephony, explicitly fill the **Speech** tab on message nodes: expand acronyms into phonetic words, write out numerical quantities ("twenty-nine" instead of "29"), and omit visual markdown decorators (`**`, `*`, `[link](...)`).


* **Latency Management for Telephony / IVR:**
* Telephony users perceive latency much more acutely than chat users. Always configure **Latency messaging** in Voice settings for any action that queries Power Automate flows, REST connectors, or generative knowledge searches exceeding 2 seconds.


* **Demo Website vs. Production Web Hosting:**
* Use the built-in **Demo website** strictly for internal SME testing, design sign-offs, and persona validation.
* For production web deployments, build a production web container hosting the **Custom Web App** or Power Pages channel with appropriate security headers, rate limiting, and authenticated user tokens.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Channel Lockout under "Authenticate with Microsoft":**
* If the agent is set to *Authenticate with Microsoft*, external consumer channels (Web, Facebook, WhatsApp, Twilio, Slack) are completely locked out and grayed out. The studio enforces this to prevent passing Entra ID tokens to unsecured external channels.


* **Test Canvas Voice Emulation Restriction:**
* While the test canvas allows switching the chat mode view dropdown to speech formatting, **speech audio cannot be tested natively in the built-in test canvas**. True audio playback, voice recognition, and DTMF inputs require publishing to a configured telephony channel or Azure Direct Line Speech endpoint.


* **DTMF Input Collisions:**
* When DTMF is enabled, if the **Interdigit timeout** is set too low (e.g., under 1.5 seconds), slow callers entering multi-digit employee IDs or ticket numbers will trigger premature evaluations, resulting in repeated input failures.


* **Suggested Prompts Channel Scope:**
* Suggested prompts configured on the Overview tab do **not** render on generic web chat, Facebook, or custom mobile apps; their display is strictly constrained to **Microsoft Teams** and **Microsoft 365 Copilot** canvas hosts.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Unlocking External Channels:**
* *Question Scenario:* An architect builds an agent for Teams and now wants to deploy the exact same bot to a public customer website and Facebook Messenger, but the channels are disabled/unavailable in the portal.
* *Root Cause:* The agent is configured with *Authenticate with Microsoft*.
* *Resolution:* The developer must navigate to **Settings** $\rightarrow$ **Security** $\rightarrow$ **Authentication** and change the mode to **No authentication** (for anonymous access) or **Authenticate manually** (OAuth2/Entra ID custom app registration).


* **Exam Trap — System.User Claim Population:**
* `System.User.DisplayName`, `System.User.Email`, and `System.User.Id` are **only populated** when authentication is active (*Authenticate with Microsoft* or *Authenticate manually*). Under *No authentication*, these variables are empty/blank at runtime.


* **Exam Trap — DTMF Parameter Definitions:**
* **Interdigit timeout:** The maximum delay allowed *between* individual keypresses before evaluating the input string.
* **Termination timeout:** The duration the system waits for the user to press the termination character (e.g., `#` or `*`).


* **Exam Trap — Voice Accuracy Optimization:**
* To improve recognition of domain-specific terminology, specialized product codes, and jargon over telephony, enable **Increase accuracy with agent data** under **Voice settings**. This automatically compiles the agent’s trigger phrases, topics, and glossary into the acoustic speech-to-text model.