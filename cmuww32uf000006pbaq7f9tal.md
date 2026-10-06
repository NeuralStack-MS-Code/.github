---
title: "Supply Chain Poisoning in AI/ML Pipelines"
datePublished: 2026-10-06T16:24:49.222Z
cuid: cmuww32uf000006pbaq7f9tal
slug: supply-chain-poisoning-in-ai-ml-pipelines
cover: https://cdn.hashnode.com/uploads/covers/68e922a757e675c5840506dd/781a6970-49ce-4181-a441-541b55932cb7.png
ogImage: https://cdn.hashnode.com/uploads/og-images/68e922a757e675c5840506dd/b04b37b8-e116-439c-8773-b2e833d51e9b.png
tags: supplychainsecurity, mlsecops, aisecurity, llmsecurity, neuralstackms

---

*NeuralStack | MS – AI Security Engineering*

* * *

## Summary

A modern ML system is assembled, not written. Training corpora are scraped or downloaded, weights are pulled from public hubs, and hundreds of transitive Python packages sit between a `pip install` and a production endpoint. Each of these inputs is a point where an adversary can act *before* the organization has written a single line of its own code.

This article maps the attack surface along the three upstream layers named in the title – **data, libraries, and pre-trained models** – and distinguishes two failure classes that are frequently conflated:

1.  **Execution compromise:** loading an artifact runs attacker-controlled code (pickle payloads, malicious packages, `trust_remote_code` bypasses, dataset loaders).
    
2.  **Behavioral compromise:** the artifact is inert as a file but the *learned function* is manipulated (backdoors, biased or trojaned weights, poisoned training samples).
    

The first class is largely addressable with classical supply chain controls, adapted to ML artifact formats. The second is not: it requires provenance, evaluation, and containment, because inspection of the artifact will not reveal it.

* * *

## 1\. Why the ML supply chain is structurally different

Traditional software supply chain security assumes that artifacts are *inspectable* (source, diffs, reproducible builds) and that *installing* is distinct from *running*. Neither assumption holds cleanly for ML.

*   **Weights are opaque.** A multi-gigabyte tensor file cannot be code-reviewed. Two checkpoints with identical architecture and near-identical benchmark scores can differ in a hidden trigger behavior.
    
*   **Loading is executing.** Several widely used serialization formats (Python `pickle`, and through it `torch.load`, `joblib`, legacy Keras Lambda layers) can run arbitrary code during deserialization.
    
*   **Repositories are code-bearing.** A model or dataset repository contains configuration, tokenizers, custom modeling code, and loader scripts – all of which may be fetched and, depending on flags, executed.
    
*   **The data is part of the program.** In a learned system, training data is the closest analogue to source code. Whoever controls a slice of it controls a slice of the behavior.
    
*   **The pipeline is automated and credentialed.** Data-processing workers, training jobs, and evaluation harnesses typically hold cloud credentials and registry tokens, so a single compromised step is a pivot point.
    

OWASP reflects this split: the 2025 LLM Top 10 lists **LLM03 Supply Chain** (compromised models, packages, and sources) separately from **LLM04 Data and Model Poisoning** (manipulation of pre-training, fine-tuning, or embedding data), and treats the two as overlapping but distinct.

## 2\. Threat model at a glance

| Layer | Attacker capability | Class | Representative evidence |
| --- | --- | --- | --- |
| Training / fine-tuning data | Inject or alter a small number of samples; control expired domains or crowd-edited sources | Behavioral | Carlini et al., split-view and frontrunning poisoning; Anthropic / UK AISI / Turing study |
| Dataset loaders and configs | Ship a dataset whose loader or config executes code | Execution | Hugging Face incident, July 2026 |
| Libraries and build tooling | Publish or inject malicious releases; register hallucinated package names; steal publishing tokens | Execution | LiteLLM / telnyx (March 2026); slopsquatting cases |
| Model files | Embed code in pickle-based formats; evade scanners | Execution | JFrog findings; PickleScan CVEs; ShadowPickle preprint |
| Model repositories and loaders | Abuse custom-code mechanisms and config parsing | Execution | `trust_remote_code` bypass CVEs in Transformers and Diffusers |
| Model weights | Publish a model with an implanted backdoor | Behavioral | Poisoning research; not detectable by file scanning |

## 3\. Data: poisoning with a small, fixed budget

