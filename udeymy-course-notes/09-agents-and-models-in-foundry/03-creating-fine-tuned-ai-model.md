## 1. Core Architecture & Concepts

* **Model Fine-Tuning:**
* The algorithmic process of taking an existing pre-trained base model and adapting its internal parameters (weights) on a targeted dataset.
* Unlike in-context prompt engineering or Retrieval-Augmented Generation (RAG), fine-tuning modifies the model's weight representations directly.
* **Core Architectural Drivers:**
* *Reducing Prompt Overhead & Token Cost:* Embeds few-shot learning and domain examples directly into model weights, eliminating the need to pass multi-turn demonstration examples in every prompt payload.
* *Style, Tone, and Schema Consistency:* Enforces strict JSON schemas, specific markdown structures, or proprietary corporate response styles reliably.
* *Tool Execution & Grounding Alignment:* Conditions the model on domain-specific function-calling schemas, API definitions, or proprietary retrieval indexes.
* *Context Window Optimization:* Overcomes conversation context window limits by training across thousands of domain examples that cannot fit into runtime prompt windows.
* *Latency Optimization:* Speeds up inference by reducing input tokens and enabling smaller models to achieve output quality comparable to larger models.




* **Model Distillation:**
* A specialized transfer-learning technique where knowledge, reasoning patterns, and tool-invocation behaviors from a larger, high-capability teacher model (e.g., GPT-4.1) are systematically transferred to train a smaller, distilled student model (e.g., GPT-4.1-mini / GPT-4.1-nano).
* Yields significant inference cost reductions and lower request latency while maintaining task-specific accuracy.
* *Constraint:* Distillation is not supported across all base models; both the teacher and student architectures must support the distillation framework.


* **Low-Rank Adaptation (LoRA):**
* Parameter-Efficient Fine-Tuning (PEFT) mechanism ($L\text{o}RA$) that freezes the pre-trained model’s base weight matrices ($W_0$) and injects trainable rank decomposition matrices into each layer.
* Dramatically reduces trainable parameter complexity, memory footprints, and compute requirements during training without degrading task performance.


* **Decision Matrix (When to Fine-Tune vs. When to Avoid):**
* *Fine-Tune When:* Small-to-medium specialized datasets are available to adapt style, schema, domain nuance (legal, medical, finance), or specific skill sets.
* *Do NOT Fine-Tune When:*
* Large volumes of foundational data exist requiring pre-training from scratch.
* The required information changes continuously and can be supplied dynamically via RAG or context injection.
* Standard prompt engineering and few-shot metaprompts already fulfill business and latency requirements.





---

## 2. Platform Features & Capabilities

* **Customization Methods in Microsoft Foundry:**
1. **Supervised Fine-Tuning (SFT):**
* The standard approach: provides paired conversational demonstration examples consisting of prompt inputs and desired assistant completions.
* Focuses on teaching the model format adherence, classification, and tone.


2. **Direct Preference Optimization (DPO):**
* Preference fine-tuning where the model learns directly from comparative human or programmatic feedback without a separate reward model.
* Requires paired datasets containing identical prompt inputs accompanied by an explicitly designated **preferred output** and **non-preferred (rejected) output**.


3. **Reinforcement Fine-Tuning (RFT):**
* Specialized optimization technique available for reasoning models (e.g., `o4-mini`).
* Optimized for deterministic, rule-based domains where answers are objectively right or wrong and guessing is ineffective (e.g., complex mathematics, chemical synthesis, physics, legal rules).
* Uses a reward/grader function to score candidate completions dynamically during training.




* **Training Compute & Deployment Types:**
* **Standard (Regional):** Training compute executes strictly within the local provisioning region (e.g., East US 2). Enforces data residency compliance; priced at the highest operational rate (~$120/hr for typical large foundation architectures).
* **Global:** Training runs across globally distributed Azure compute clusters. Offers lower costs (~$100/hr) at the expense of regional data residency guarantees.
* **Developer:** Designed for non-urgent dev/test workloads; provides preemptible, lower-priority GPU compute at significant cost savings (~$50/hr), resulting in longer job queue and execution times.


* **Training Data Ingestion Options:**
* *Existing Dataset:* Select previously uploaded assets in the Foundry storage registry.
* *Upload New Dataset:* Direct file upload of JSON Lines (`.jsonl`) files from local storage.
* *Curated Sample Datasets:* Built-in reference sets (e.g., GSM8K mathematical reasoning benchmarks: `text-gsm8k-train-sample`).
* *Stored Completions:* Native ingestion of production conversation history and logged chat completions from Azure OpenAI / Foundry runtime endpoints.


