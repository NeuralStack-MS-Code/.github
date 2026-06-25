---
title: "Sentinel-LLM: A Hybrid Guardrail Architecture for OWASP LLM06 – Sensitive Information Disclosure"
datePublished: 2026-06-25T09:06:02.639Z
cuid: cmqta22g900000akc6e48h5rl
slug: sentinel-llm-a-hybrid-guardrail-architecture-for-owasp-llm06-sensitive-information-disclosure
cover: https://cdn.hashnode.com/uploads/covers/68e922a757e675c5840506dd/4d8ebc92-2686-4e41-8edd-9b42f7267296.png
ogImage: https://cdn.hashnode.com/uploads/og-images/68e922a757e675c5840506dd/c16612f2-88a8-4ada-8f91-37672ec21542.png
tags: resources, tutorials, pytorch, vector-similarity, sentence-transformers, aisecurity, llm-guardrails, pii-redaction, cyber-ai

---

*Published on NeuralStack | MS · AI Security Engineering · June 2026*

* * *

Every production LLM deployment carries a category of risk that is simultaneously well-understood in theory and chronically under-engineered in practice: the model itself becomes the data exfiltration path. Whether through memorized training data, in-context leakage, or paraphrased reproduction of proprietary logic, Large Language Models can – and do – disclose information that should never leave the system boundary. OWASP codified this as **LLM06: Sensitive Information Disclosure**, and it remains one of the most practically exploitable entries in the LLM Top 10.

Most teams reach for the obvious solution: keyword filters and regex patterns. This is, to be direct, insufficient. It is also the starting assumption I set out to challenge with **Sentinel-LLM**, my latest open-source security tooling release.

* * *

## The Core Problem: Why Deterministic Scanners Fail

Regex-based Data Loss Prevention (DLP) works reasonably well in a world where sensitive data has a fixed, predictable syntax. API keys, Social Security Numbers, and IBAN strings are structurally distinct enough that pattern matching catches them at an acceptable rate.

LLM outputs do not live in that world.

A model that has seen a name, an email, and a role together in training data will reproduce them together in natural prose. A model asked to explain an authentication mechanism may paraphrase internal proprietary logic, stripping the variable names that a regex would have matched. A model may hallucinate a plausible-looking API key that happens to share the structural entropy of a real credential. In each case, the deterministic scanner produces either a false negative or – if tuned aggressively – an operationally useless volume of false positives.

The solution is not to choose between deterministic and probabilistic detection. The solution is to compose both, along with semantic analysis, into a layered pipeline where each layer handles what it is actually suited for.

* * *

## Architecture: Three Detection Layers, One Security Sidecar

Sentinel-LLM is designed as a **security sidecar** – a proxy that intercepts raw LLM output before it reaches the end user, passes it through three independent detection layers, and either redacts or blocks the response depending on what is found. The entire pipeline runs as local inference, meaning the contents of the "Security Vault" never leave the host environment. This is a non-negotiable design constraint for any tool handling internal IP.

```plaintext
LLM Raw Output
      │
      ▼
┌─────────────────────────┐
│   Layer 1: NER / PII    │  → Redaction
│   (Microsoft Presidio)  │
└─────────────────────────┘
      │
      ▼
┌─────────────────────────┐
│  Layer 2: Secret Scan   │  → Redaction
│  (Entropy Regex Patterns│
└─────────────────────────┘
      │
      ▼
┌─────────────────────────┐
│  Layer 3: Semantic IP   │  → Block
│  (Sentence-Transformers)│
└─────────────────────────┘
      │
      ▼
  Sanitized Response
```

### Layer 1 – Contextual PII Detection (Probabilistic)

The first layer uses **Microsoft Presidio** backed by SpaCy's `en_core_web_lg` Named Entity Recognition model. Unlike regex, NER operates on linguistic context. It understands that "John Doe" followed by an email address in a support-ticket-style sentence is a PII construct, not merely a string matching a name pattern. Each detected entity receives a confidence score; entities exceeding the operator-defined threshold are anonymized using type-specific placeholders rather than generic redaction markers.

This distinction matters in practice. Replacing `j.doe@example.com` with `[EMAIL_ADDRESS]` preserves the semantic structure of the response for legitimate auditing purposes, whereas a blanket redaction signal would simply break downstream consumers.

### Layer 2 – Secret and Credential Scanning (Deterministic)

The second layer applies custom `PatternRecognizers` targeting high-entropy strings: API keys, database connection strings, authentication tokens. The patterns are specifically hardened against the structural conventions of OpenAI and Anthropic key formats, as well as generic secret patterns that NLP models – trained on natural language – are architecturally unsuited to detect. Non-linguistic structure is where regex earns its place.

### Layer 3 – Semantic IP Guardrail (Vector Analysis)

This is the technically distinctive layer of the architecture.