### 3.1 The "percentage of the corpus" intuition is wrong

A long-standing assumption held that poisoning a large model requires controlling a meaningful *fraction* of its training data. Research by Anthropic, the UK AI Security Institute, and the Alan Turing Institute challenges this. Across models from 600M to 13B parameters, trained on corpora differing by more than 20x, roughly **250 poisoned documents** were sufficient to install a backdoor reliably, while 100 were not. The required count was near-constant rather than proportional to dataset size.

Two caveats matter for an accurate reading. The studied backdoor was deliberately narrow (a trigger that induces gibberish output, i.e. a denial-of-service behavior), and the experiments covered models up to 13B parameters. The result does not demonstrate that frontier-scale models are trivially steerable toward arbitrary harmful behavior. It does establish that the *cost model* attackers face is more favorable than previously assumed, and that defenders should reason in terms of the **absolute number** of adversarial samples, not their share of the corpus. Related work reports the same pattern for fine-tuning datasets, including those used to train safety classifiers.

### 3.2 Acquisition-time attacks on web-scale datasets

Carlini et al. showed that large distributed datasets, which are essentially lists of URLs, inherit the weaknesses of the web:

*   **Split-view poisoning.** The dataset index records what a URL returned when it was curated; clients later download whatever it returns *now*. Because domains are leased, an attacker can purchase expired domains referenced by a dataset and serve different content. The paper estimates that for roughly $60 an attacker could have controlled about 0.01% of LAION-400M.
    
*   **Frontrunning poisoning.** Datasets built from periodic snapshots of crowd-edited sources can be poisoned by timing malicious edits just before a scheduled snapshot, before moderators revert them.
    

The root cause is the absence of integrity binding between the dataset *manifest* and the dataset *content*. Hashes in the manifest remove the split-view vector for any content that is re-downloaded.

### 3.3 Dataset loaders as an execution path

Data is not always inert. In July 2026 Hugging Face disclosed an intrusion that began with a **malicious dataset** abusing two code-execution paths in its dataset-processing pipeline: a remote-code dataset loader and a template-injection flaw in dataset configuration. Per the company's disclosure as summarized by the Cloud Security Alliance, the intruder then escalated from a processing worker to node level, harvested cloud and cluster credentials, and moved laterally. Hugging Face reported no evidence that public models, datasets, Spaces, or its published packages and container images were modified. Reporting differs on details of the attacking agent, and the underlying model has not been confirmed by the company; this article draws only on the points on which the primary-derived sources agree.

The architectural lesson is independent of attribution: **a pipeline that executes untrusted dataset code on a worker holding broad credentials converts a data-integrity problem into an infrastructure compromise.**

### 3.4 Retrieval and embedding data

OWASP's LLM04 explicitly includes embedding data. For RAG systems, the knowledge base is an ingestion path that bypasses training altogether: a poisoned document in the index can steer outputs at inference time with no model change. Controls are the same in spirit – provenance on ingested documents, write-access control on the index, and treating retrieved text as untrusted input.

## 4\. Libraries: the classical supply chain, with AI-specific accelerants

### 4.1 Compromise of widely used packages

In March 2026, releases of `litellm` (a popular LLM proxy layer) and `telnyx` were published to PyPI containing credential-harvesting malware that executed at install time. According to the PyPI incident report, the trigger was an API token exposed through an exploited dependency of a security tool (Trivy). The report's measurements are instructive:

*   Time from upload to quarantine for `litellm`: about 2 hours 32 minutes; for `telnyx`, about 3 hours 42 minutes.
    
*   The affected `litellm` versions were downloaded more than 119,000 times in that window.
    
*   PyPI estimated that roughly 40–50% of `litellm` installs were *unpinned*, fetching the latest release on each run.
    

Two properties make this class dangerous for ML teams. First, the malware is injected into packages that are *already trusted and widely deployed*, not into look-alikes. Second, stolen tokens fund the next compromise, so the campaign is self-propagating.

### 4.2 Slopsquatting

Coding assistants occasionally recommend package names that do not exist. An attacker who observes recurring hallucinations can register those names with a post-install payload. This is no longer purely theoretical: the Cloud Security Alliance documents the malicious npm package `unused-imports` (the real package being `eslint-plugin-unused-imports`) and the case of `react-codeshift`, a hallucinated name that appeared in a commit of 47 LLM-generated agent skills before being registered. Measured hallucination rates vary widely by model; some code models exceeded 33% in certain configurations in the cited research. Agentic workflows amplify the risk because no human may read the dependency list before `install` runs.