* **Validation Data & Application Lifecycle Management (ALM):**
* **Validation Dataset:** Dedicated holdout `.jsonl` dataset used during training to monitor overfitting and generalization on unseen prompts.
* **Model Suffix:** Custom alphanumeric tag appended to the deployment name (e.g., `ft-leave-model:v1-dec2026`). Enables semantic tracking, side-by-side canary deployments, and rollbacks in enterprise ALM pipelines.


* **Training Observability & Convergence Metrics:**
* **Final Train Loss:** Cross-entropy loss metric ($0.0 \text{ to } 1.0$) measuring the model's prediction error and uncertainty when forecasting the next token. *Lower is better*.
* **Final Train Mean Token Accuracy:** Ratio of correctly predicted next tokens across training steps ($0.0 \text{ to } 1.0$). *Higher is better*.
* **Metrics Visualizer:** Interactive loss/accuracy curves with moving-average smoothing over training epochs.


* **Post-Training Lifecycle:**
* **Model Deployment:** Registering the fine-tuned adapter into serving infrastructure (Serverless pay-as-you-go or Provisioned Throughput Units).
* **Testing Playground:** Testing conversational completions in the Foundry Chat playground.



---

## 3. Formulas, Expressions & Schemas

### Supervised Fine-Tuning (SFT) Multi-Turn Chat Schema (`.jsonl`)

Data must be formatted as UTF-8 encoded JSON Lines where each discrete line is a standalone JSON object containing an array of message roles (`system`, `user`, `assistant`):

```json
{"messages": [{"role": "system", "content": "You are a specialized enterprise HR Leave Assistant. Answer employee inquiries accurately, showing step-by-step calculations, and terminate final balances with three exclamation marks."}, {"role": "user", "content": "An employee has 25 base vacation days, 5 carry-over days, and took 8 days off. What is the remaining balance?"}, {"role": "assistant", "content": "Let's calculate the leave balance step by step:\n1. Total Entitlement = Base Days (25) + Carry-over (5) = 30 days.\n2. Deduct Used Leave = 30 - 8 = 22 days.\n\nAnswer: 22 days remaining!!!"}]}
{"messages": [{"role": "system", "content": "You are a specialized enterprise HR Leave Assistant. Answer employee inquiries accurately, showing step-by-step calculations, and terminate final balances with three exclamation marks."}, {"role": "user", "content": "Jane has an allowance of 10 sick days and took 3 half-days. How much sick leave is left?"}, {"role": "assistant", "content": "1. Used sick leave = 3 * 0.5 = 1.5 days.\n2. Remaining sick leave = 10 - 1.5 = 8.5 days.\n\nAnswer: 8.5 days remaining!!!"}]}

```

### Direct Preference Optimization (DPO) Dataset Schema (`.jsonl`)

DPO requires an input prompt paired with comparative outputs (`chosen` vs. `rejected`):

```json
{
  "prompt": "Calculate the remaining sick leave for an employee with 10 days who used 2 days.",
  "chosen": "The employee has 8 days of sick leave remaining (10 total - 2 used).",
  "rejected": "I think you probably have some sick leave left. Maybe 8 days, but check with your manager."
}

```

### Token Training Loss & Evaluation Logic (Mathematical Foundations)

Cross-entropy training loss $L$ across target sequence tokens $T$ with ground-truth probabilities $y_t$:

$$L = -\frac{1}{T} \sum_{t=1}^{T} \log P(y_t \mid x, y_{<t})$$

* As the model learns to assign higher probability to expected tokens in the training set, $L \to 0$ and Mean Token Accuracy $\to 1.0$.

### Power Fx: Calling Deployed Fine-Tuned Model via Custom REST Connector in Copilot Studio

When invoking a fine-tuned Foundry model deployment endpoint from a Copilot Studio topic node:

```powerfx
// Step 1: Initialize payload for deployed fine-tuned model
Set(Topic.RequestPayload, 
    {
        model: "ft-gpt-4o-mini-leave-calculator-v1",
        messages: Table(
            { role: "system", content: "You are an enterprise HR Leave Assistant." },
            { role: "user", content: "Employee has 30 days total allowance and took 5 days. Calculate remaining." }
        ),
        temperature: 0.1,
        max_tokens: 300
    }
);

// Step 2: Dispatch REST request via Azure OpenAI connector
Set(Topic.ModelResponse,
    AzureOpenAIService.CreateChatCompletion(Topic.RequestPayload)
);

// Step 3: Extract completion text and clean custom suffix delimiters
Set(Topic.RawAnswer, First(Topic.ModelResponse.choices).message.content);
Set(Topic.CleanedAnswer, Substitute(Topic.RawAnswer, "!!!", ""));

```

