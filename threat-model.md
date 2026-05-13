# Threat Model

> STRIDE applied to the AI system itself, mapped across five deployment topologies along a sovereignty gradient, with a topology-by-topology addendum for regulated and defense workloads.

Most published AI security guidance treats the model as an external risk surface — *what can the LLM be tricked into saying, what data can leak through prompts*. That framing is insufficient for production agentic systems. The threat model below treats the AI system as a distributed, multi-agent execution environment with its own trust boundaries, supply chain, and operational topology. The model changes substantially depending on whether the system runs against a managed API, a sovereignty-qualified cloud, a US sovereign stack on customer infrastructure, self-hosted infrastructure with egress, or an air-gapped enclave. That distinction is rarely made explicit; this document makes it.

## Scope and methodology

In scope: the orchestrator, agents, skills, retrieval pipeline, prompt registry, tool integrations, model inference endpoint, observability pipeline, and the artifacts they consume (model weights, embeddings, knowledge base, configuration).

Out of scope: classical web application security (CSRF, session fixation, etc.) for the API surface — those are addressed by `security/` middleware and are not specific to AI systems.

Method: STRIDE applied to every trust boundary, cross-referenced with OWASP LLM Top 10 (v2.0) and MITRE ATLAS (v4). Each threat is rated for residual risk under each of five deployment topologies, because the same threat changes character substantially across them.

## Trust boundaries

The system crosses eight trust boundaries. Each is a place where authority, data sensitivity, or operational ownership changes. Threats accumulate at boundaries, not inside components.

1. **User ↔ Application** — authentication, session, intent capture
2. **Application ↔ Orchestrator** — prompt construction, context injection, request routing
3. **Orchestrator ↔ Agent** — task delegation, role enforcement, agent identity assertion
4. **Agent ↔ Skill** — capability authorization (which agent can invoke which skill, with which arguments)
5. **Skill ↔ External tool** — outbound API calls, database queries, file I/O
6. **Agent ↔ LLM** — prompt assembly (system + retrieved context + user input) and response handling
7. **LLM ↔ Model artifact** — weights provenance, signature verification, supply chain
8. **System ↔ Telemetry** — what crosses the observability boundary, including PII, secrets, and proprietary domain knowledge

A trust-boundary diagram is provided in `docs/architecture.md`. The key insight: in classical web architectures, the trust boundary count is roughly 3-4. Agentic AI systems double that, and each new boundary is a new place where authority must be re-established and data sensitivity re-evaluated.

## Threat catalog

The catalog cross-walks STRIDE with OWASP LLM Top 10 and MITRE ATLAS. Each row has a primary boundary where it materializes.

| ID | Threat | STRIDE | OWASP LLM | MITRE ATLAS | Boundary |
|---|---|---|---|---|---|
| T01 | Direct prompt injection | T | LLM01 | AML.T0051 | 2, 6 |
| T02 | Indirect prompt injection (via retrieved content) | T | LLM01 | AML.T0051.001 | 2, 6 |
| T03 | Insecure output handling (XSS/SQLi from LLM output) | T | LLM02 | — | 5 |
| T04 | RAG index poisoning | T | LLM03 | AML.T0019 | 6 |
| T05 | Model DoS via cost amplification | D | LLM04 | AML.T0029 | 6 |
| T06 | Supply chain — tampered model weights | T, E | LLM05 | AML.T0010 | 7 |
| T07 | Supply chain — tampered embedding model | T | LLM05 | AML.T0010 | 6, 7 |
| T08 | Sensitive information disclosure via response | I | LLM06 | AML.T0024 | 6 |
| T09 | Secrets leakage via tool argument | I | LLM06 | — | 5 |
| T10 | Insecure tool design (over-privileged skill) | E | LLM07 | AML.T0053 | 4, 5 |
| T11 | Excessive agency (agent action beyond intent) | E | LLM08 | AML.T0053 | 3, 4 |
| T12 | Model theft via inference probing | I | LLM10 | AML.T0044 | 6, 7 |
| T13 | Telemetry channel leakage (prompts in traces) | I | — | — | 8 |
| T14 | Agent identity spoofing | S | — | — | 3 |
| T15 | Repudiation of agent decision | R | — | — | 3, 8 |
| T16 | Jailbreak escalating to skill misuse | E | LLM01, LLM08 | AML.T0054 | 4, 6 |

Two threats deserve emphasis because they are routinely under-treated:

