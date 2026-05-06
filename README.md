# SaaS Security Compliance Toolkit

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source compliance platform for SOC 2, ISO 27001, HIPAA, and GDPR evidence collection and audit preparation.

The SaaS Security Compliance Toolkit automates evidence collection, control monitoring, and audit preparation for SaaS companies pursuing SOC 2, ISO 27001, HIPAA, and GDPR certification. It targets engineering-led teams who need continuous compliance without the $10K-$80K+/year price tag of incumbent platforms, and pairs agentic automation with full self-hosting for teams that require code inspection and data sovereignty.

---

## Why SaaS Security Compliance Toolkit?

- Incumbents (Vanta, Drata, Secureframe, Sprinto, Thoropass) start at $10K/year and scale to $80K+, leaving seed and Series A teams under-served.
- All major commercial platforms are proprietary SaaS with no self-hosting or air-gapped deployment option, blocking buyers with strict data residency or supply-chain requirements.
- "Alert-only" compliance is now table stakes; buyers in 2026 expect agentic AI for continuous evidence collection, automated remediation suggestions, and auditor Q&A.
- No incumbent correlates threat intelligence (CVEs, breach reports) with a company's specific control gaps to produce a dynamic, risk-aware compliance posture.
- Existing open-source efforts (Comp AI, Shasta) prove demand but remain early-stage with limited auditor familiarity and thin enterprise feature sets.

---

## Key Features

### Multi-Framework Compliance Core

- SOC 2 Type II, ISO 27001, and GDPR support at MVP, with HIPAA, PCI DSS, and FedRAMP planned
- Pre-built control library with map-once-apply-everywhere mapping across frameworks
- Real-time control status dashboard (passing, failing, evidence needed)
- Policy library with version control and attestation workflows
- Full evidence audit trail for regulatory review

### Automated Evidence Collection

- 100+ integrations targeted at MVP (AWS, GCP, Azure, Okta, GitHub, Slack, Jira, etc.)
- Continuous control monitoring across cloud, identity, and CI/CD systems
- Evidence tracker with upload, tagging, and search
- Webhook and API support for custom evidence submission
- Remediation tracker assigning failing controls to owners with due dates

### AI-Native Automation

- Natural-language automation builder: describe an evidence rule in plain English, agent constructs the integration
- AI-powered questionnaire auto-answering from the evidence library
- Automated auditor Q&A that retrieves evidence, synthesises responses, and flags low-confidence answers for review
- Continuous compliance scoring with historical trend analysis and predictive audit-readiness dates

### Vendor Risk and Audit Collaboration

- Vendor risk questionnaire templates with scoring
- Audit collaboration tools: evidence export, auditor portal, Q&A tracking
- Regulatory change digest with impact assessment as standards evolve

### Self-Hosting and DevOps Integration

- Self-hostable via Docker Compose and Kubernetes for closed networks and air-gapped environments
- Policy-as-code support for embedding compliance checks in CI/CD pipelines
- Threat intelligence integration for risk-aware control prioritisation

---

## AI-Native Advantage

Unlike rule-based incumbents, this toolkit applies agentic AI across the compliance lifecycle: agents continuously monitor cloud, SaaS, and CI/CD environments and file evidence against controls without human intervention; LLMs ingest existing policies and infrastructure configuration to produce a prioritised, effort-estimated remediation roadmap; and natural-language policy generation produces auditor-accepted policies from short descriptions of company practices, eliminating the need for expensive GRC consultants. Threat intelligence correlation with the company's specific control gaps yields a real-time, risk-weighted compliance posture that no incumbent currently provides.

---

## Tech Stack & Deployment

The toolkit is designed for both managed cloud and self-hosted deployment via Docker Compose or Kubernetes, addressing the gap left by proprietary incumbents that offer no air-gapped option. Integration breadth follows the pattern set by Vanta (375+), Comp AI (500+), and Drata (200+), with API and webhook support for custom evidence submission and policy-as-code CLI tooling for GitOps-style compliance workflows.

---

## Market Context

The global compliance management software market is projected to reach $8-12 billion by 2027, with compliance automation as the fastest-growing sub-segment. Vanta (~35% share, $203M raised) and Drata (~25%, $328M raised) lead, with Secureframe, Sprinto, and Thoropass filling mid-market and compliance-as-a-service niches. Pricing ranges from $10K to $80K+/year for commercial SaaS; Comp AI's $199-$997/month tiers and free self-hosted option signal the open-source disruption opportunity. Primary buyers are CTOs at Series A/B SaaS companies needing SOC 2 Type 2 to close enterprise deals, security engineers automating control monitoring, and compliance officers running multi-framework programmes.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
