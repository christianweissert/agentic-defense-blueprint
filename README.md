# Agentic Defense Blueprint

> Vertical AI for security operations. Sovereign Design

Most AI repositories ship horizontal frameworks — a chat interface, a RAG pipeline, a generic agent loop. They demo well but reveal nothing about how to actually deploy AI inside an enterprise that must answer to a CISO, an auditor, and a regulator.

This repository is a production-grade reference architecture for **vertical AI systems** — AI workforces that operate inside a specific domain, govern themselves end-to-end, and remain deployable in highly regulated or air-gapped environments.

## Wedge: CISO Decision OS
The reference vertical is a **CISO Decision OS**: a set of agents that reason over security frameworks (MITRE ATT&CK, NIST CSF, ISO 27001), score and prioritize incidents, and propose remediation backed by traceable evidence.

* **Input:** Incident artifacts, vulnerability feeds, asset inventory, threat intel.
* **Output:** Prioritized incidents with MITRE-mapped technique chains, remediation playbooks with effort/impact scoring, and audit-grade evidence trails.

## Architecture

The system is built on three architectural layers:

1. **Domain Brain — the moat:** Ontology, playbooks, policies, and KPIs that encode how security operations actually work. The domain pack is the IP, not the agents or the LLM.
2. **Agent Workforce:** Specialized agents (Orchestrator, Incident Analyst, Remediation Planner, Reviewer) that reason and coordinate. Each agent has a contract, not just a system prompt.
3. **Skill Layer:** Atomic, reusable capabilities (retrieve, classify, score, map_attack) that agents compose. Skills are testable in isolation and observable per call.

## Sovereignty by Design
The architecture is designed to be topology-portable, supporting five deployment modes along a sovereignty gradient:
* **Managed API** (e.g., OpenAI, Anthropic).
* **Sovereign Cloud** (e.g., SecNumCloud, C5, ENS Alto).
* **EU Sovereign JVs** (e.g., Bleu, S3NS, Cirrus).
* **US Sovereign Stack** (e.g., Azure Stack Hub, GDC Hosted, AWS Outposts).
* **Air-Gapped Enclaves** (DGA-contracted and OIV-equivalent workloads).

## Stack
FastAPI · LangGraph · Postgres · Redis · Weaviate · Langfuse · Docker · Kubernetes · Terraform

---
*See `docs/architecture.md` for the full rationale and `docs/threat-model.md` for the STRIDE analysis of the AI system itself.*