**T11 — Excessive agency.** The default risk in agentic systems is not that the agent fails to act, but that it acts correctly on a wrong premise (poisoned context, injected instruction, or hallucinated intermediate goal). The mitigation is not better prompts; it is contract-bounded skills and per-skill authorization checks that ignore the agent's stated intent.

**T13 — Telemetry channel leakage.** Observability infrastructure typically receives full prompt text, retrieved context, and tool arguments. In multi-tenant deployments or deployments with external SaaS observability vendors, the telemetry path can become the largest data exfiltration channel in the system. It is also the one most often missed in audit, because it is operated by SRE rather than security.

## Deployment topology matrix

The same threat behaves differently across deployment topologies. The matrix below maps the dominant residual risk after standard mitigations are applied. *L* = low, *M* = medium, *H* = high. A topology is not "more secure" in the abstract; it shifts the distribution of residual risk and changes who carries each one.

Five topologies are considered. They form a sovereignty gradient — each step trades provider abstraction for sovereign control. The choice between them is not a security question alone; it is regulatory exposure, operational autonomy, cost model, and contractual responsibility:

- **Managed API** — OpenAI, Anthropic, Azure OpenAI in standard regions. Provider owns the inference plane end-to-end.
- **Sovereign Cloud** — sovereignty-qualified providers (SecNumCloud in France, C5 in Germany, ENS Alto in Spain, ACN in Italy) and the EU sovereign joint ventures (Bleu — Microsoft + Capgemini + Orange ; S3NS — Google + Thales ; Cirrus — Oracle + Thales). Provider runs the inference plane under a sovereignty-qualified or sovereignty-claimed operational framework.
- **US Sovereign Stack** — Azure Stack Hub Sovereign, Azure Local sovereign mode, Google Distributed Cloud Hosted, AWS Outposts in sovereign configuration. US-vendor stack on customer-controlled infrastructure with customer-managed operational boundary.
- **Self-Hosted** — vLLM, Ollama, or equivalent on customer-managed Kubernetes with external network egress. Customer holds the inference plane end-to-end.
- **Air-Gapped** — customer-managed inference plane with no outbound network egress, typical of DGA-contracted and OIV-equivalent workloads.

| Threat | Managed API | Sovereign Cloud | US Sov Stack | Self-Hosted | Air-Gapped |
|---|---|---|---|---|---|
| T01 — Direct prompt injection | M | M | M | M | M |
| T02 — Indirect prompt injection | M | M | M | M | L |
| T03 — Insecure output handling | M | M | M | M | M |
| T04 — RAG index poisoning | M | M | M | M | L |
| T05 — Cost amplification DoS | H | H | M | M | L |
| T06 — Model weights tampering | L | L | M | M | L |
| T07 — Embedding model tampering | L | L | M | M | L |
| T08 — Sensitive disclosure | M | M | M | M | M |
| T09 — Secrets leakage via tool args | M | M | M | M | M |
| T10 — Insecure tool design | M | M | M | M | M |
| T11 — Excessive agency | M | M | M | M | M |
| T12 — Model theft via probing | L | L | M | H | M |
| T13 — Telemetry channel leakage | H | M | L | M | L |
| T14 — Agent identity spoofing | M | M | M | M | M |
| T15 — Repudiation of decision | M | L | L | L | L |
| T16 — Jailbreak to skill misuse | M | M | M | M | M |
| Data residency (regulatory) | M-H | L | L | L | L |
| Operational autonomy | H | M | L-M | L | L |
| Update cadence control | H | M | L | L | L |
| Cost predictability | L | L | M | M | M |
| Auditability | M | L-M | L | L | L |

Four observations drive architecture decisions:

**Sovereign Cloud and US Sovereign Stack are not interchangeable.** Sovereign Cloud shifts operational and audit risk to a sovereignty-claimed provider; US Sovereign Stack shifts the inference plane onto customer infrastructure while keeping the US vendor as the artifact supplier. The two topologies optimize for different anxieties. The JVs play *legal-construct autonomy* for operational simplicity; the US Sovereign Stack plays *physical control* for vendor-lineage continuity.

**The sovereign-managed topologies do not always reduce AI-specific risk.** Hosting Azure OpenAI on Bleu does not change the model artifact provenance — the weights still come from Microsoft's training pipeline. The sovereignty claim covers operations and data flow, not model lineage. This distinction is rarely made explicit in vendor marketing and matters at qualification audit.