---

## 4. Best Practices & Design Patterns

* **Data Preparation Quality Over Quantity:**
* High-quality, diverse, and representative demonstration pairs (typically 100 to 500 carefully vetted samples) yield better results than thousands of noisy, conflicting examples.
* Maintain consistent system metaprompts across all training examples.


* **Format Injection for Deterministic Parser Adherence:**
* When downstream systems (or Copilot Studio topics) rely on specific syntax (such as regex-parseable footers or specific delimiters like `Answer: X!!!`), encode this exact formatting into every `assistant` completion in the training dataset.


* **Application Lifecycle Management (ALM) Model Tagging:**
* Always utilize semantic suffixes during fine-tuning job creation (e.g., `-prod-v1`, `-sft-eval-02`).
* Decouple application endpoints from specific fine-tuned model versions by using generic Azure AI deployment names (e.g., `prod-leave-engine`). When a new fine-tuned model is trained, update the underlying deployment target rather than modifying endpoints across consumer apps.


* **Validation Data Isolation:**
* Ensure strict separation between the training set and validation set. Including training examples in validation sets creates false-positive accuracy metrics and conceals overfitting.



---

## 5. Technical Limitations, Gotchas & Guardrails

* **Training Queue & Compute Durations:**
* Fine-tuning jobs run asynchronously and take **several hours** to complete, even for small datasets.
* Selecting the **Developer** training tier reduces costs but can lead to variable scheduling delays due to resource preemption.


* **Compliance Boundaries with Global Training:**
* Selecting the **Global** training tier routes training data across Azure data centers outside the local environment region. In organizations governed by GDPR, HIPAA, or strict data residency mandates, Global training violates regional boundary policies; **Standard (Regional)** training must be enforced.


* **Architectural Overkill Anti-Pattern:**
* Do not fine-tune models to inject static factual data (e.g., uploading company policy handbooks for Q&A). Fine-tuning modifies *behavior and style*, not dynamic knowledge retrieval. Use **Retrieval-Augmented Generation (RAG)** via Azure AI Search or Copilot Studio Knowledge Sources for factual retrieval.


* **Deployment Propagation Lag:**
* Once a fine-tuned training run is marked `Completed`, deploying the model is not instantaneous. Deployment initialization takes **several minutes** before the endpoint can process requests in the playground or via REST calls.



---

## 6. AB-620 Exam Takeaways & Trap Alerts

* **Exam Trap — Fine-Tuning vs. Retrieval-Augmented Generation (RAG):**
* *Question:* "An organization needs their AI to answer questions using rapidly changing company policy manuals updated weekly. How should this be implemented?"
* *Trap Answer:* Fine-tune a base model using supervised fine-tuning.
* *Correct Answer:* Use **RAG / Knowledge Sources** (Azure AI Search or Copilot Studio Knowledge). Fine-tuning is for style, formatting, and behavioral alignment, not dynamic document retrieval.


* **Exam Trap — Customization Method Taxonomy:**
* **Supervised Fine-Tuning (SFT):** Learns from explicit prompt-and-response input pairs.
* **Direct Preference Optimization (DPO):** Learns from paired preferred and rejected outputs.
* **Reinforcement Fine-Tuning (RFT):** Available specifically for reasoning architectures (`o4-mini`) for rule-based, deterministic problem domains with verifiable right/wrong answers (math, physics, chemistry).


* **Exam Trap — Training Tier Boundaries:**
* *Standard:* Regional execution, guarantees data residency, higher cost.
* *Global:* Multi-region execution, breaks regional data residency constraints, lower cost.
* *Developer:* Non-urgent, preemptible compute, lowest priority, lowest cost.


* **Exam Trap — Metric Interpretations:**
* *Final Train Loss:* Measures error/uncertainty. **Lower is better** ($0.0 \text{ to } 1.0$).
* *Final Train Mean Token Accuracy:* Measures next-token prediction correctness. **Higher is better** ($0.0 \text{ to } 1.0$).


* **Dataset Format Requirement:**
* Fine-tuning datasets in Microsoft Foundry must be supplied in **JSON Lines (`.jsonl`)** format containing serialized chat message role/content structures. Plain `.csv`, `.docx`, or raw `.json` files are not accepted directly for training jobs.


