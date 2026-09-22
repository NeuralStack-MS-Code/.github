---
title: "Adversarial Attacks on Multi-Modal LLMs: How Image, Audio, and Video Inputs Get Weaponized and How to Defend Against It"
datePublished: 2026-09-22T11:30:45.581Z
cuid: cmuclezls00010agmc10o8u5p
slug: adversarial-attacks-on-multi-modal-llms-how-image-audio-and-video-inputs-get-weaponized-and-how-to-defend-against-it
cover: https://cdn.hashnode.com/uploads/covers/68e922a757e675c5840506dd/d14ed204-cd9e-41df-aae4-e2286e8b8cdd.png
ogImage: https://cdn.hashnode.com/uploads/og-images/68e922a757e675c5840506dd/fd5d7c14-2150-4f20-bf39-b7fcce0510be.png
tags: adversarial-machine-learning, aisecurity, multimodal-llm-security, image-scaling-attacks

---

*NeuralStack | MS — AI Security*

For most of the past three years, the AI security community's attention has been fixed on text: prompt injection, jailbreak templates, token-level adversarial suffixes. That focus made sense while text was the only input surface. It no longer is. Production systems now routinely accept images, audio, and video as first-class inputs — customer support bots that read screenshots, meeting assistants that ingest recordings, agentic pipelines that browse the web and screenshot what they find. Every one of those modalities is a new channel through which an attacker can reach the model, and each channel comes with its own preprocessing pipeline, its own encoder, and — critically — its own gaps in safety alignment.

This article maps the current attack surface across image, audio, and video inputs to vision-language and audio-language models, and lays out the defensive patterns that are actually holding up in practice, at the input layer, the model layer, and the system architecture layer.

## Why Multimodality Expands the Attack Surface

Three structural properties of multimodal LLMs work against their defenders.

**The modality gap.** Text-based safety alignment (RLHF, constitutional training, refusal fine-tuning) is trained overwhelmingly on text. When an image or audio encoder feeds embeddings into the same instruction-following pathway, the model has not been trained with the same intensity to recognize *malicious intent expressed non-textually*. Comprehensive surveys of adversarial robustness in multimodal LLMs note that introducing a second modality tends to make models more susceptible to harmful outputs than their text-only counterparts, both because each modality carries its own independent vulnerabilities and because alignment across modalities is imperfectly synchronized.

**Preprocessing as an attack surface.** Vision-language models rarely process raw user uploads; they resize, crop, and normalize first. Audio pipelines resample and denoise. These transformations are usually treated as plumbing, invisible to security review — but plumbing that transforms attacker-controlled input before it reaches the model is exactly where adversarial payloads hide.

**No architectural boundary between "content" and "instruction."** A vision-language model does not natively distinguish between an image a user wants described and text embedded in that image that the model is meant to execute as an instruction. Once the vision encoder has processed the pixels, both flow into the same context as the system and user prompts. OWASP's LLM Top 10 revision has formally extended its top-ranked prompt injection category (LLM01) to cover exactly this class of multimodal injection vector.

## Attack Taxonomy

### Image-Based Attacks

**Adversarial perturbations.** The classical adversarial-example toolkit — FGSM, PGD, and their descendants — transfers to vision-language models with modest adaptation. Rather than flipping a classifier's label, the objective shifts to manipulating the last hidden layer the LLM sees, so that an imperceptibly perturbed image steers the model's generated text. Frameworks such as AnyAttack have shown that these perturbations, once crafted against one open model, transfer with meaningful success to commercial systems including GPT-4V, Claude, and Gemini — a finding that should concern anyone assuming "closed model" implies "untested against transfer attacks."

**Image scaling / downscaling attacks.** This is arguably the most consequential image-based technique to emerge in the last eighteen months. Most production pipelines downscale user-uploaded images before they reach the model, for cost and context-window reasons. Trail of Bits demonstrated that an attacker can craft a high-resolution image that looks innocuous at full size but reveals a hidden instruction once the pipeline's resizing algorithm (nearest-neighbor, bilinear, or bicubic interpolation) processes it — the interpolation artifacts assemble into legible text that was invisible before. Their proof-of-concept achieved data exfiltration against Gemini CLI and several other production systems; their open-source tool, Anamorpher, operationalizes the technique. A systematic follow-up evaluation confirmed the severity at scale: across a benchmark of 10,000 crafted images, payload rendering after downscaling succeeded in over 90% of cases against LLaVA-1.5, and roughly 70% of attacks against that model achieved a complete safety bypass. A more recent adaptive variant, Chameleon, uses an agentic optimization loop to keep the payload robust across varying scale factors and downsampling libraries, reaching 84–91% attack success even where static baselines average around 32%.