**The threats that improve in air-gapped mode are not the threats most public AI security guidance optimizes for.** Prompt injection and excessive agency do not improve with air-gapping. What improves is everything operational — data residency, telemetry leakage, model theft, supply chain. This inverts the typical AI security framing.

**Self-Hosted is the worst position on T12 (model theft).** A model deployed inside a network without provider rate-limiting and tenant isolation is the easiest target for extraction attacks. Self-hosting without an explicit anti-extraction posture is a sovereignty regression, not progress.

## Mitigation mapping

Mitigations live in three layers of the repository: `security/` for runtime guards, `evaluation/` for adversarial coverage, `observability/` for detection and forensics. The mapping below is non-exhaustive; the full coverage matrix lives in `security/coverage.md`.

- **T01, T02, T16** — `security/prompt_injection.py` (input guard + content classifier), adversarial scenarios in `evaluation/red_team/`, anomaly detection on agent loop patterns in `observability/`.
- **T03** — `security/output_filter.py` (strict output schema enforcement, HTML/SQL sanitization at the skill boundary, not at the agent boundary).
- **T04** — `data/ingestion/` provenance enforcement, signed source attestation, retrieval anomaly detection.
- **T05** — token budget enforcement per request, per agent, per workflow; circuit breakers; cost-based rate limiting (not request-based).
- **T06, T07** — model artifact signature verification at load time, fixed SHA-256 pinning, optional Sigstore attestation.
- **T08, T09** — PII redaction at three points: pre-prompt, pre-tool-call, pre-telemetry-emit. Secrets management routed through a dedicated skill that strips arguments from telemetry.
- **T10, T11** — skill contract enforcement: every skill declares allowed argument shapes, side-effect class (read / write / destructive), and required authorization. Agent intent is not authority.
- **T12** — query rate limits, output entropy monitoring, distillation-attack detection patterns.
- **T13** — telemetry redaction policy enforced at the SDK level; PII and prompt content tagged and routed to a restricted retention tier.
- **T14, T15** — signed agent identity assertion, full audit log with hash-chained entries, decision provenance captured per agent turn.

## Sovereignty addendum

Each of the four non-managed topologies has its own operational implications. Deployment manifests live in `sovereignty/`; this section summarizes the qualification-relevant detail.

### Sovereign Cloud — qualified providers

Includes SecNumCloud (France, ANSSI référentiel 3.2), C5 (Germany, BSI), ENS Alto (Spain, CCN), ACN (Italy). State-issued qualifications with audit-backed operational requirements.

For AI systems, the implications:

- **Data localization.** All prompts, retrieved context, and generated outputs are user data. The inference endpoint must be located in a qualified region. Telemetry must be retained in qualified storage.
- **Operational autonomy.** No foreign-controlled entity may have administrative access to the inference plane, the artifact store, or the telemetry pipeline. This is the clause that disqualifies most managed-API offerings even when data residency is technically satisfied.
- **Cryptographic key management.** Sovereign HSM, customer-controlled key flow, Bring-Your-Own-Key support.
- **Model artifact provenance.** The référentiels do not yet specify AI-model lineage requirements explicitly, but the implied requirement under the operational-autonomy clause is that the model artifact supply chain cannot route through foreign-controlled operations. ANSSI's published direction supports this reading.
- **Audit and traceability.** Full reconstruction of any agent decision, hash-chained logging, offline-replicable storage.

### Sovereign Cloud — EU sovereign joint ventures (Bleu, S3NS, Cirrus)

Bleu (Microsoft + Capgemini + Orange), S3NS (Google + Thales), Cirrus (Oracle + Thales) provide a pragmatic middle ground. The legal construct argues for sovereignty through ownership and operational isolation; the underlying technology stack remains the US vendor's.

For AI workloads, the material distinctions from a pure managed API:

- **Operational boundary is real but partial.** The joint venture controls operations within the EU; the technology roadmap, code base, and software updates remain under US-vendor control. For SecNumCloud qualification specifically, the JV must demonstrate independence of operations — a position that has been contested at audit by some parties and accepted by others.
- **Model artifact lineage is unchanged.** Hosting Azure OpenAI on Bleu does not sovereignize the model itself. The weights are trained, signed, and distributed by Microsoft. For regulated workloads where model provenance is a qualification criterion, the JVs do not solve the problem; they bound it operationally.
- **Telemetry boundary.** Telemetry stays within the JV's operational scope. This is meaningfully better than standard managed API, where telemetry typically flows to the US-vendor's global infrastructure plus third-party observability providers.
- **Fit profile.** Customers who need EU operational sovereignty with AI/Office co-deployment continuity. Not appropriate for customers who require sovereign model lineage.

