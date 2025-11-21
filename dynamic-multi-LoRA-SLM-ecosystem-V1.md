# Architecture: The Dynamic Multi-LoRA SLM Ecosystem

## Phase 1: The Data Engineering Engine
**Goal:** Convert raw domain noise into high-signal training fuel. SLMs require higher quality data than LLMs because they have less capacity to filter out noise.

### 1.1. Domain Knowledge Synthesis (For Understanding)
* **Input:** Unstructured PDFs, Wikis, Logs, Documentation.
* **Action:** Clean, chunk, and deduplicate.
* **Rationale:** The model cannot answer questions about "Company Policy X" if it has never seen the text of Policy X.
* **Example:** Converting a 500-page PDF into markdown chunks.

### 1.2. Synthetic Instruction Generation (For Reasoning)
* **Input:** The clean chunks from 1.1.
* **Action:** Use a "Teacher Model" (e.g., GPT-4o or Llama-3.1-405B) to generate QA pairs.
* **Rationale:** Raw text teaches *information*; Q&A pairs teach the model *how to retrieve* that information in a chat format.
* **Example:**
    * *Chunk:* "Error 503 means service unavailable."
    * *Generated Pair:* User: "What does 503 mean?" -> Assistant: "It indicates the service is unavailable."

### 1.3. Task-Specific Formatting (For Capabilities)
* **Action:** format data strictly for the intended LoRA adapter.
* **Rationale:** Specific formats trigger specific behaviors.
* **Examples:**
    * **NER:** `Input: "Bob called." -> Output: "['Bob': PERSON]"`
    * **Tool Calling:** `Input: "Get weather." -> Output: "<tool>get_weather</tool>"`
    * **Sentiment:** `Input: "I hate this." -> Output: "NEGATIVE"`

---

## Phase 2: The Training Pipeline (Offline)
**Goal:** Inject knowledge and skills without causing "Catastrophic Forgetting."

### 2.1. Continued Pre-Training (CPT)
* **Target:** The Base Model (e.g., Llama-3-8B-Instruct).
* **Method:** Full-parameter or LoRA training on **raw text** (from 1.1).
* **Rationale:** "Domain Adaptation." Before the model can summarize a legal brief, it must understand legal vocabulary.
* **Outcome:** A "Domain-Base" model that understands your jargon.

### 2.2. The Adapter Fork (Multi-LoRA Training)
Instead of training one giant model, we train parallel **Low-Rank Adapters (LoRAs)** on top of the Domain-Base.

| Adapter Name | Dataset Focus | Training Goal | Rationale |
| :--- | :--- | :--- | :--- |
| **`chat_adapter`** | General Q&A + Domain Q&A | Fluency & Knowledge | Handles general user queries and domain explanations. |
| **`tool_adapter`** | User Query $\to$ JSON | Precision & Syntax | Overfits on JSON structure to prevent syntax errors. |
| **`classifier_adapter`** | Text $\to$ Label | Constraint Adherence | Forces the model to output *only* specific labels (e.g., POSITIVE). |
| **`ner_adapter`** | Text $\to$ List | Entity Extraction | Specialized in identifying domain entities (Drugs, Part Numbers). |

### 2.3. Alignment (DPO - Direct Preference Optimization)
* **Target:** Specifically for `tool_adapter` and `chat_adapter`.
* **Method:** Train on `(Chosen, Rejected)` pairs.
* **Rationale:** SFT teaches the model *what* to say; DPO teaches it *what NOT* to say (e.g., hallucinations or invalid JSON).

---

## Phase 3: The Real-Time Inference Engine
**Goal:** Serve all tasks on a single GPU with millisecond latency.



### 3.1. The Serving Layer (vLLM or LoRAX)
* **Configuration:** Load the **Base Model** once (VRAM heavy). Reserve a "Swap Space" in VRAM for adapters.
* **Mechanism:** **Continuous Batching**.
    * The engine does not wait for a batch to fill.
    * New requests join the processing stream immediately.
* **Rationale:** Allows a single 24GB GPU to serve 4 distinct use cases simultaneously.

### 3.2. The Router (The "Brain")
* **Component:** A lightweight classifier (e.g., `distilbert` or a fast embedding similarity check) sitting before the SLM.
* **Action:** Analyzes the incoming prompt intent and assigns the correct `adapter_id`.
* **Flow:**
    1.  User: "Check the status of ticket #1234."
    2.  Router detects intent: `lookup_action`.
    3.  Router API Call: `POST /generate { "adapter_id": "tool_adapter", "prompt": ... }`

### 3.3. Dynamic LoRA Swapping (The "Magic")
* **Action:** The GPU applies the specific math weights for `tool_adapter` only for that specific request within the batch.
* **Latency:** ~0ms for cached adapters; <50ms for cold adapters.
* **Rationale:** Provides the accuracy of a specialized model with the cost of a shared model.

---

## Phase 4: End-to-End Workflow Example

Here is how a single user interaction flows through this architecture:

1.  **Input:** User types *"I'm frustrated with the login error, can you reset my password?"*
2.  **Router Analysis:**
    * The Router sees "frustrated" $\to$ tags for **Sentiment Analysis**.
    * The Router sees "reset password" $\to$ tags for **Tool Calling**.
3.  **Parallel Execution (Optional):**
    * **Request A (Sentiment):** Sent to SLM with `classifier_adapter`.
        * *Result:* "NEGATIVE" (Stored for analytics).
    * **Request B (Action):** Sent to SLM with `tool_adapter`.
        * *Result:* `{"tool": "reset_password_flow", "user_id": "current"}`.
4.  **Client Application:**
    * Receives the JSON from Request B.
    * Triggers the frontend "Reset Password" widget.
    * Logs the negative sentiment from Request A to the CRM.

---

## Summary of Technologies

| Component | Recommended Tech | Why? |
| :--- | :--- | :--- |
| **Data Gen** | `Distilabel` / GPT-4o | Best orchestration for synthetic data pipelines. |
| **Training** | `Unsloth` or `Axolotl` | Unsloth is 2x faster and uses 60% less memory. |
| **Serving** | `LoRAX` (Predibase) or `vLLM` | Built specifically for Multi-LoRA continuous batching. |
| **Routing** | `Semantic Router` (Aurelio) | Fast, local embedding-based routing (no API latency). |
