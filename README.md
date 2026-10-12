# Creative Disability Casting for Film and TV Drama

Guidance and machine-readable artifacts for integrating disability as part of the creative casting process. This repository is intended for two main audiences:
- Studio AI management systems and platform engineers integrating automated talent-discovery or decision-support agents.
- Creative teams (producers, casting directors, accessibility consultants, publicists) using the guidance to shape inclusive, rights-respecting casting and publicity practice.

Core use cases: talent discovery, casting evaluation, script analysis, and production decision recording.

---

## Quick links (top-level files)
- AGENTS.md — Agent policy document, rules, machine-readable schemas and audit/observability guidance (agent-run rules and required fields).
- Creative Disability Casting Guidelines.md — Human-readable practical guidance and recommended creative workflows.
- Creative_Disability_Casting_Reference.md — Detailed reference materials, terminology, evidence sources and examples.
- Publicity Guidance_ Telling the Disability Story of a Production.md — Guidance for public communications and responsible storytelling.
- Studio Policies and Commitments (UK and US).md — Suggested studio-level policy templates and legal/commitment considerations.
- LICENSE — Project license (CC BY 4.0).

---

## What this repository contains
- Policy artifacts (AGENTS.md) designed for programmatic consumption by agents: rules, JSON Schemas, observability and redact rules, and decision-record templates.
- Practical guidance for casting teams and creatives: how to assess authenticity, design access summaries, and structure consultations.
- Publicity and communications guidance to ensure respectful representation of performers.
- Studio policy templates to support implementation across UK and US jurisdictions.

---

## For studio AI management systems — how to integrate

Design principles
- Preserve human ownership: automated systems must not finalize casting decisions. Every recommendation must include a named human decision_owner (see R001).
- Forbid medical inference: agents must not infer diagnoses, impairment from images, or neurotype from bios (see R002).
- Audit everything: emit an audit record for every run with prescribed fields (see R003). Keep audit logs in restricted storage with a long retention policy.
- Escalate where appropriate: escalate when a non-disabled candidate is recommended for a role explicitly designated for disabled performers (R004).

Machine-readable artifacts to ingest
- AGENTS.md — contains:
  - Rules (R001–R004): enforceable triggers and blockers.
  - DecisionRecord JSON Schema: canonical output schema for longlist/shortlist/final_decision and audit metadata.
  - Observability/audit spec and redact rules (patterns and replacement tokens).
- DecisionRecord schema (draft-07) — use as the canonical contract for any agent output intended to become a stored decision record.

Integration checklist (minimum)
1. Validate every agent output against the DecisionRecord JSON Schema.
2. Enforce R002 by blocking any feature extraction pipeline that attempts to predict medical/diagnostic attributes.
3. Ensure every recommendation includes decision_owner and that human review is required before final_decision is set.
4. Persist audit_record per the observability spec: include agent_run_id, agent_version, data_sources_snapshot, top_results, timestamp, and any human_overrides.
5. Apply redact_rules before any logs or exported data leave restricted storage.

Example: ingest flow
- Agent runs search → produces `agent_query` and candidate longlist → validate schema → emit audit_record (signed/immutable) → place result into a human review queue with decision_owner assigned → human reviewer records final_decision in DecisionRecord and signs off.

APIs and formats
- Accept and emit JSON following the DecisionRecord schema.
- Store audit logs in a restricted storage class with retention_days = 3650 (per observability spec).
- When exposing candidate evidence links, only include URIs to verified sources (self-hosted portfolios, talent agencies, or consented profiles).

Operational notes
- Provide an explainability bundle per recommendation: top contributing signals, confidence bands, and the query that produced the result.
- Maintain a versioned data_sources_snapshot for reproducibility.
- Run periodic policy checks to ensure the agent hasn’t regressed (unit tests around forbidden-inference detectors, schema validators, and audit emission).

---

## For creative teams — how to use these materials

Principles for creatives
- Prioritise authentic representation: combine lived experience consultation with demonstrable skills and credits.
- Ask for consent and agency: use access_summary created by performers to understand practical access needs (non-medical).
- Avoid medicalisation: do not request or record medical diagnoses; focus on functional access and on set adjustments.