**Typographic attacks.** CLIP-based vision encoders — which underpin most open vision-language models — contain "multimodal neurons" that respond to a concept regardless of whether it appears as an image or as text. This means an attacker can defeat a classifier or bias a downstream generation simply by writing a misleading word on a sticky note and placing it in the frame. The SCAM benchmark, the largest real-world typographic attack dataset to date, confirms the vulnerability is not an artifact of synthetic test cases. Mitigations in this space are still largely research-stage: Defense-Prefix learns a token prepended to class names to suppress the effect without retraining, while the more recent Dyslexify approach identifies and ablates the specific attention heads responsible for propagating typographic information to the model's output token, improving robustness on typographic ImageNet variants by up to roughly 20 percentage points.

**Steganographic and OCR-path visual prompt injection.** Rather than an adversarial perturbation invisible to a classifier, this class embeds a plainly readable but visually unobtrusive instruction — small font, background-matched color, edge-of-frame placement — that a human reviewer is unlikely to notice but that the model's OCR-capable vision encoder will read and execute. Published evaluations of this "IPI" (image-based prompt injection) pipeline against GPT-4-turbo achieved up to 64% attack success under stealth constraints, using segmentation-based region selection and adaptive font scaling to conceal the payload from human inspection while preserving model interpretability.

### Audio-Based Attacks

**Waveform-level adversarial perturbations.** The foundational work here predates today's audio-LLMs by several years — Carlini and Wagner demonstrated targeted adversarial examples against speech-to-text systems in 2018, and Schönherr et al. showed the same year that perturbations could be hidden below the threshold of human psychoacoustic perception. That lineage now targets large audio-language models directly: AdvWave achieved a working jailbreak against GPT-4o's speech-to-speech API in roughly 30 queries, and subsequent work (universal acoustic adversarial attacks) has produced fixed or conditional trigger segments that reliably alter model behavior across inputs, building on universal triggers first demonstrated against Whisper.

**Narrative and compositional framing.** Not every effective audio attack requires imperceptible noise. Embedding a harmful request inside role-play, a fictional dialogue, or a multi-speaker conversation increases jailbreak success simply by changing how the model contextualizes the request — a technique that generalizes the narrative-jailbreak patterns well known from text but appears to be even more effective when delivered as spoken audio. A related "flanking attack" inserts the malicious query among a sequence of otherwise benign spoken questions, exploiting attention allocation across the sequence rather than any single adversarial signal.

**Content dilution.** Rather than restructuring the narrative, this family conceals the adversarial payload inside neutral surrounding material to lower its salience and reduce the odds a safety classifier flags it — the audio equivalent of burying a malicious clause inside a long, boring contract.

### Video-Based Attacks

Video inherits every image- and audio-based technique and adds a temporal dimension that is, if anything, less well defended. Two patterns stand out.

**Temporal frame injection.** Because video-understanding pipelines typically sample or clip frames rather than processing every one, an attacker can place benign content at the start of a clip — passing any initial content-screening step — while embedding the actual malicious instruction only in later frames that are processed once the pipeline has already accepted the video as legitimate input. A video-summarization assistant that ingests meeting recordings is a realistic target: five seconds of ordinary footage followed by a single frame carrying a visually embedded instruction to exfiltrate data is enough, if the sampling and safety-screening logic does not treat every sampled frame with equal suspicion.

**Flow-based key-frame perturbation.** FMM-Attack was the first systematic demonstration that adversarial perturbations can be optimized against video-LLMs directly, using a temporal mask to concentrate the perturbation budget on the frames most influential to the model's output rather than spreading it uniformly across the clip. Follow-up work has shown that adversarial video samples generated against *image*\-only multimodal LLMs transfer with meaningful success to dedicated video-LLMs, meaning an attacker does not need access to the target video model at all to craft an effective attack against it.

