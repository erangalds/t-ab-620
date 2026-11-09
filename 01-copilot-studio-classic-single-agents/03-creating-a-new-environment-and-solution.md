## 1. Core Architecture & Concepts

* **Power Platform Environment Architecture:**
* An **Environment** is a dedicated boundary container for Power Platform assets (Copilot Studio agents, Power Apps, Power Automate cloud flows, connection references, environment variables, and Dataverse tables).
* Acts as an administrative, physical, identity, data storage, and geographic data-residency boundary.


* **Environment Types & Operational Scopes:**
* **Default Environment:** Automatically created per Entra ID tenant; named after the tenant/organization. Accessible by all licensed users in the tenant. Intended solely for personal productivity, experimentation, and ad-hoc exploration. Lacks dedicated point-in-time backup/restore guarantees and is unsupported for enterprise or production ALM workloads.
* **Developer Environment:** Provisioned under the Power Apps Developer Plan. Dedicated to single-user development and prototyping (owner-only access).
* **Sandbox Environment:** Dedicated non-production environment designed for isolated development, multi-user system integration testing (SIT), and user acceptance testing (UAT). Supports administrative resets, point-in-time restores, and environment copy operations.
* **Production Environment:** Intended for permanent, hardened, end-user business workloads; requires at least **1 GB of available Dataverse database capacity** across the tenant to provision.
* **Trial (Standard) Environment:** 30-day temporary container designed for short-term evaluation; automatically expires and deprovisions after 30 days unless converted or backed by paid capacity.


* **Solutions & ALM Architecture:**
* A **Solution** is the fundamental package/carrier mechanism in Dataverse for Application Lifecycle Management (ALM). It bundles agent components, topics, entities, actions, and bot dependencies so they can be exported (as Unmanaged or Managed) and promoted across dev, test, and production environments.
* **System Solutions:**
* *Default Solution:* Automatically contains all custom and uncustomized system components residing within the environment.
* *Common Data Services Default Solution:* Default container for ad-hoc custom additions made outside a dedicated custom solution.


* **Preferred Solution:** An environment-level configuration setting that automatically routes any newly authored components (e.g., agents created directly in the studio UI) into the designated solution rather than dumping them into the Common Data Services Default Solution.


* **Solution Publisher & Prefixing:**
* A **Publisher** establishes identity, ownership, and namespace isolation for solution assets.
* **Customization Prefix:** A short alphanumeric string (e.g., `ab100_`) appended to the logical schema names of all components (tables, columns, bot sub-components, actions) authored within that publisher's solutions.
* Prevents naming collisions between multiple third-party solutions, ISV packages, and internal enterprise modules.



---

## 2. Platform Features & Capabilities

* **Power Platform Admin Center (PPAC) Navigation:**
* URL: `admin.powerplatform.microsoft.com`
* Path to Provision: **Settings (gear icon)** in Copilot Studio / Power Apps $\rightarrow$ **Power Platform admin center** $\rightarrow$ Left navigation: **Manage** $\rightarrow$ **Environments** $\rightarrow$ **New**.


* **Environment Provisioning Parameters:**
* **Name:** Display identifier (e.g., `ab-100 dev`).
* **Managed Environment Toggle:** Optional governance/governed controls layer (left off for standard dev containers).
* **Environment Group:** Logical grouping construct for bulk governance rules and shared policy application across environments (`None` selected).
* **Region:** Geographic datacenter hosting the database and services (e.g., `United States`, `United Kingdom`). Enforces data sovereignty and dictates access to geo-restricted preview features (such as experimental agent computer-use or preview AI models).
* **Get new features early (Preview Cadence):** Toggle to opt the environment into early-release/targeted preview waves.
* **Type:** Selection between `Trial`, `Developer`, `Sandbox`, or `Production`.
* **Add a Dataverse data store:** Mandatory toggle for Copilot Studio agent persistence; provisions the underlying relational schema and system metadata store.
* **Pay-as-you-go with Azure:** Links environment consumption (Power Apps, Power Automate, Copilot Studio overages) directly to an Azure Subscription ID via an Azure Synapse/Azure Resource Manager meter without upfront per-user licenses.


* **Dataverse Database Configuration Settings:**
* **Language:** Base language of Dataverse schema metadata (e.g., English).
* **Currency:** Base/base-reporting currency for monetary fields.
* **Security Group:** Entra ID (Azure AD) security group assignment. Setting to `None (Open access)` allows any tenant user with an assigned environment role to connect; assigning a security group restricts environment access exclusively to group members.
* **Enable Dynamics 365 apps:** Toggles installation of CRM/ERP core solutions (disabled for pure low-code dev).
* **Deploy sample apps and data:** Controls provisioning of sample tables and data.


* **Copilot Studio Environment Switching:**
* Top navigation header: **Environment Picker** dropdown displays active tenant environments. Refreshing the browser registers newly provisioned environments.


* **Solution Authoring & Preferred Solution Configuration:**
* Navigation: Copilot Studio left rail $\rightarrow$ **... (More)** $\rightarrow$ **Solutions** $\rightarrow$ **New solution**.
* **Publisher Configuration Parameters:**
* *Display Name:* Friendly publisher name (e.g., `Philip Burton`).
* *Name (Unique Logical Name):* Must contain only alphanumeric characters and underscores (`_`); spaces and special characters trigger validation errors.
* *Prefix:* Namespace identifier (e.g., `ab100`).
* *Choice Value Prefix:* Integer prefix automatically assigned to custom option sets/choice values (e.g., `10000`) to guarantee global uniqueness.
* *Contact details:* Optional publisher metadata.


