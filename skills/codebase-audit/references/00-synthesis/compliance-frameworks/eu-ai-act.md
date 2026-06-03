# EU AI Act Readiness Mapping

Map findings to **EU AI Act** (Regulation (EU) 2024/1689). Phased
application from Feb 2025 (prohibited practices) through Aug 2027 (high-
risk in regulated products).

Outputs:
- `/docs/audits/compliance/eu-ai-act-readiness.md`
- `/docs/audits/compliance/eu-ai-act-controls.csv`

---

## Live Discovery

- Fetch https://artificialintelligenceact.eu/ — community-maintained
  consolidated text + analysis.
- Fetch https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai — Commission portal.
- Search "EU AI Act guidelines <current year>" — Commission +
  AI Office implementing guidance evolves.
- Search "AI Act high-risk classification" — practical classification
  guidance.

Phased application timeline:

| Date | What enters force |
|---|---|
| **Aug 1, 2024** | Regulation entered force |
| **Feb 2, 2025** | Prohibited AI practices, AI literacy obligations |
| **Aug 2, 2025** | General Purpose AI (GPAI) model rules + governance |
| **Aug 2, 2026** | High-risk AI system rules (most provisions) |
| **Aug 2, 2027** | High-risk AI in regulated products (machinery, toys, medical devices, etc.) |

---

## Scope

Applies to:

- **Providers** placing AI systems on the EU market (regardless of where
  established)
- **Deployers** of AI systems located in the EU
- **Importers and distributors** of AI systems in the EU
- **Product manufacturers** placing AI-incorporating products on the EU
  market
- **Authorised representatives** of providers established outside the EU
- **Affected persons** in the EU (extraterritorial reach when output is
  used in the EU)

Detection signals from Module 18 (AI/ML) + Module 01:
- LLM SDKs in use
- Custom ML models
- Classification / decision-making algorithms (eligibility, scoring,
  ranking)
- Biometric identification / categorization
- Emotion recognition
- Generative AI (text, image, audio, video)
- EU customers receiving AI outputs

---

## Step 1: Risk classification

Each AI system the project uses or provides falls into one of four
risk tiers:

### Unacceptable risk (PROHIBITED — Article 5)

These are banned outright. Audit ensures the project does NOT do any of
these:

- **Subliminal manipulation** that materially distorts behavior
- **Exploiting vulnerabilities** of specific groups (age, disability,
  socio-economic)
- **Social scoring** by public authorities
- **Real-time remote biometric identification** in publicly accessible
  spaces (with narrow law-enforcement exceptions)
- **Predictive policing** based solely on profiling
- **Emotion recognition** in workplace / education (with exceptions)
- **Biometric categorization** to infer race, political opinions, trade-
  union membership, religious / philosophical beliefs, sex life, sexual
  orientation
- **Untargeted scraping** of facial images for facial-recognition databases

If any are detected → **Critical** finding. Immediate remediation required.

### High risk (Annex III — most stringent obligations)

AI systems used in:

- Biometrics (categorization, emotion recognition, identification)
- Critical infrastructure (water, gas, electricity, road traffic)
- Education and vocational training (admissions, assessment, monitoring)
- Employment (hiring, firing, performance evaluation, task allocation)
- Essential private and public services (creditworthiness, insurance,
  public benefits, emergency dispatch)
- Law enforcement
- Migration, asylum, border control
- Administration of justice and democratic processes
- Specific safety-component use in regulated products

If applicable, the AI system has substantial obligations (see Step 2).

### Limited risk (transparency obligations)

- AI systems interacting with humans (chatbots, voice agents) — must
  inform user they're interacting with AI
- AI generating / manipulating image, audio, video content (deepfakes) —
  must label as artificially generated
- AI generating text published to inform public on matters of public
  interest — must label as AI-generated
- Emotion recognition / biometric categorization — must inform persons

### Minimal risk

Most other AI (spam filters, AI-powered video games, recommendation
engines, etc.) — no specific obligations beyond transparency where
applicable.

### General Purpose AI (GPAI) models

A separate classification (Articles 51–55). Models with broad capabilities
(LLMs, large diffusion models). Two tiers:
- **GPAI models without systemic risk** — basic transparency, technical
  documentation, copyright compliance, training data summary
- **GPAI models with systemic risk** (training compute > 10^25 FLOPs OR
  designated by Commission) — additional obligations: risk assessment,
  incident reporting, cybersecurity, energy efficiency

---

## Step 2: For high-risk systems — required practices

