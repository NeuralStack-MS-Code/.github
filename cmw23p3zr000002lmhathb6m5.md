---
title: "AI-Powered Social Engineering: Hyper-Personalized Phishing, Deepfake Vishing, and Automated Disinformation"
datePublished: 2026-11-04T12:36:27.624Z
cuid: cmw23p3zr000002lmhathb6m5
slug: ai-powered-social-engineering-hyper-personalized-phishing-deepfake-vishing-and-automated-disinformation
cover: https://cdn.hashnode.com/uploads/covers/68e922a757e675c5840506dd/33cd3eb6-3195-48d4-a90c-b4a8aa9bcc71.jpg
ogImage: https://cdn.hashnode.com/uploads/og-images/68e922a757e675c5840506dd/1c75ed80-b683-47ec-b632-a8ebf789730c.jpg
tags: osint, red-teaming, threat-modeling, llm-security, eu-ai-act, aisecurity, llm-grooming

---

Nov 4, 2026 · @Manuela Schrittwieser · NeuralStack | MS – AI Security Engineering

## Introduction

Generative AI has not created a new class of social engineering attack; it has removed the cost constraints that previously limited the existing classes. Phishing, vishing and influence operations all relied on a trade-off between scale and quality: attackers could write many generic messages or a few tailored ones. Large language models (LLMs), voice-cloning models and image or video generators collapse this trade-off.

This article examines three attack families in which the shift is most visible: hyper-personalized phishing, deepfake vishing and impersonation, and automated disinformation. For each, it describes the mechanism, the documented evidence and the limits of the technique. It then derives a layered defensive model and a red-team approach for testing it.

The central argument is that detection of AI-generated content is a weak primary control. The more durable defenses are architectural: authenticated channels, cryptographic provenance where available, and business processes that do not depend on a person judging whether a voice, face or message is genuine.

## Threat model: what AI changes

Generative AI changes the economics of social engineering in four ways, while the psychology it exploits (urgency, authority, familiarity) stays the same.