### 4.3 Loader flags are part of the trust boundary

`trust_remote_code=False` is widely treated as a safety switch. Recent disclosures show it is not an absolute guarantee:

*   **CVE-2026-4372** allowed remote code execution through crafted model configurations in Hugging Face Transformers when the optional `kernels` package was installed, bypassing `trust_remote_code=False`. Affected versions had been downloaded roughly 232 million times before a patch.
    
*   **CVE-2026-44827** and **CVE-2026-45804** (Zafran Labs, Diffusers) describe a bypass of the remote-code trust check and a time-of-check/time-of-use variant that altered executed code after consent had been given. Fixed in Diffusers 0.38.0.
    

The pattern: any code path that interprets repository-controlled configuration is attack surface, and *version currency of ML libraries is a security control*.

## 5\. Pre-trained models: files and functions

### 5.1 Serialization formats that execute code

Pickle-based formats run code on load. JFrog identified at least 100 malicious model instances on Hugging Face in 2024, about 95% of them PyTorch-based, with the remainder using TensorFlow/Keras Lambda layers. Disclosures in 2026 continue to include unsafe deserialization in ML frameworks (for example, CWE-502 findings in several projects that call `torch.load` without `weights_only=True`).

### 5.2 Scanners are a filter, not a guarantee

Pickle scanners must interpret a file exactly as the loader does; any parsing divergence is a bypass. This has been demonstrated repeatedly:

*   JFrog reported critical PickleScan flaws (CVE-2025-10155, a file-extension bypass; CVE-2025-10156, a ZIP CRC bypass) in which PyTorch loaded and executed files the scanner failed to flag.
    
*   The PickleBall paper (ACM CCS 2025) built backdoored models that bypassed two state-of-the-art scanners and found that PyTorch's weights-only unpickler prevented about 15% of Hugging Face pickle repositories from loading, which explains why teams disable it.
    
*   The July 2026 *ShadowPickle* preprint (arXiv:2607.17503) presents further stealthy pickle attacks and a benchmark for evaluating scanners.
    

A clean scan result means only that no known signature matched in that scanner version.

### 5.3 The safer format – and what it does not cover

**safetensors** stores raw tensors and metadata and has no code-execution path on load, which removes the execution class for weights. It does **not** address behavioral compromise: a safetensors file can contain a backdoored model. It also does not cover the surrounding repository content (custom code, configs, tokenizers).

### 5.4 Behavioral backdoors

A trojaned checkpoint behaves normally on standard evaluations and deviates on a trigger. No file-level scanner can detect this, because nothing about the file is malformed. Realistic defenses are: provenance (know who trained it and from what), behavioral evaluation including trigger-search and red teaming, preferring models whose training lineage is documented, and limiting the *authority* granted to any model output (see §6.6).

## 6\. A defense architecture

The controls below are organized as a pipeline. They are deliberately layered because no single control covers both failure classes.

### 6.1 Treat every external artifact as untrusted input at an ingestion boundary

Route models, datasets, and packages through a single **quarantine stage**: pinned acquisition, format policy, scanning, hashing, and registration – then promote to an internal registry. Production and training jobs read only from the internal registry.

### 6.2 Pin by content, not by name

```bash
# Python: hash-locked, reproducible installs
uv lock                                  # or: pip-compile --generate-hashes
pip install --require-hashes -r requirements.txt
```

```toml
# pyproject.toml – dependency cooldown (relative window, per the PyPI guidance)
[tool.uv]
exclude-newer = "P3D"
```

A cooldown gives the community time to detect and quarantine malicious releases (the LiteLLM exposure window was hours). Pair it with vulnerability scanning so security fixes are not delayed; Dependabot and Renovate bypass cooldowns for security updates by default.

For models, pin to an immutable **commit SHA**, not a branch or tag, and restrict file types:

```python
from huggingface_hub import snapshot_download

path = snapshot_download(
    repo_id="org/model",
    revision="<full 40-character commit SHA>",
    allow_patterns=["*.safetensors", "config.json", "tokenizer*"],
)
```

For datasets, record a SHA-256 manifest at curation time and verify at every download; this closes split-view poisoning for content you re-fetch.

### 6.3 Enforce a format and loader policy

