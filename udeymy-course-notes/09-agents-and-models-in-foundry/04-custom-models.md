## 1. Core Architecture & Concepts

* **Custom Models via Fireworks Inference Runtime (Bring Your Own Weights - BYOW):**
* An alternative deployment architecture in Microsoft Foundry that enables developers to import, register, and serve custom open-weight models, proprietary weights, or specialized LoRA adapters directly on high-performance Fireworks AI inference infrastructure.
* **Day-Zero Open Source Availability:** Provides immediate access to cutting-edge open-source models (e.g., DeepSeek, Kimi, GLM, Qwen, OpenAI open models) prior to their broad general availability as first-party managed models in Microsoft Foundry.


* **Separation of Planes (Foundry vs. Fireworks):**
* *Control & Governance Plane (Microsoft Foundry):* Governs identity via Microsoft Entra ID, Azure role-based access control (Azure RBAC), tenant audit logging, and Azure billing/consumption (MACC-eligible).
* *Data & Execution Plane (Fireworks AI):* Dedicated GPU clusters running the Fireworks inference engine and compilation optimizations (such as FireOptimizer) tuned per model architecture for low latency and high token throughput.


* **Weight Types Supported:**
* *Full Weight Model:* Direct deployment of entire uncompressed or quantized model parameter tensors.
* *LoRA Adapters (Preview):* Parameter-efficient fine-tuned layers that are merged automatically into base weights by the runtime for deterministic serving performance.
* *Draft Models (Preview):* Speculative decoding configurations using lightweight companion models to accelerate token generation speeds of larger target models.


* **Required Model Directory File Taxonomy:**
To successfully import and register a model asset, the local directory must contain an exact set of foundational artifacts:
* `config.json`: Hardware, architectural, and hyperparameter configuration metadata.
* `*.safetensors` or `*.bin`: Serialized model weight tensors (strongly preferring Safetensors for memory-mapped security and fast deserialization).
* `*model.safetensors.index.json` or `*.index.json`: Weight sharding maps indexing parameter boundaries across multi-part weight files.
* `tokenizer*`: Tokenizer definition and vocabulary files (e.g., `tokenizer.json`, `tokenizer_config.json`, `special_tokens_map.json`).


* **Supported Base Model Architectures:**
* Pre-validated model families include **DeepSeek**, **Kimi**, **GLM**, **OpenAI (e.g., gpt-oss)**, and **Qwen** (Chuen).



---

## 2. Platform Features & Capabilities

* **Navigation & Onboarding Workflow in Microsoft Foundry:**
1. Navigate to **Microsoft Foundry** (`ai.azure.com`) $\rightarrow$ select project.
2. Open **Build** $\rightarrow$ **Models** $\rightarrow$ **Custom models** tab.
3. Click **Add custom model** (or **Generate upload command**).


* **Model Configuration Parameters:**
* *Base Model / Architecture:* Select the corresponding target architecture (e.g., *DeepSeek*, *Qwen*, *Kimi*).
* *Model Details:* Define human-readable display name, unique model asset name, and version identifier.
* *Weight Type:* Choose between `Full weight model`, `LoRA adapter (preview)`, or `Draft model (preview)`.
* *Local Model Path:* Enter the absolute local folder directory containing the compiled weight artifacts.


* **Azure Developer CLI (`azd`) Tooling:**
* Foundry does not upload heavy weights through browser forms; it generates a target **Azure Developer CLI** (`azd`) upload command with the `azure.ai.models` extension.
* The command transfers weights directly from local storage/workstations to the underlying Azure/Fireworks storage endpoint.
* Model status transitions to **Registered** once upload and manifest hashing conclude.


* **Capacity & Scaling Provisioning:**
* **Provisioned Throughput Units (PTU):** Deployed using dedicated GPU capacity allocations (PTU-backed) to guarantee stable latency, predictable tokens per minute (TPM), and high-volume concurrent agent scaling without regional noisy-neighbor interference.
* **Deployment Endpoint:** Produces an OpenAI-compatible REST endpoint (Chat Completions and Responses API) callable by custom connectors or downstream agents.



---

## 3. Formulas, Expressions & Schemas

### Azure Developer CLI (`azd`) Upload Command Generation

```bash
# 1. Install required AI models extension for Azure Developer CLI
azd ext install azure.ai.models

# 2. Authenticate CLI session against the target Azure subscription/tenant
azd auth login

# 3. Execute generated model upload command (pointing to local directory)
azd ai models create \
  --name "custom-deepseek-enterprise" \
  --version "1.0.0" \
  --base-model "deepseek-v3" \
  --weight-type "full" \
  --source "/path/to/local/model_directory"

```

### Model Directory Configuration Schema (`config.json`)

```json
{
  "architectures": [
    "DeepseekV3ForCausalLM"
  ],
  "model_type": "deepseek_v3",
  "vocab_size": 129280,
  "hidden_size": 4096,
  "intermediate_size": 11008,
  "num_attention_heads": 32,
  "num_hidden_layers": 30,
  "torch_dtype": "bfloat16",
  "transformers_version": "4.48.0"
}

```

### Weight Index Manifest Schema (`model.safetensors.index.json`)