| Requirement | Code-auditable | Evidence module |
|---|---|---|
| **Risk management system** (continuous, iterative across lifecycle) | OUT_OF_BAND + Module 18 | — |
| **Data governance** — training/validation/testing data quality | Module 18 | Data inventory |
| **Technical documentation** | OUT_OF_BAND | Maintained per Annex IV |
| **Record-keeping** — automatic logging of events for traceability | Module 18 + 12 | Logging |
| **Transparency and instructions for use** | Module 18 | User-facing docs |
| **Human oversight** — measures enabling natural persons to oversee | Module 18 | UI for human-in-the-loop |
| **Accuracy, robustness, cybersecurity** | Module 18 + 02 | Eval harness + adversarial testing |
| **Quality management system** | OUT_OF_BAND | — |
| **Conformity assessment** before placing on market | OUT_OF_BAND | — |
| **CE marking** | OUT_OF_BAND | — |
| **Registration in EU database** | OUT_OF_BAND | — |
| **Post-market monitoring** | Module 12 + 18 | Production observability |
| **Reporting of serious incidents** to authorities | Module 14 | IR process |

---

## Step 3: Limited-risk transparency

- **Chatbot disclosure.** UI clearly tells the user they're interacting
  with an AI (unless obvious from context). Search for chatbot components,
  AI-assistant features.
- **Generated-content labeling.** AI-generated images / audio / video
  watermarked or labeled as artificially generated.
- **Deepfake labeling.** Same.
- **AI-authored text** in public-interest contexts labeled.

Most consumer SaaS using LLMs falls here — limited-risk + transparency.

---

## Step 4: GPAI usage

If the project *uses* GPAI models (Anthropic, OpenAI, Mistral, Cohere,
etc.):

- The provider's GPAI obligations are mostly the provider's responsibility
  — but as a downstream deployer, the project should:
  - Have access to the provider's documentation (technical docs, training
    data summary)
  - Note any provider-imposed restrictions in deployment documentation
  - Consider downstream modifications (fine-tuning) — if substantial,
    project may become a "GPAI provider" for the modified model

If the project *trains and provides* a GPAI model — substantial
obligations; consult specialist counsel.

---

## Output: markdown report

```markdown
# EU AI Act Readiness Report

Date: YYYY-MM-DD
Repository commit: <hash>
Framework version: Regulation (EU) 2024/1689 (fetched <date>)
Application date relevant: <Feb 2025 prohibited / Aug 2025 GPAI / Aug 2026 high-risk>
Compliance automation: <Vanta / Drata / etc. / DIY>

## AI inventory

| AI system | Purpose | Risk classification | Provider / in-house |
|---|---|---|---|

## Prohibited practices check

- Subliminal manipulation: NOT DETECTED / DETECTED (location)
- Vulnerability exploitation: NOT DETECTED / DETECTED
- Social scoring: NOT DETECTED / DETECTED
- (etc. — full Article 5 list)

## Risk-tier posture

### High-risk systems
(per-system breakdown of Article-related requirements)

### Limited-risk systems — transparency
- Chatbot disclosure present: yes / no
- Generated-content labeling: yes / no / N/A
- Deepfake labeling: yes / no / N/A

### Minimal-risk systems
(noted but no obligations)

## GPAI usage

- Models in use: <list>
- Provider documentation accessible: yes / no
- Project as GPAI deployer obligations met: yes / no
- Project as GPAI provider (if fine-tuning substantially): yes / no

## Top gaps

## Out-of-band checklist
- Risk management system documentation
- Data governance documentation
- Technical documentation per Annex IV (high-risk)
- Quality management system
- Conformity assessment + CE marking + registration (high-risk)
- Post-market monitoring plan
- Serious incident reporting playbook

## Confidence
H / M / L
```

## CSV matrix

Same schema. Rows per AI system + per applicable obligation.

---

## Severity calibration

- **Critical:** prohibited practice detected; high-risk system without
  required documentation/oversight; missing transparency disclosure for
  EU-facing chatbot.
- **High:** high-risk system PARTIAL on key requirements (risk mgmt,
  data governance, oversight); GPAI deployer not maintaining provider
  docs.
- **Medium:** OUT_OF_BAND quality management / technical documentation
  gaps.
- **Low:** disclosure UX polish.

Penalties:
- **Prohibited practices:** up to **€35M or 7% global turnover**
- **Other obligations:** up to **€15M or 3%**
- **Supplying incorrect / misleading info to authorities:** up to **€7.5M
  or 1.5%**

Highest fines in EU digital regulation. Calibrate severity carefully.