### US Sovereign Stack on customer infrastructure

Azure Stack Hub Sovereign (and Azure Local sovereign mode), Google Distributed Cloud Hosted (including the air-gapped variant), and AWS Outposts in sovereign configuration. The customer holds the bare metal, the network boundary, and the operational keys; the vendor supplies the software, the model artifacts, and the support relationship.

For AI workloads, this topology has properties that neither pure managed nor pure self-hosted can match:

- **Model selection is constrained.** The set of models available on US Sovereign Stack is a subset of what runs in the public cloud. Azure OpenAI on Azure Stack Hub Sovereign exposes a curated model list; GDC Hosted similarly. For agentic systems that depend on specific model capabilities (long context, native tool use, structured output), the topology must be verified against the model roadmap before commitment.
- **Provenance gap.** Model weights are still produced and signed by the US vendor. Air-gapped variants of GDC support offline artifact ingestion under signed manifests, mitigating the supply chain risk without resolving the lineage question.
- **Operational autonomy is high but bounded.** The customer controls who has physical and logical access. The vendor retains code-base ownership and update authority. For most regulated workloads this is acceptable; for the strictest defense workloads, the air-gapped configuration is required.
- **Fit profile.** Customers under data residency obligation who need US-vendor model capabilities, customers who want sovereignty without giving up the strategic vendor relationship, defense workloads where the customer owns the bare metal under a known software stack.

### Air-gapped defense operations

For DGA-contracted and OIV-equivalent workloads, the air-gapped topology adds operational requirements beyond what sovereignty-qualified or US Sovereign Stack topologies demand:

- **Red/black network separation.** The inference plane operates entirely within the trusted enclave. No outbound network egress, including for telemetry, license validation, or model update checks. License and entitlement state are validated offline against signed manifests.
- **Model artifact ingestion procedure.** Weights and embedding models cross the air gap through a controlled procedure: physically transferred signed artifact, integrity verification against an offline-validated trust anchor, manual approval before promotion to the inference plane.
- **Inference-only execution.** No fine-tuning, no continued pre-training, no gradient updates inside the enclave. Any model adaptation happens outside and re-enters as a new signed artifact. This eliminates a class of supply-chain-flowing-backwards attacks.
- **Compartmented logging.** Telemetry is captured locally, encrypted with enclave keys, and made available only through controlled export procedures. The default assumption is that logs themselves are classified.
- **Inference observability without external SaaS.** Langfuse and equivalent vendors are not deployable inside an air gap. The `observability/` layer must support a fully self-contained backend (Postgres + S3-compatible object store + local UI) with no external dependencies.

The `sovereignty/` directory contains the deployment manifests for each topology. Same agent code, same skills, same domain pack — five operational envelopes.

## Known gaps and open questions

A threat model that claims completeness is unfinished. Current gaps:

- **Multi-agent collusion.** STRIDE does not cleanly model emergent behavior across cooperating agents. The catalog above treats agents as independent; in practice, two agents with complementary partial authority can compose into unintended capability. Initial mitigation lives in the orchestrator's authorization model, but the threat is not fully characterized.
- **Long-running memory poisoning.** Agents with persistent memory accumulate state that survives the request boundary. The threat model treats memory as a retrieval source (T04) but does not yet address slow-drift poisoning over weeks or months.
- **Inter-tenant isolation in shared inference.** Self-hosted and US Sovereign Stack multi-tenant deployments share GPU memory and KV-cache. Side-channel risk under this model is documented in research but is not yet operationally characterized for this architecture.
- **Regulatory evolution.** SecNumCloud 4.x, the EU AI Act implementing acts, and NIS2 transposition will all add specific obligations within the next 12–24 months. This document is versioned against the references as of its date.

Contributions and challenges to this threat model are welcome through pull request or direct contact.

---

*Last reviewed: 2026-05. References: STRIDE (Microsoft), OWASP LLM Top 10 v2.0, MITRE ATLAS v4, ANSSI référentiel SecNumCloud 3.2, BSI C5, CCN ENS Alto.*
