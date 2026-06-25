# NeuralStack | MS

> *Practitioner-level writing at the intersection of LLM engineering and AI security.*

**Manuela Schrittwieser** · Agentic AI & LLM Developer → AI Security Engineering Specialist · Linz, Austria

[![Blog](https://img.shields.io/badge/Blog-neuralstackms.tech-0a0a0a?style=flat-square&logo=hashnode&logoColor=white)](https://neuralstackms.tech)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-manuela--schrittwieser-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/manuela-schrittwieser)
[![GitHub](https://img.shields.io/badge/GitHub-MANU--de-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/MANU-de)

---

## About This Repository

This repository is the Hashnode-synced article store for the **NeuralStack | MS Tech Blog**. Every published article is mirrored here as a Markdown file with full frontmatter (title, slug, tags, publication date). It serves as the canonical version-controlled record of all published content under the NeuralStack brand.

The writing targets practitioners who are already building — not onboarding. Expect architecture breakdowns, threat models, and implementation specifics over introductory survey coverage.

---

## Latest · Featured Article

### [Sentinel-LLM: A Hybrid Guardrail Architecture for OWASP LLM06](https://neuralstackms.tech/sentinel-llm-a-hybrid-guardrail-architecture-for-owasp-llm06-sensitive-information-disclosure)
*Published June 25, 2026 · AI Security Engineering*

Technical deep dive into **Sentinel-LLM** — an open-source, three-layer detection pipeline for sensitive information disclosure in LLM outputs. Covers NER-based PII redaction (Microsoft Presidio), entropy-pattern credential scanning, and semantic IP leak detection via Sentence-Transformers cosine similarity in 384-dimensional vector space. Evaluated at 92% recall with ~120ms latency overhead.

→ [GitHub Repository](https://github.com/MANU-de/llm-leak-detector) · [Technical Paper (Ready Tensor)](https://app.readytensor.ai/publications/a-hybrid-approach-to-mitigating-sensitive-information-disclosure-owasp-llm06-in-llm-outputs-HTyMAudFXPZr) · [Demo (Loom)](https://www.loom.com/share/848661f6acec4384bc4ad4a8ac797860)

---

## Published Series

### Databases & Data Engineering in AI Security Engineering
*Four-part series examining the data layer as an attack surface in AI systems.*

| # | Title | Published |
|---|-------|-----------|
| 1 | Data Ingestion Pipeline Attacks | 2026 |
| 2 | Vector Database Security | 2026 |
| 3 | RAG Under Fire: Retrieval Pipeline Vulnerabilities & Indirect Prompt Injection | 2026 |
| 4 | Data Provenance & Governance as Security Controls | 2026 |

**Series coverage:** retrieval poisoning, adversarial embedding injection, indirect prompt injection via retrieved documents, chunk boundary exploitation, hybrid search attack vectors (BM25 manipulation, fusion layer exploitation), context window manipulation, and data lineage as a defensive control.

---

### Agentic AI & LLM Engineering
*Earlier work covering agentic system design, LLM tooling, and Python-based AI development.*

Selected article from the blog's foundational phase (October 2025 onward):

- [Beginner's Guide to Using Agentic AI in Python](https://neuralstackms.tech/beginners-guide-to-using-agentic-ai-in-python) — LangChain agent setup with tool integration

→ Full archive at [neuralstackms.tech](https://neuralstackms.tech)

---

## Active Security Projects

### Sentinel-LLM · LLM Leak Detector
Addresses **OWASP LLM06: Sensitive Information Disclosure** via a three-layer hybrid detection pipeline.

```
Layer 1 │ Microsoft Presidio + SpaCy en_core_web_lg  →  Contextual PII redaction
Layer 2 │ Custom regex + entropy analysis             →  Credential & secret scanning  
Layer 3 │ Sentence-Transformers all-MiniLM-L6-v2     →  Semantic IP leak detection (cosine similarity)
```

**Stack:** Python 3.11 · Presidio · SpaCy · Sentence-Transformers · Streamlit · Apache 2.0  
**Evaluation:** 92% recall on proprietary code leak detection · ~120ms avg pipeline latency

→ [github.com/MANU-de/llm-leak-detector](https://github.com/MANU-de/llm-leak-detector)

---

### MCP Security Scanner *(in development)*
Four-layer static and semantic analysis pipeline for Model Context Protocol server configurations.

```
Layer 1 │ Static config scan (structural misconfigs, permission over-grants)
Layer 2 │ Taint analysis via Semgrep (data flow across MCP tool definitions)
Layer 3 │ LLM semantic judgment (intent classification of tool descriptions)
Layer 4 │ Runtime monitoring (deferred behavioral anomaly detection)
```

Design philosophy: **precision over coverage** — false positives erode operational trust in security tooling.

---

## Topics Covered

`OWASP LLM Top 10` · `LLM Security` · `RAG Architecture` · `Prompt Injection` · `Vector Database Security` · `Sensitive Information Disclosure` · `Data Provenance` · `MCP Security` · `Agentic AI` · `AI Red Teaming` · `Sentence-Transformers` · `Microsoft Presidio` · `LangChain` · `Python`

---

## Repository Structure

Files at the repository root are Hashnode-synced article exports, named by CUID. Each file contains YAML frontmatter (title, slug, SEO metadata, cover image, tags, publication date) followed by the full article body in Markdown.

The `profile/` directory contains the GitHub organization profile README.

---

## Connect

- 🌐 [neuralstackms.tech](https://neuralstackms.tech)
- 💼 [linkedin.com/in/manuela-schrittwieser](https://www.linkedin.com/in/manuela-schrittwieser)
- 🐙 [github.com/MANU-de](https://github.com/MANU-de)

---

<sub>All content © NeuralStack | MS · Manuela Schrittwieser. Articles represent the author's independent research and practitioner perspective.</sub>