Suggested workflow (casting table)
1. Define role: mark essential_characteristics vs adaptable_characteristics in the DecisionRecord.
2. Create agent_query (if using automated search) with role filters limited to skills, credits, self-declared access_summary, location, and availability.
3. Produce a longlist and verify evidence_links for each candidate (verification_status = verified/unverified/verification_pending).
4. Human review/consultation with disability advisors and the named decision_owner.
5. Record final_decision, final_rationale, any dissenting_advice, and review_date_or_trigger.

How to evaluate candidates
- Use skills_and_credits, showreel/evidence_links, and candidate-provided access_summary as primary inputs.
- Verification steps should be human-led: contact references, confirm credits, and confirm access arrangements.
- If a role explicitly requires a disabled performer, flagged recommendations that conflict must be escalated to the decision_owner.

Publicity & storytelling
- Follow the publicity guidance document before publishing casting or accessibility stories. Consent, accurate language, and performer agency are mandatory; never publish medical or diagnostic information.

---

## Decision record — sample (illustrative)
This is a minimal example showing required fields; agents must validate produced JSON against the canonical schema in AGENTS.md.

{
  "production": "Example Show S1",
  "role": "Jamie, late 20s, wheelchair user (role requires a wheelchair user performer)",
  "decision_owner": "Head of Casting — name@example.com",
  "agent_query": { "skills":"theatre,screen", "access_summary_required":"wheelchair_user" },
  "longlist": [
    {
      "candidate_id": "cand-2026-001",
      "evidence_links": ["https://portfolio.example/cand-2026-001"],
      "skills_and_credits": ["TV Series X (supporting)","Theatre Y (lead)"],
      "access_summary": "User-provided: uses manual wheelchair, needs ramp access and seating on set",
      "verification_status": "verification_pending"
    }
  ],
  "shortlist": ["cand-2026-001"],
  "essential_characteristics": ["wheelchair_user", "age_range:25-32"],
  "final_decision": "shortlist",
  "final_rationale": "Candidate has required lived experience and screen credits; human verification to complete.",
  "timestamp": "2026-10-12T12:00:00Z",
  "agent_run_id": "run-uuid-0001"
}

(Replace example emails and IDs with real values and remove any personal data before publishing.)

---

## Governance, audits and privacy

- Human ownership required: every decision must have a named decision_owner (R001).
- No medical inferences: system designs must explicitly prevent and test against inferences about diagnosis or neurotype from images or bios (R002).
- Audit records: agents must emit audit records with prescribed fields; store them securely and apply redact rules before exports.
- Redaction: AGENTS.md includes example regex-based redaction patterns for email, phone, national ID, and medical terms; apply these before logs or exports leave the restricted environment.
- Review cycle: repository review metadata lists 6-month review cadence; adopt the same cadence for policy and model audits.

---

## How this helps legal & studio policy teams
- Provides templates and practical obligations for studio policy integration (see Studio Policies and Commitments).
- Supplies machine-readable rules for enforcement by governance tooling (policy-as-code).
- Enables traceability from an agent-produced recommendation to the human-approved decision record and public-facing communications.

---

## How to contribute
- Issues and pull requests welcome. Before contributing:
  - Read the AGENTS.md (policy and schema) and keep changes backwards-compatible for agents.
  - For changes that affect rule semantics (e.g., R002–R004), open an issue describing the proposed change, rationale, and migration plan.
- For content contributions (guidance, case studies), include sources and consent statements for any quoted performer material.

---

## Tests and validation recommendations for engineers
- Schema validation unit tests for DecisionRecord outputs.
- Policy regression tests ensuring no forbidden-inference features are accessible to models.
- End-to-end integration tests for audit emission and redact pipeline.
- Human-in-the-loop acceptance tests: ensure every recommended record cannot be marked final without a decision_owner signing off.

---

## Contacts and maintenance
- Repository owner: Double FW
- Review cadence: every 6 months (see AGENTS.md review metadata)
- Related projects: Authentic-Representation-Project, Drama-and-Disability (links inside AGENTS.md)

---

## License
This repository is licensed CC BY 4.0 (see LICENSE). Use and adaptation are permitted with attribution; consult the LICENSE for details.

---