* **Version Format:** Follows semantic versioning format `Major.Minor.Build.Revision` (initial default: `1.0.0.0`).
* **Set as preferred solution:** Checkbox or contextual action button (`Set preferred solution`) that binds the active maker context to that solution container.



---

## 3. Formulas, Expressions & Schemas

### Publisher & Solution Dataverse Metadata Schema

When a solution and publisher are instantiated, Dataverse registers the definition in its system solution catalog:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "DataverseSolutionPayload",
  "type": "object",
  "properties": {
    "solution": {
      "type": "object",
      "properties": {
        "uniquename": { "type": "string", "example": "ab100_dev_solution" },
        "friendlyname": { "type": "string", "example": "ab-100 dev solution" },
        "version": { "type": "string", "pattern": "^[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+$", "example": "1.0.0.0" },
        "ispreferred": { "type": "boolean", "default": true },
        "ismanaged": { "type": "boolean", "default": false }
      },
      "required": ["uniquename", "friendlyname", "version"]
    },
    "publisher": {
      "type": "object",
      "properties": {
        "uniquename": { "type": "string", "pattern": "^[a-zA-Z0-9_]+$", "example": "PhilipBurton" },
        "friendlyname": { "type": "string", "example": "Philip Burton" },
        "customizationprefix": { "type": "string", "maxLength": 8, "example": "ab100" },
        "customizationoptionvalueprefix": { "type": "integer", "example": 10000 }
      },
      "required": ["uniquename", "friendlyname", "customizationprefix", "customizationoptionvalueprefix"]
    }
  }
}

```

### Power Fx: Evaluation of Environment Capacity & Storage Verification

To programmatically check whether tenant capacity supports provisioning a production environment (requiring $\ge 1\text{ GB}$), an administrator can evaluate environment metrics:

```powerfx
// Evaluates environment capacity adequacy before triggering provisioning
If(
    TenantAvailableDatabaseCapacityGB >= 1.00,
    Set(varCanProvisionProduction, true),
    Set(varCanProvisionProduction, false)
);

// Evaluates solution component naming convention against Publisher Prefix
If(
    StartsWith(ComponentLogicalName, Lower(PublisherPrefix) & "_"),
    Notify("Component conforms to ALM publisher prefixing standard", NotificationType.Success),
    Notify("Warning: Component is in the Default Solution or non-prefixed publisher!", NotificationType.Warning)
)

```

---

## 4. Best Practices & Design Patterns

* **Strict ALM Segregation:**
* **Never build in the Default Environment:** The Default environment cannot be isolated, backed up on a custom schedule, or placed behind strict security groups; it lacks ALM boundaries.
* Maintain at minimum a three-tier or two-tier environment architecture:
* **Development (Dev):** Unmanaged solutions; active development by builders.
* **Test / UAT:** Managed solutions deployed via automated pipelines for validation.
* **Production (Prod):** Managed solutions only; strictly isolated from direct maker edits.




* **Preferred Solution Governance:**
* Always configure a **Preferred Solution** in development environments immediately after provisioning. This ensures that agents created via the Copilot Studio UI canvas are automatically scoped to a custom publisher prefix rather than defaulting to `crXXX_` under the Common Data Services Default Solution.


* **Geographic Feature Alignment:**
* Deploy development environments into the **United States (US)** region when testing preview or cutting-edge agent capabilities (such as computer-use actions or advanced model features), provided corporate data residency policies permit non-production test data outside the domestic region. For production, strictly align environment region to compliance, legal, and data sovereignty boundaries.


* **Security Group Hardening:**
* In non-test enterprise environments, restrict access by binding a dedicated Microsoft Entra ID Security Group to the environment rather than leaving it open to the entire tenant (`None`).



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Publisher Naming Restrictions:**
* **Unique Name Validation:** Publisher unique names strictly accept letters, numbers, and underscores (`_`). Spaces, dashes (`-`), and special characters cause immediate validation failure (`invalid character in unique publisher name`).


* **Production Environment Prerequisites:**
* Creating a Production or Sandbox environment requires at least **1 GB of available Dataverse database capacity** remaining in the tenant capacity pool. If the tenant is at or over capacity, environment creation is blocked until capacity is added or freed.


* **Copilot Studio Persistence Dependency:**
* An agent **cannot exist** in an environment without a Dataverse database. Selecting "No" to *Add a Dataverse data store* during environment creation makes the environment unavailable for Copilot Studio authoring.


* **Studio Environment Refresh Delay:**
* New environments provisioned in the Power Platform admin center do not immediately populate the active Copilot Studio web session. The browser tab must be refreshed to pull the updated environment directory from the tenant discovery service.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Default Environment Capabilities:**
* *Trap:* A scenario asks where to build an enterprise-grade agent with automated pipelines and custom backup requirements.
* *Fact:* The **Default environment cannot be deleted, cannot be backed up/restored independently, cannot have its security restricted to a specific security group, and should never host production workloads.**


* **Exam Trap — Preferred Solution vs. Default Solution:**
* If a developer builds an agent directly in Copilot Studio without setting a preferred solution, the agent assets end up in the **Common Data Services Default Solution** with the default prefix (`cr...` or `new...`).
* To ensure automated packaging and clean export, set the custom solution as the **Preferred Solution**.


* **Exam Trap — Solution Component Types:**
* Copilot Studio agents are fully solution-aware Dataverse components. When moving agents across environments, you export the containing **Solution** (as Managed for downstream environments), not individual agent files.


* **Exam Trap — Pay-As-You-Go vs. Subscriptions:**
* Linking an Azure Subscription to an environment enables **Pay-As-You-Go** for Power Platform consumption, eliminating the need to assign upfront user or per-app capacity licenses to the environment beforehand.