### Cross-Modal and Compositional Attacks

The most dangerous attacks in this space are rarely single-modality. **Bi-modal adversarial prompts** distort the visual and textual channels simultaneously, forcing the model to reconcile inconsistent signals in a way that degrades its ability to apply either modality's safety filtering. **Chain-of-attack** research presented at CVPR 2025 demonstrated that success rates compound when multiple techniques — steganographic embedding plus semantic manipulation, for instance — are chained rather than deployed independently. And architecture matters: models using early-fusion designs that blend visual and text tokens from the very first layer (as opposed to processing each modality separately before a later fusion step) have been observed by red teams to treat purely visual symbols — emoji sequences, rebus-style puzzles — as functional instructions without any accompanying text at all, collapsing the already-thin line between "content" and "instruction" even further.

| Modality | Representative technique | Reported effectiveness | Primary countermeasure |
| --- | --- | --- | --- |
| Image | Downscaling / image-scaling injection | ~70–92% payload rendering, up to 70% full bypass on vulnerable models | Preview-before-scale, dimension limits, confirmation gating |
| Image | Typographic attack on CLIP encoder | Significant, consistent robustness drop across CLIP variants | Mechanistic ablation (Dyslexify), defense-prefix tokens |
| Image | Steganographic / OCR-path injection | Up to 64% under stealth constraints (GPT-4-turbo) | OCR-aware input screening, trust-boundary segregation |
| Audio | Waveform adversarial perturbation | Working jailbreak in ~30 queries against closed API | Adversarial training (partial), activation patching |
| Audio | Narrative / flanking framing | High and improving with elaboration | Guardrail stack independent of narrative framing |
| Video | Temporal frame injection | Demonstrated feasible; sampling-dependent | Uniform per-frame screening, not just clip-level |
| Cross-modal | Bi-modal / chained attacks | Compounding success vs. single-technique baselines | Defense-in-depth, no single-layer reliance |

## Defensive Strategies

No single control closes this attack surface. The pattern that survives contact with real red-team results is defense-in-depth, applied at four distinct layers.

### 1\. Input-Level Hardening

Treat every non-text input as untrusted the same way you would treat an uploaded file on a web server. Concretely:

*   Enforce dimension and resolution limits before any scaling operation, and where downscaling is unavoidable, render and log the *post-scaling* version the model will actually see — Trail of Bits's own recommendation after discovering the scaling attack class, and one of the few mitigations with direct empirical backing.
    
*   Run OCR or equivalent text-detection on every image before it reaches the model context, and require explicit user confirmation before any sensitive tool call that was triggered by text discovered inside an image rather than typed directly by the user.
    
*   Apply the same per-frame scrutiny to video that you apply to a single image; clip-level screening that only inspects the first few seconds is exactly the gap that temporal frame injection exploits.
    

### 2\. Model-Level Robustness

Adversarial training measurably helps but does not close the gap. Interspeech 2025 research on defending speech-enabled LLMs found that larger, more heavily adversarially trained models held up better, but every configuration tested remained breakable given sufficient attacker effort. Treat adversarial training as raising the cost of attack, not as elimination. Where feasible, mechanistic defenses that target the specific circuit responsible for a vulnerability — as Dyslexify does for typographic attacks in CLIP — offer a more surgical and more interpretable alternative to blanket retraining, and can be layered on top of an existing model without a full fine-tuning cycle.

### 3\. Architectural and Systemic Defenses

This is where the discipline connects most directly to broader LLM security architecture, and where I'd point back to patterns we've already validated on the text side of NeuralStack | MS's own tooling.

**Multi-scale and cross-modal consistency checks.** If an image's interpretation changes materially between the resolution a human reviewer sees and the resolution the model actually processes, that discrepancy is itself a detection signal — this is precisely the defense the Chameleon paper's authors propose after demonstrating how reliably their attack survives standard downscaling.

**The Dual-LLM "Watchdog" pattern, extended to multimodal input.** LLM-Guard's v2.0 architecture uses a lightweight guard model (Llama Guard 3 1B) to inspect the intent and output of a primary reasoning model before either reaches the user or triggers a tool call. The same pattern generalizes cleanly to multimodal input: route image, audio, and video content through a dedicated, narrowly-scoped inspection pass — OCR extraction, audio transcription, frame sampling — *before* it enters the primary model's context, rather than trusting the primary model to police content it is simultaneously trying to interpret.