```python
# Weights: safetensors only
from safetensors.torch import load_file
state = load_file("model.safetensors", device="cpu")

# If a pickle-based checkpoint is unavoidable, constrain the unpickler explicitly
import torch
state = torch.load("ckpt.pt", weights_only=True, map_location="cpu")

# Transformers: refuse repository-supplied code and non-safetensors weights
from transformers import AutoModel
model = AutoModel.from_pretrained(path, trust_remote_code=False, use_safetensors=True)
```

Recent PyTorch releases default to `weights_only=True`; set it explicitly anyway so that behavior does not depend on the installed version. Convert legacy checkpoints **inside the isolated environment in §6.4**, never on a workstation or a credentialed worker.

### 6.4 Isolate anything that must parse untrusted content

Ingestion, conversion, dataset preprocessing, and scanning run in ephemeral sandboxes with: no cloud credentials or registry tokens, no outbound network except an allowlisted mirror, a read-only root filesystem, and short-lived per-task identity. The Hugging Face incident is the case study for why: initial code execution on a worker was inexpensive for the attacker; the *credentials reachable from that worker* determined the blast radius.

### 6.5 Provenance, signing, and bills of materials

*   **Model signing.** The OpenSSF Model Signing (OMS) specification defines a detached, Sigstore-bundle-based signature that covers a model directory (weights, configuration, tokenizer, even datasets) as one verifiable unit, and is agnostic about key infrastructure (bare keys, certificate chains, or keyless Sigstore identity). The reference tooling is `pip install model-signing`, with sign and verify commands. Verification is only meaningful if you pin the *expected signer identity*; consult the project documentation for the verification options.
    
*   **AI-BOM.** CycloneDX 1.6 includes an ML-BOM representation for models, datasets, and their lineage. A 2.0 AI/ML schema is under proposal in the specification repository at the time of writing and should be treated as a draft.
    
*   **Package provenance.** Prefer packages published through Trusted Publishers, which use short-lived credentials and produce attestations that downstream consumers can check.
    

Provenance does not prove a model is benign; it makes tampering in transit detectable and gives you a defensible record of what you deployed.

### 6.6 Evaluate for behavior, and constrain authority

*   Run a **pre-promotion evaluation gate**: capability and safety regressions against the previous approved model, plus targeted probing for trigger-conditioned behavior (unusual token patterns, out-of-distribution prompts, differential testing against a trusted reference model).
    
*   Keep training-time telemetry (loss curves, gradient or influence anomalies) and data lineage to support post-hoc investigation. Note the practical difficulty reported in the literature: once a poisoned model is trained, the responsible samples generally cannot be identified or removed without retraining.
    
*   Assume a residual risk of behavioral compromise and **limit what a model's output can do**: least-privilege tool access, human approval for high-impact actions, output validation, and monitoring.
    

### 6.7 Secure the pipeline itself

For code you publish or build: avoid insecure CI triggers (notably `pull_request_target`), pass workflow inputs as environment variables to prevent template injection, pin actions by commit SHA, use Trusted Publishers instead of long-lived tokens, and enable phishing-resistant 2FA on all maintainer accounts.

## 7\. Minimum viable checklist

| # | Control | Addresses |
| --- | --- | --- |
| 1 | Hash-locked dependencies plus a cooldown window | Compromised releases, slopsquatting |
| 2 | Commit-SHA pinning for models and datasets; hash manifests | Tampering in transit, split-view poisoning |
| 3 | safetensors-only policy; `trust_remote_code=False`; explicit `weights_only=True` | Execution via model files and repos |
| 4 | Patch currency for Transformers, Diffusers, and PyTorch | Loader-flag bypass CVEs |
| 5 | Credential-free, network-restricted ingestion sandbox | Blast radius of any loader exploit |
| 6 | Signature verification against a pinned signer; ML-BOM per release | Provenance and auditability |
| 7 | Behavioral evaluation gate and least-privilege deployment | Residual backdoor risk |
| 8 | Review of AI-suggested dependencies before install | Slopsquatting |

## 8\. Open problems

*   **Scanning will remain adversarial.** Deserialization scanners chase loader behavior; the durable fix is to remove the dangerous format, not to improve the filter.
    
*   **Backdoor detection lacks a general solution.** Existing evaluations are probabilistic, and the fixed-count poisoning result implies that small, well-placed contamination may evade statistical monitoring.
    