- **Scale.** An automated pipeline produced personalized phishing at roughly four cents per email in the [Heiding et al. study](https://www.schneier.com/wp-content/uploads/2026/02/Evaluating-Large-Language-Models-Ability-to-Automate.pdf).
- **Personalization.** Browsing agents assemble target profiles from public sources; 88% were accurate and sufficient in that study.
- **Fluency.** Errors of grammar and layout no longer separate fraud from legitimate mail.
- **Interaction.** Voice agents hold adaptive conversations, as in the [ShinyHunters-linked campaigns](https://blog.eclecticiq.com/shinyhunters-calling-financially-motivated-data-extortion-group-targeting-enterprise-cloud-applications), which moves attacks from static messages to live dialogue.

The chain below applies to all three attack families discussed in this article. Each stage offers a point of intervention, but only the last does not depend on recognizing a fake.

![](https://cdn.hashnode.com/uploads/covers/68e922a757e675c5840506dd/da48e05c-4f8f-439f-a3d3-736794220d50.png align="center")

Controls on the first three stages reduce the probability of a successful attack; process controls at the victim's action step remain effective even if the forgery is flawless.

## Hyper-personalized phishing

Fully automated, OSINT-driven spear phishing now matches human experts in click-through rate at a fraction of the cost. The most rigorous recent evidence is a controlled study by [Heiding et al. (Expert Systems With Applications, 2026)](https://www.schneier.com/wp-content/uploads/2026/02/Evaluating-Large-Language-Models-Ability-to-Automate.pdf), which sent four types of email to 101 participants.

| Group | Click-through rate |
| --- | --- |
| Generic control (scam text) | 12% |
| Human expert, semi-personalized | 54% |
| Fully AI-automated, hyper-personalized | 54% |
| AI with human review | 56% |

The researchers' tool chained five steps, none of which required a human:

1. Target collection from public records, news and social media.
2. Autonomous OSINT profiling by a web-browsing agent (GPT-4o); the profile was accurate and sufficient in 88% of cases.
3. Message generation (Claude 3.5 Sonnet) from the profile and a prompt template built on Cialdini's principles and the V-Triad model.
4. Batch delivery, timed and throttled to avoid spam filters.
5. Outcome tracking that feeds back into prompts and templates.

The authors report about four cents per email for the fully automated variant, against roughly 34 minutes per target for a manual equivalent. Their economic model estimates that automation raises phishing profitability by up to 50× in large campaigns.

Three findings matter for defenders:

- **Guardrails are a weak control.** Rewording "phishing email" to "email" was enough to bypass refusals, and the reconnaissance agent was never refused. The authors attribute this to weaker safeguards in agentic, tool-using settings.
- **Language quality no longer discriminates.** 71% of the AI-written emails in 2024 needed no edits, against 25% in the authors' 2023 comparison. Spelling and grammar are obsolete indicators.
- **The channel is shifting.** The [2026 Verizon DBIR](https://www.pymnts.com/fraud-prevention/2026/verizon-report-warns-of-ai-fueled-social-engineering-surge/) (more than 31,000 incidents, 22,000 confirmed breaches, 145 countries) attributes 16% of breaches to social engineering and finds the human element in 62%. Simulated mobile-centric attacks reached a median click rate 40% higher than email. Verizon's reading is that AI currently scales known techniques rather than creating new ones.

Two limits apply. The study used 101 university-affiliated participants, measured clicks rather than downstream compromise, and relied on 2024-era models; it demonstrates capability, not prevalence. On the defensive side, the same paper reports that Claude 3.5 Sonnet flagged 97.25% of 381 test emails with no false positives when prompted to look for suspicion. That dataset was heavily imbalanced (363 phishing, 18 legitimate emails), and detection of conventional phishing was lower, at 81%.

## Deepfake vishing and video impersonation

Synthetic voice and video attack the verification step itself: a call or video meeting, long treated as proof of identity, no longer is one. Three documented patterns show how this plays out.

| Case | Technique | Control that failed |
| --- | --- | --- |
| [Arup, Hong Kong (reported to police January 2024)](https://abc17news.com/money/cnn-business-consumer/2024/05/16/british-engineering-giant-arup-revealed-as-25-million-deepfake-scam-victim/) | Phishing email, then a video call with deepfaked CFO and colleagues | Visual and audio recognition used to confirm a payment request |
| [Impersonation of senior US officials (since April 2025)](https://www.gov1.com/technology/fbi-smishing-vishing-campaign-impersonating-senior-u-s-officials) | SMS plus AI-generated voice messages, then redirection to other platforms | Familiarity with a known voice and name |
| [ShinyHunters-linked vishing against enterprise SSO (2025)](https://blog.eclecticiq.com/shinyhunters-calling-financially-motivated-data-extortion-group-targeting-enterprise-cloud-applications) | Commercial AI voice-agent platforms placing adaptive calls as IT support | Help desk and employee trust in an internal-sounding caller |

**Arup.** According to Hong Kong police as reported by CNN, the finance worker first suspected a phishing email asking for a confidential transaction, then set those doubts aside after a video call in which everyone appeared and sounded like known colleagues. He sent HK$200 million (about US$25.6 million) in 15 transfers. Arup confirmed that fake voices and images were used and that its internal systems were not compromised. The call functioned as the control, and the control was defeated.

**Official impersonation.** The FBI's IC3 advisory of 15 May 2025 describes texts and AI-generated voice messages posing as senior officials, aimed at current and former federal and state officials and their contacts. The goal was to move targets to other platforms or links to harvest credentials and take over accounts, which are then used to reach further contacts.

**AI voice agents at scale.** EclecticIQ reports with high confidence that ShinyHunters hired some Scattered Spider members for voice phishing, and that affiliates abused commercial voice-agent platforms (Vapi, Bland) to automate calls that adjust to the victim's answers in real time. Callers posed as IT staff and walked employees into authorizing attacker-controlled applications in Salesforce. These agents are synthetic personas rather than clones of a specific person, which is a distinct and cheaper capability: conversation at scale, not one convincing impersonation.

**Human detection is not a control.** In a study by Lavan et al. ([PLOS One, reported by The Register](https://www.theregister.com/software/2025/10/09/humans-flunk-the-turing-test-for-voices-as-bots-get-chattier/318345)), listeners judged 80 samples. Voice clones built with off-the-shelf software from under five minutes of speech were rated as human 58% of the time, against 62% for genuine voices; the authors found no significant sensitivity. Fully synthetic voices were easier to spot.

Two analytical points follow. First, attackers usually pair synthetic media with pre-staged context (a prior email, a spoofed number, a plausible pretext) and manufactured urgency, so the weakest element is the process, not the audio. Second, impersonation of one specific person depends on a single trusted identity; a process that requires confirmation through an independent, pre-registered channel removes that dependency regardless of how good the clone is.

## Automated disinformation

AI lowers the cost of running persona networks, localizing content and managing engagement; the documented reach of such operations remains limited so far, while a newer risk is aimed at the AI systems that people use to look things up. Three lines of evidence illustrate the range.

**Persona orchestration.** In an April 2025 report, [Anthropic described](https://thehackernews.com/2025/05/claude-ai-exploited-to-operate-100-fake.html) a commercial "influence-as-a-service" operation that ran about 100 social media personas on X and Facebook, engaging with tens of thousands of authentic accounts. What the company called a novel was the use of the model as an orchestrator: it decided whether each bot account should comment, like, re-share or ignore a post within a JSON-based persona framework. The operation favored persistence over virality and appeared to serve several clients.

**Commercial and false-front operations.** Anthropic's [September 2026 report](https://theedgemalaysia.com/node/817799) describes a paid platform that used Claude to build a voter-profiling tool covering all 222 Malaysian parliamentary constituencies, to run roughly 1,000 fake X accounts and to operate a fabricated news outlet that rewrote real articles under fake bylines. When Claude refused requests it identified as political defamation, the operator negotiated sanitized wording. Engagement figures were self-reported by the operator, and Anthropic found no evidence linking the operation to the prime minister's office. OpenAI's [October 2026 report](https://www.unite.ai/openai-bans-two-covert-influence-operations-using-false-fronts/) covers a Russia-origin operation rated Category 5 on its six-point Breakout Scale (the first it has assigned that rating) and an Iran-origin operation that placed almost 100 articles under seven fake journalist personas. OpenAI notes that both resemble pre-AI operations, with AI easing individual workflows, and that the operators' internal impact metrics greatly overstated their effect.

**Targeting the models.** NewsGuard's [audit of ten leading chatbots](https://www.newsguardrealitycheck.com/p/a-well-funded-moscow-based-global) (early 2025; 450 responses to 15 false narratives) found that they repeated claims from the Moscow-based Pravda network 33.55% of the time, and 56 responses linked to its articles. The network publishes at industrial scale (an estimated 3.6 million articles in 2024 across about 150 domains) yet draws almost no human readers; its apparent audience is crawlers and retrieval systems. The American Sunlight Project calls this "LLM grooming". Blocking known domains does not solve it, because the network republishes material from other sources and adds new domains.

The evidence has limits. Most findings come from the vendors who detected the operations, so they describe what was caught, not what was missed. Operator-reported results are unreliable, and reach varies widely: OpenAI rates the Iranian operation's comment campaign at Category 2 but its article placement in real outlets at Category 4, and notes that operations landing content in genuine media tend to have the highest potential reach. For defenders, the practical consequence is attribution and provenance work: assessing who is amplifying a narrative, how coordinated the accounts are, and which sources feed retrieval-augmented systems inside the organization.

## Countermeasures

Because people cannot reliably judge synthetic voices and software detection is probabilistic, the durable defences are those that do not depend on judging authenticity. The table lists the main controls by layer, with the basis for each and its limit.

| Layer | Measure | Basis | Limit |
| --- | --- | --- | --- |
| Payment and help-desk process | Call-back to a pre-registered number or internal directory entry; code phrase or second approver for MFA resets and payments; no override for urgency or seniority | [EclecticIQ](https://blog.eclecticiq.com/shinyhunters-calling-financially-motivated-data-extortion-group-targeting-enterprise-cloud-applications) and the [FBI advisory](https://www.gov1.com/technology/fbi-smishing-vishing-campaign-impersonating-senior-u-s-officials) both advise verifying through a known channel | Only works if the channels are registered in advance and enforced for executives too |
| Authentication | Phishing-resistant MFA (FIDO2 keys) for privileged and data-rich roles; never disclose one-time codes | EclecticIQ recommends FIDO2 for staff managing sensitive data | Does not stop a user from authorizing an attacker-controlled connected app |
| SaaS hardening | Least privilege for export tools; allowlist connected apps; alerts on new MFA enrolment, privileged resets and bulk exports | EclecticIQ's Salesforce and Okta case analysis | Operational overhead; needs log coverage |
| Message filtering | LLM-assisted detection, prompted to assess suspicion; URL sandboxing; BEC checks | [Heiding et al.](https://www.schneier.com/wp-content/uploads/2026/02/Evaluating-Large-Language-Models-Ability-to-Automate.pdf): 97.25% detection, no false positives on 381 emails; priming for suspicion improved results | Imbalanced test set (18 legitimate emails); 81% on conventional phishing; attackers adapt |
| Provenance and labelling | C2PA Content Credentials, watermarks, EU AI Act Article 50 duties | [Article 50 applies from 2 August 2026](https://compliancehub.wiki/eu-ai-act-marking-labelling-code-of-practice-article-50-2026/), with a transition to 2 December 2026 for marking by systems already on the market | [Metadata can be stripped, forged or bypassed by re-recording](https://newsletter.ai-frontiers.org/p/ai-content-must-now-carry-a-label); does not cover live calls |
| Awareness training | Include voice, SMS and collaboration-tool scenarios; train staff to call back on unexpected IT requests | EclecticIQ; Heiding et al. note that training alone is unlikely to suffice against AI-enhanced attacks | Measure whether people verify through a second channel, not only whether they click |

Provider-side controls matter but are not sufficient. Anthropic and OpenAI report detecting and banning the operations described above, yet Heiding et al. bypassed refusals by rewording the request, and the Malaysian operator negotiated around refusals. Organizations should treat model guardrails as one signal among several, not as a barrier.

Four design principles follow from the evidence (analysis, not a quotation from any source):

1. **Assume the face and voice can be forged.** No approval should rest on recognition alone.
2. **Bind high-risk actions to channels, not identities.** A request is valid because it arrived or was confirmed through a registered channel.
3. **Limit what any single person can do.** Dual control for payments, MFA resets and application authorizations removes the single point that social engineering targets.
4. **Log for the help desk as you do for the network.** Resets, enrolments and unusual connected-app grants are the detectable footprint of a successful call.

On provenance, the regulatory timeline is still moving: the Code of Practice on marking is voluntary, and the 2 December 2026 transition derives from the Digital Omnibus, which the cited June 2026 analysis describes as still provisional, so teams should confirm the current legal status with counsel before relying on any date.

## Red team perspective

A red team should test whether a convincing forgery is stopped by a control, not whether an employee can spot it. This section is practical analysis for authorized engagements; it describes test design, not attack tooling.

**Scenarios worth testing**

1. **Help-desk pretext call.** A caller using a synthetic persona, not a clone of a real employee, requests an MFA reset or an application authorization. The question is whether the call-back and second-approver rules hold under mild pressure.
2. **Executive payment request in a sandboxed workflow.** A staged request, with no real funds, attributed to a senior role and reinforced by a video or voice call. The question is whether the payment process accepts the call as verification, as it did in the Arup case.
3. **OSINT exposure review.** Measure what a profiling agent could assemble about key staff from public sources. Heiding et al. propose turning such profiles into personalized guidance on what to remove from public view.
4. **Narrative tabletop.** Walk through a coordinated campaign against the organization (fake outlet, persona accounts, forged audio) and test escalation paths, legal contacts and platform takedown routes.
5. **Retrieval hygiene.** Check whether internal assistants or search tools ingest unvetted web sources, which is the exposure that LLM grooming exploits.

**Rules of engagement**

- Written authorization covering the channels used (phone, SMS, collaboration tools) and named exclusions.
- No cloning of real employees' or executives' voices or likenesses without their documented consent. Voice and image are personal data, so involve legal counsel and, where applicable, the works council before any such test.
- No real financial transfers and no live credential capture from staff; use sandboxed or simulated back-ends.
- Data minimization in OSINT: collect only what the scenario needs, and delete it after the debrief.
- A debrief for every targeted person, framed around the process failure rather than individual blame.
- Restraint in publication. The authors of the Heiding study withheld their source code and prompts because of dual-use risk; the same caution applies to internal tooling and write-ups.

**Metrics that reflect the real objective**

- Share of high-risk requests verified through a second channel.
- Time from suspicious contact to report, and from report to containment.
- Whether help-desk and finance controls failed open under urgency or seniority claims.
- Whether the security operations team detected the reset, enrolment or connected-app grant that the test produced.

Click rates alone are a poor measure here: they capture the first step of the email attack, while the voice and video scenarios succeed or fail at the verification step.

## Outlook and key takeaways

The direction of travel is towards cheaper, more conversational and more mobile attacks, while the defensive advantage lies in process design rather than in better detection of fakes. Verizon's 2026 data already shows mobile-centric social engineering outperforming email in simulations, commercial voice-agent platforms are being used for vishing, and influence operators are experimenting with targeting the retrieval layer of AI systems.

The evidence has gaps. Controlled studies such as Heiding et al. measure capability on small samples, vendor reports describe only what vendors detected, and operators' own impact claims are unreliable. Prevalence in the wild is not well measured.

**Key takeaways**

- AI removes the trade-off between scale and personalization: fully automated spear phishing matched human experts at 54% click-through in a controlled study.
- Synthetic voice is not reliably detectable by listeners, and a video call is no longer proof of identity.
- Documented influence operations show AI easing operators' workflows, with reach that varies widely; the newer exposure is poisoning of what AI assistants retrieve.
- Technical detection helps but is probabilistic; provenance standards help but can be stripped and do not cover live calls.
- The most durable controls are out-of-band verification through pre-registered channels, phishing-resistant authentication, dual control for high-risk actions and monitoring of help-desk and connected-app events.
- Red-team exercises should measure verification behaviour and control failure under pressure, using consented or synthetic identities.

## Sources

All pages below were opened on 4 November 2026. Where the primary document could not be opened, the secondary report is named as such.

- [Heiding et al., "Evaluating large language models' ability to automate spear phishing", Expert Systems With Applications 314 (2026)](https://www.schneier.com/wp-content/uploads/2026/02/Evaluating-Large-Language-Models-Ability-to-Automate.pdf)
- [PYMNTS, "Verizon Report Warns of AI-Fueled Social Engineering Surge" (summary of the 2026 Verizon DBIR; secondary report)](https://www.pymnts.com/fraud-prevention/2026/verizon-report-warns-of-ai-fueled-social-engineering-surge/)
- [CNN via ABC17, "British engineering giant Arup revealed as $25 million deepfake scam victim"](https://abc17news.com/money/cnn-business-consumer/2024/05/16/british-engineering-giant-arup-revealed-as-25-million-deepfake-scam-victim/)
- [Gov1, "FBI: Smishing, vishing campaign impersonating senior U.S. officials" (summary of IC3 PSA I-051525; secondary report)](https://www.gov1.com/technology/fbi-smishing-vishing-campaign-impersonating-senior-u-s-officials)
- [EclecticIQ, "ShinyHunters Calling" (22 September 2025)](https://blog.eclecticiq.com/shinyhunters-calling-financially-motivated-data-extortion-group-targeting-enterprise-cloud-applications)
- [The Register, "Humans flunk the Turing test for voices as bots get chattier" (report on a PLOS One study by Lavan et al.)](https://www.theregister.com/software/2025/10/09/humans-flunk-the-turing-test-for-voices-as-bots-get-chattier/318345)
- [The Hacker News, "Claude AI Exploited to Operate 100+ Fake Political Personas" (report on Anthropic's April 2025 threat report; secondary report)](https://thehackernews.com/2025/05/claude-ai-exploited-to-operate-100-fake.html)
- [The Edge Malaysia, "Anthropic reveals it disrupted election-manipulation platform targeting Malaysian voters" (report on Anthropic's September 2026 threat report; secondary report)](https://theedgemalaysia.com/node/817799)
- [Unite.AI, "OpenAI Bans Two Covert Influence Operations Using False Fronts" (report on OpenAI's 8 October 2026 report; secondary report, as OpenAI's own page blocked automated access)](https://www.unite.ai/openai-bans-two-covert-influence-operations-using-false-fronts/)
- [NewsGuard, "A well-funded Moscow-based global 'news' network has infected Western artificial intelligence tools"](https://www.newsguardrealitycheck.com/p/a-well-funded-moscow-based-global)
- [ComplianceHub.Wiki, "Watermarks and Metadata: How to Actually Comply With the EU AI Act's Article 50" (15 June 2026; secondary analysis, not legal advice)](https://compliancehub.wiki/eu-ai-act-marking-labelling-code-of-practice-article-50-2026/)
- [AI Frontiers, "AI Content Must Now Carry a Label. Cameras Are Next." (13 August 2026; the author discloses working for a provenance-infrastructure company)](https://newsletter.ai-frontiers.org/p/ai-content-must-now-carry-a-label)