To address the scenario where an LLM paraphrases proprietary internal source code – restructuring the logic, renaming variables, reordering operations – while preserving semantic equivalence, the system encodes both the "Protected Vault" (a corpus of sensitive internal code) and the LLM output as dense vector representations using **Sentence-Transformers** (`all-MiniLM-L6-v2`). It then computes **Cosine Similarity** between the two embeddings in a 384-dimensional space.

If the similarity score exceeds the configured threshold (default: 0.70), the response is blocked entirely rather than partially redacted. The semantic layer cannot recover a paraphrased function signature by removing a fragment – the leak is the semantic equivalence itself.

Initial evaluation results indicate a **92% Recall** rate for proprietary code leak detection, with an average pipeline latency overhead of approximately **120ms** – well within the tolerance range for real-time human-in-the-loop applications.

* * *

## Dynamic Thresholding: Operationalizing the False Positive Trade-off

One of the persistent engineering challenges in security tooling is threshold calibration. Set sensitivity too high, and legitimate outputs are blocked at a rate that makes the system unusable. Set it too low, and the guardrail fails its primary function.

Sentinel-LLM addresses this with a **Dynamic Threshold Controller** exposed through a Streamlit interface. Operators can tune the PII confidence threshold between 0.1 and 1.0 and adjust the semantic similarity threshold for the IP guardrail independently. The design is explicitly intended to give security teams a "Paranoia Slider" – the ability to dial between operational usability and enforcement strictness without touching configuration files.

This is also the foundation for future integration of per-classification-type risk profiles, where PII leakage and IP leakage would carry independent threshold policies reflecting their distinct regulatory and commercial risk profiles.

* * *

## Detection in Practice

To make the threat model concrete, three representative detection scenarios illustrate the pipeline in operation:

| Leak Category | Input Excerpt | Detection | Response |
| --- | --- | --- | --- |
| PII | "Contact John Doe at j.doe@email.com" | PERSON, EMAIL\_ADDRESS (Presidio) | Redact |
| Credential | "API\_KEY: sk-ant-api03-abcdefg12345" | Generic API Key (Pattern) | Redact |
| IP Leak | Paraphrased internal auth logic | 78% Semantic Similarity | **Block** |

The third scenario is the one that separates Sentinel-LLM from standard DLP tooling. There is no keyword in a paraphrased function that will match a pattern rule. Blocking it requires understanding that two texts are semantically equivalent – a property that only becomes computable through vector space representation.

* * *

## Technical Stack

*   **Core Runtime:** Python 3.11
    
*   **NER / PII Engine:** Microsoft Presidio, SpaCy `en_core_web_lg`
    
*   **Embedding Model:** Sentence-Transformers `all-MiniLM-L6-v2`
    
*   **Interface:** Streamlit
    
*   **License:** Apache 2.0
    

Local inference throughout. No scanning data is transmitted to external APIs.

* * *

## Roadmap: V2 and Beyond

The current release focuses exclusively on **outbound protection** – scanning LLM-generated output before delivery. The V2 roadmap addresses the full bidirectional attack surface:

*   **Inbound Prompt Injection Detection** via a DeBERTa-v3 classifier, creating a bidirectional security sandbox
    
*   **Vector DB Integration** with ChromaDB for enterprise-scale "Forbidden Vault" management, supporting large proprietary codebases
    
*   **Audit Logging** with structured JSON export for SIEM/ELK Stack integration
    
*   **FastAPI Wrapper** for production middleware deployment as an HTTP interceptor rather than a Streamlit prototype
    

The inbound capability is architecturally significant: once both layers are operational, Sentinel-LLM functions as a complete LLM traffic inspection proxy, applying security controls symmetrically to both the attack surface (prompts) and the exfiltration surface (completions).

* * *

## Availability

Sentinel-LLM is released under the Apache License 2.0 and is available now.

*   **GitHub Repository:** [github.com/MANU-de/llm-leak-detector](https://github.com/MANU-de/llm-leak-detector)
    
*   **Technical Publication:** [Ready Tensor — A Hybrid Approach to Mitigating OWASP LLM06](https://app.readytensor.ai/publications/a-hybrid-approach-to-mitigating-sensitive-information-disclosure-owasp-llm06-in-llm-outputs-HTyMAudFXPZr)
    
*   **Demo Video:** [Loom — Technical Walkthrough](https://www.loom.com/share/848661f6acec4384bc4ad4a8ac797860)
    

Contributions, issues, and discussion are welcome on GitHub. For questions, suggestions, and opportunities for collaboration, reach out via [LinkedIn](https://www.linkedin.com/in/manuela-schrittwieser/) or contact [neuralstackms.tech](https://neuralstackms.tech/).

* * *

*Manuela Schrittwieser is an Agentic AI & LLM Developer and AI Security Engineering Specialist , based in Linz, Austria. Her work covers LLM attack surfaces, output guardrails, and adversarial AI security tooling.*