**Zero-knowledge tool-gating for sensitive actions.** LLM-Guard's v3.0 evolution removes secrets from the model's context entirely and requires a valid access token, retrieved through a separate tool call, before any sensitive data leaves the vault. Applied to multimodal input, the equivalent control is: no image- or audio-derived instruction should be able to authorize a sensitive tool call on its own. A human-in-the-loop confirmation step for exactly this scenario is what Trail of Bits recommends as the strongest available defense against image-based prompt injection, and it holds regardless of which specific encoding trick was used to hide the instruction.

**Trust boundaries at the gateway.** In NeuralStack | MS's AI Security Blueprint, the Prompt Gateway component is defined as the layer responsible for validating input before it crosses into the model's trust boundary. For multimodal deployments, this component needs to do real work on non-text modalities too: canonicalizing image formats and resolutions, transcribing and diffing audio against its own transcript for hidden acoustic content, and enforcing that no modality can silently escalate privilege relative to what a typed instruction would be allowed to do.

### 4\. Detection and Monitoring

Where prevention fails, detection buys time. Activation-patching approaches — comparing model internals on suspected adversarial audio against a denoised or clean reference — have shown promise as a runtime defense for speech-based jailbreaks (SPIRIT). On the video side, diffusion-based adversarial purification techniques originally developed for video recognition (VideoPure) point toward a viable pattern for cleaning perturbations from video frames before they reach a downstream multimodal LLM, though this remains an active research area rather than a production-hardened control.

### 5\. Process and Governance

Threat-model multimodal input explicitly rather than treating it as a footnote to text-based threat modeling. MITRE ATLAS already classifies multimodal and visual prompt injection under its AML.T0051 technique family, alongside steganographic embedding and image-scaling attacks specifically. OWASP's LLM Top 10 revision extends its highest-severity category to the same territory. Red-team your own systems across every modality you accept — not just the one that's easiest to fuzz with existing text-based tooling — and build your reporting templates (CVSS/CWE mappings, reproduction steps) to capture modality-specific detail like interpolation method, sampling rate, or frame index, since that detail is usually what determines whether a fix actually closes the gap.

## Practical Checklist

*   \[ \] Every non-text input is treated as untrusted, regardless of apparent source.
    
*   \[ \] Downscaling/resizing pipelines render and log the post-transform artifact the model actually sees.
    
*   \[ \] OCR/text-in-image detection runs before model ingestion, with confirmation gating on resulting tool calls.
    
*   \[ \] Video pipelines screen every sampled frame, not just the first N seconds of a clip.
    
*   \[ \] A guard-model or equivalent inspection pass sits between raw multimodal input and the primary reasoning model.
    
*   \[ \] Sensitive tool calls require an authorization step that cannot be satisfied by model-interpreted content alone.
    
*   \[ \] Red-team exercises explicitly include image, audio, and video vectors — not text-only jailbreak libraries.
    
*   \[ \] Threat models reference MITRE ATLAS AML.T0051 and the current OWASP LLM Top 10 multimodal guidance.
    

## Conclusion

The shift from text-only to multimodal LLMs did not just add new features — it added new, largely unguarded doors into the same trust-critical instruction-following pathway that text-based prompt injection has spent years teaching us to defend. Image scaling attacks, typographic exploits, acoustic adversarial perturbations, and temporal video injection are not exotic edge cases; several of them already achieve reliable, high attack-success rates against production-grade models. The defenses that hold up share a common shape: don't trust any single layer, validate content before it reaches the model rather than after, and never let a modality the model merely *interprets* carry the same authority as an instruction the user actually *typed*. That is, structurally, the same lesson text-based prompt injection defense has already taught us — multimodal input just makes ignoring it considerably more expensive.

* * *

*For a concrete implementation of the Dual-LLM watchdog and zero-knowledge tool-gating patterns referenced above, see* [*LLM-Guard on GitHub*](https://github.com/MANU-de/LLM-Guard/tree/main)*.*

* * *

\[Co-authored by Claude Sonnet 5\]