*   **Provenance has limited coverage.** Signing and ML-BOMs are maturing, but adoption by hubs and publishers is uneven, and a signature attests *origin*, not *safety*.
    
*   **Agentic ingestion compresses response time.** When agents install dependencies and fetch artifacts autonomously, the interval between publication of a malicious artifact and its execution can shrink below human review cycles. Policy enforcement must therefore be automated and sit in front of the agent, not after it.
    

## Conclusion

Supply chain poisoning in ML is two problems wearing one name. The execution class – malicious pickles, packages, loaders – yields to disciplined engineering: remove unsafe formats, pin by content, isolate untrusted parsing, and keep libraries patched. The behavioral class – poisoned data and trojaned weights – cannot be filtered at the file level and demands provenance, evaluation, and architectural containment. Organizations that treat their model registry and data store with the same rigor as their production code repository will be materially better positioned against both.

* * *

## Sources

*   Anthropic, "A small number of samples can poison LLMs of any size" – https://www.anthropic.com/research/small-samples-poison
    
*   Carlini et al., "Poisoning Web-Scale Training Datasets is Practical" (IEEE S&P 2024) – https://arxiv.org/abs/2302.10149
    
*   Anthropic Alignment, "Poisoning Fine-tuning Datasets of Constitutional Classifiers" – https://alignment.anthropic.com/2026/backdooring-classifiers/
    
*   Cloud Security Alliance, "Hugging Face's Autonomous AI Agent Breach" (July 20, 2026) – https://labs.cloudsecurityalliance.org/research/csa-research-note-huggingface-autonomous-agent-breach-202607/
    
*   The Hacker News, "World's Largest AI Model Repository Hugging Face Breached by Autonomous AI Agent" – https://thehackernews.com/2026/07/worlds-largest-ai-model-repository.html
    
*   PyPI Blog, "Incident Report: LiteLLM/Telnyx supply-chain attacks, with guidance" (April 2, 2026) – https://blog.pypi.org/posts/2026-04-02-incident-report-litellm-telnyx-supply-chain-attack/
    
*   Cloud Security Alliance, "Slopsquatting: AI Code Hallucinations Fuel Supply Chain Attacks" – https://labs.cloudsecurityalliance.org/research/csa-research-note-slopsquatting-ai-supply-chain-20260419-csa/
    
*   Aikido, "Slopsquatting: The AI Package Hallucination Attack Already Happening" – https://www.aikido.dev/blog/slopsquatting-ai-package-hallucination-attacks
    
*   TechRepublic, "Malicious Hugging Face Models Could Trigger Remote Code Execution" (CVE-2026-4372) – https://www.techrepublic.com/article/news-hugging-face-transformers-rce-flaw/
    
*   Zafran Labs, "FaceHugger: Vulnerabilities in Hugging Face Diffusers" – https://www.zafran.io/resources/facehugger-vulnerabilities-in-hugging-face-diffusers-open-door-to-supply-chain-attacks-on-enterprise-ai
    
*   JFrog, "Unveiling 3 Zero-Day PickleScan Vulnerabilities" – https://jfrog.com/blog/unveiling-3-zero-day-vulnerabilities-in-picklescan/
    
*   NSFOCUS, "AI Supply Chain Security: Hugging Face Malicious ML Models" (summarizing JFrog findings) – https://nsfocusglobal.com/ai-supply-chain-security-hugging-face-malicious-ml-models/
    
*   PickleBall, "Secure Deserialization of Pickle-based Machine Learning Models" (CCS 2025) – https://cs.brown.edu/people/vpk/papers/pickleball.ccs25.pdf
    
*   Stingrai, "A Clean Model Scan Is Not a Safe Model" (referencing ShadowPickle, arXiv:2607.17503) – https://www.stingrai.io/blog/clean-model-scan-not-safe-picklescan-safetensors
    
*   OpenSSF, "An Introduction to the OpenSSF Model Signing (OMS) Specification" – https://openssf.org/blog/2025/06/25/an-introduction-to-the-openssf-model-signing-oms-specification/
    
*   OpenSSF Model Signing Specification – https://github.com/ossf/model-signing-spec
    
*   CycloneDX proposed 2.0 AI/ML schema (draft) – https://github.com/CycloneDX/specification/pull/948
    
*   OWASP Top 10 for LLM Applications 2025 – https://genai.owasp.org/