```json
{
  "metadata": {
    "total_size": 32212254720
  },
  "weight_map": {
    "model.embed_tokens.weight": "model-00001-of-00004.safetensors",
    "model.layers.0.self_attn.q_proj.weight": "model-00001-of-00004.safetensors",
    "model.layers.29.mlp.down_proj.weight": "model-00004-of-00004.safetensors",
    "lm_head.weight": "model-00004-of-00004.safetensors"
  }
}

```

### Power Fx: Calling Deployed Custom Model Endpoint from Copilot Studio

When connecting Copilot Studio topics to a custom model deployed via Fireworks on Foundry through an authenticated custom connector:

```powerfx
// Step 1: Formulate OpenAI-compatible chat payload targeting the custom deployment
Set(Topic.InferencePayload, {
    model: "custom-deepseek-enterprise",
    messages: Table(
        { role: "system", content: "You are an enterprise reasoning agent running on custom infrastructure." },
        { role: "user", content: Topic.UserPrompt }
    ),
    temperature: 0.2,
    max_tokens: 1024
});

// Step 2: Invoke Custom Connector mapped to the Foundry/Fireworks runtime
Set(Topic.CustomModelResponse, 
    FoundryCustomModelConnector.CreateChatCompletion(Topic.InferencePayload)
);

// Step 3: Extract and validate completion text
Set(Topic.GeneratedAnswer, 
    First(Topic.CustomModelResponse.choices).message.content
);

```

---

## 4. Best Practices & Design Patterns

* **File Integrity & Pre-Flight Validation:**
* Verify that `config.json`, weight shards (`.safetensors`), weight map (`.index.json`), and all `tokenizer*` configuration files are present in the directory before initiating CLI uploads. Missing tokenizer or sharding files will fail server-side asset validation in Foundry.


* **Format Selection (`.safetensors` vs. `.bin`):**
* Prioritize `.safetensors` over legacy Python pickle-based `.bin` files. Safetensors prevent arbitrary code execution vulnerabilities, allow zero-copy memory mapping, and significantly accelerate model loading times on GPU clusters.


* **Throughput Sizing (Provisioned Throughput Units):**
* Deploy custom models using **Provisioned Throughput Units (PTU)** for enterprise workloads with low-latency SLAs. Unlike multi-tenant pay-per-token models that may throttle during peak hours, PTU guarantees dedicated GPU availability, stable latency, and deterministic throughput.


* **Copilot Studio Front-End Decoupling:**
* Expose the deployed custom model as a **Custom Connector** or an **Agent Flow** tool in Copilot Studio.
* Wrap the endpoint in Dataverse Environment Variables (Base URL, Deployment Name) to ensure smooth ALM migration between Development, Test, and Production environments without modifying topic schemas.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Strict Regional Availability Constraints:**
* Custom Models powered by the Fireworks inference runtime are restricted to **six US regions**.
* *Unsupported Regions:* European, Asian, and UK Azure regions cannot host or run these deployments.


* **Exclusion from EU Data Boundary:**
* Fireworks inference on Microsoft Foundry is **explicitly excluded from EU Data Boundary commitments**. Organizations governed by strict European data residency or GDPR localization mandates cannot route production customer data through this runtime.


* **Compliance Boundaries (FedRAMP):**
* Fireworks on Foundry is currently not certified for FedRAMP high-impact levels. Government and defense sector tenants must consult Authorization Officials prior to use.


* **Upload Tooling Requirement:**
* Model weights cannot be uploaded via drag-and-drop within the Azure web portal. Uploads require the **Azure Developer CLI (`azd`)** with the specialized `azure.ai.models` extension.


* **Upload Bandwidth & Transfer Durations:**
* Uploading large model weights (spanning tens to hundreds of gigabytes) requires significant network bandwidth and can take hours over standard broadband connections. Upload drops can invalidate partial manifests, requiring command restarts.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Fine-Tuning vs. Custom Models (BYOW):**
* *Fine-Tuning:* Adapts an existing base model’s weights using training datasets (`.jsonl`) executed directly within Foundry compute.
* *Custom Models (Fireworks BYOW):* Imports pre-existing, externally trained, or newly open-sourced weight files (`.safetensors`, `.bin`, `config.json`) and runs them on optimized third-party inference infrastructure.


* **Exam Trap — Deployment Scaling Mechanism:**
* Scaling custom models to handle high concurrency and provide predictable response times requires configuring **Provisioned Throughput Units (PTUs)**.


* **Exam Trap — Required Artifacts for Custom Model Upload:**
* An exam question asking for mandatory files to register a custom model requires:
1. `config.json` (Architecture and hyperparameters)
2. `*.safetensors` or `*.bin` (Model weights)
3. `*.index.json` (Weight shard mappings)
4. `tokenizer*` files (Tokenizer vocabulary and configuration)




* **Exam Trap — Geographic & Data Residency Constraints:**
* If a scenario requires compliance with the **EU Data Boundary** or strict **European regional residency**, deploying custom models via Fireworks **violates this requirement** and is unsupported.


* **Upload Method Trap:**
* Custom model weights are uploaded via the **Azure Developer CLI (`azd`)**, NOT the standard Power Platform CLI (`pac`), NOT AzCopy, and NOT via browser upload in the Foundry web UI.