---
title: "AGENTS"
version: "1.1"
author: "Double FW"
date: "2026-10-12"
license: "CC BY 4.0"
language: "en-GB"
repository: "Double-FW/Creative-Disability-Casting-for-Film-and-TV-Drama"
artifact_type: "agent_policy_document"
use_case:
  - "talent_discovery"
  - "casting_evaluation"
  - "script_analysis"
  - "production_decision_recording"
entities:
  - name: "candidate"
    description: "A performer or talent entry including public credits and consenting access_summary"
  - name: "search_query"
    description: "Search parameters used to discover talent"
  - name: "decision_record"
    description: "Structured record of recommendations and final human decisions"
  - name: "access_summary"
    description: "Performer-provided, consented access adjustments (non-medical)"
  - name: "agent_run"
    description: "Metadata for each automated run: agent_version, data_sources_snapshot, commit_or_run_id"
review:
  last_reviewed: "2026-10-12"
  review_cycle_months: 6
  reviewer: "repository owner"
change_log_url: "./AGENTS.md#change-log"
related_repos:
  - "https://github.com/Double-FW/Authentic-Representation-Project"
  - "https://github.com/Double-FW/Drama-and-Disability"
---

## MACHINE-READABLE: rules, schemas, templates and flow

This section contains machine-readable artifacts an agent can ingest before using the human-readable guidance below.

Machine rules (JSON)
```json
{
  "rules_version": "1.0",
  "rules": [
    {
      "id": "R001",
      "name": "human_ownership_required",
      "trigger": "any_decision_recommendation",
      "action": "require_field",
      "field": "decision_owner",
      "escalate_to": "producer_or_casting_director",
      "severity": "blocker"
    },
    {
      "id": "R002",
      "name": "no_inference_of_medical_status",
      "trigger": "candidate_profile_build",
      "action": "forbid",
      "forbidden_inference": ["diagnosis","impairment_from_image","neurotype_from_bio"],
      "severity": "high"
    },
    {
      "id": "R003",
      "name": "preserve_audit",
      "trigger": "agent_run_complete",
      "action": "emit_audit_record",
      "audit_record_fields": ["agent_version","agent_query","data_sources_snapshot","top_results","timestamp","run_id"],
      "severity": "must"
    },
    {
      "id": "R004",
      "name": "escalate_on_non_disabled_for_disabled_role",
      "trigger": "recommendation_for_explicitly_disabled_role_with_non_disabled_candidate",
      "action": "escalate",
      "escalate_to": "named_human_decision_owner",
      "severity": "blocker"
    }
  ]
}
```

Decision record schema (JSON Schema)
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "DecisionRecord",
  "type": "object",
  "required": ["production","role","decision_owner","agent_query","longlist","final_decision","timestamp"],
  "properties": {
    "production": {"type": "string"},
    "role": {"type": "string"},
    "decision_owner": {"type": "string"},
    "agent_query": {"type": "object"},
    "longlist": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["candidate_id","evidence_links","verification_status"],
        "properties": {
          "candidate_id": {"type": "string"},
          "evidence_links": {"type": "array","items":{"type":"string","format":"uri"}},
          "skills_and_credits": {"type":"array","items":{"type":"string"}},
          "access_summary": {"type":"string"},
          "verification_status": {"type":"string","enum":["verified","unverified","verification_pending"]}
        }
      }
    },
    "shortlist": {"type":"array","items":{"type":"string"}},
    "essential_characteristics": {"type":"array","items":{"type":"string"}},
    "adaptable_characteristics": {"type":"array","items":{"type":"string"}},
    "consultation": {"type":"array","items":{"type":"object"}},
    "access_implications": {"type":"object"},
    "verification_steps": {"type":"array","items":{"type":"string"}},
    "final_decision": {"type":"string"},
    "final_rationale": {"type":"string"},
    "dissenting_advice": {"type":"string"},
    "review_date_or_trigger": {"type":"string","format":"date-time"},
    "timestamp": {"type":"string","format":"date-time"},
    "agent_run_id": {"type":"string"}
  }
}
```

Observability / Audit spec (YAML)
```yaml
observability:
  audit_record:
    required_fields:
      - agent_run_id
      - agent_version
      - agent_query
      - timestamp
      - data_sources_snapshot
      - top_results
      - human_overrides
    retention_days: 3650
    storage_class: "restricted"
  redact_rules:
    - name: "email"
      pattern: "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}"
      replacement: "[REDACTED_EMAIL]"
    - name: "phone"
      pattern: "\\+?\\d[\\d \\-()]{7,}\\d"
      replacement: "[REDACTED_PHONE]"
    - name: "national_id"
      pattern: "\\b\\d{8,}\\b"
      replacement: "[REDACTED_SENSITIVE]"
    - name: "medical_terms"
      pattern: "(?i)\\b(diagnosis|diagnosed|has\\s+\\w+)\\b"
      replacement: "[REDACTED_MEDICAL]"
  retention_policy_notes: "Audit records stored in restricted location. Decision records that include sensitive personal data must be pseudonymised or stored in operational records with lawful basis and consent."
```

Minimal decision flowgraph (JSON)
```json
{
  "flow_version": "1.0",
  "nodes": [
    {"id":"start","type":"start"},
    {"id":"search","type":"action","label":"Run inclusive search"},
    {"id":"longlist","type":"action","label":"Build longlist (blind mode available)"},
    {"id":"blind_review","type":"action","label":"Creative blind review"},
    {"id":"unmask_if_needed","type":"decision","label":"Require identity for authenticity/access?"},
    {"id":"verify","type":"action","label":"Verify availability/credits/access_summary"},
    {"id":"escalate","type":"action","label":"Escalate to human (per triggers)"},
    {"id":"decision","type":"action","label":"Human final decision"},
    {"id":"archive","type":"end","label":"Archive decision record & audit"}
  ],
  "edges": [
    ["start","search"],
    ["search","longlist"],
    ["longlist","blind_review"],
    ["blind_review","unmask_if_needed"],
    ["unmask_if_needed","verify"],
    ["unmask_if_needed","escalate"],
    ["verify","decision"],
    ["escalate","decision"],
    ["decision","archive"]
  ]
}
```

Templates (short prompts and expected responses)

- Prompt: talent-search (short)
  - "Search for candidates for role: {role}. Essential characteristics: {essential_characteristics}. Adaptable: {adaptable_characteristics}. Max results: {n}. Return JSON longlist with fields candidate_id, display_name(if consented), evidence_links, skills_and_credits, access_summary (consented only), verification_status."

- Expected agent response (JSON)
```json
{
  "agent_run_id": "uuid-v4",
  "timestamp": "2026-10-12T12:00:00Z",
  "query": { "role":"{role}","essential":["..."],"adaptable":["..."], "max_results": 50 },
  "longlist": [
    {
      "candidate_id": "profile-12345",
      "display_name": null,
      "evidence_links": ["https://..."],
      "skills_and_credits": ["credit1","credit2"],
      "access_summary": "requires accessible travel; consented",
      "verification_status": "verification_pending"
    }
  ]
}
```

- Prompt: verify-candidate
  - "Verify candidate {candidate_id} availability and representation. Check agency page or public contact, return verification_status and confirmation timestamp."

- Prompt: prepare-decision-record
  - "Produce DecisionRecord JSON conforming to DecisionRecord schema for production {production} role {role}. Include agent_run_id and timestamp."

Quick-start checklist (compact)
```yaml
quick_start:
  - "Load canonical Guidelines and AGENTS front matter (check review metadata)."
  - "Prepare role brief (essential/adaptable characteristics)."
  - "Run inclusive search query (record agent_query & agent_run_id)."
  - "Produce blind longlist; human does creative blind review."
  - "Unmask finalists only when needed for authenticity/access."
  - "Verify with authorised contact; record verification_steps."
  - "Create DecisionRecord and assign decision_owner."
  - "Archive DecisionRecord and audit_record with retention metadata."
```

Observability & escalation logging requirements (short)

- Emit an audit record at the end of every run containing the required_fields from the audit_spec above.
- Store audit records in access-limited storage, tagged with retention_days.
- Apply redact_rules before any record is stored in non-restricted locations (e.g., public PRs).
- For any escalation, generate an escalation_event with: run_id, trigger_reason (rule id), implicated_candidate_id, suggested_actions, and assigned_decision_owner.
- All human overrides must be stored in the audit record with user_id and timestamp.

Machine-checkable changelog (YAML)
```yaml
changelog:
  - date: "2026-10-12"
    version: "1.1"
    author: "Double FW"
    notes: "Add machine-readable metadata, schemas, templates, observability hooks and flowgraph."
```

---

# AGENTS

Purpose
-------
This document defines operating principles and practical rules for automated agents and AI-assisted workflows used with this repository. The focus is on: evaluation of candidate and evidence material; searching for talent and production information online; ensuring proportional and meaningful representation of disabled people; and removing, detecting and mitigating bias from discovery, shortlisting and decision processes.

Principles (non-negotiable)
---------------------------
- Human accountability: an identifiable human (producer, casting director or named decision owner) must own every casting or portrayal decision an agent supports. An agent provides recommendations, not approvals.
- Do no harm: never take actions that could expose, out, or endanger a person (performer, consultant or source). Treat disability, health and access information as sensitive.
- Evidence hygiene: label all claims with evidence type (documented fact, reasoned interpretation, practitioner approach, discovery lead, verification pending). Never promote unverified retrospective diagnoses as fact.
- Authenticity and proportionality: prioritise relevant lived experience when disability, language or culture materially affects character embodiment, communication or meaning. For incidental roles, prioritise open casting and widened candidate pools.
- Minimal inference: do not infer a person's diagnosis, impairment, neurotype, or access needs from images, video, social media, performance, assistive technology or biography.
- Auditability and traceability: record agent inputs, search queries, results, verification steps, and the human decisions that follow.

Scope and where agents may be used
----------------------------------
- Talent discovery (research and longlists)
- Script analysis for casting-relevant characteristics
- Evidence collation and production decision record preparation
- Monitoring public discourse, policy and production trends

Authorised external resources
-----------------------------
Agents may query and retrieve information from the following source types when relevant and within legal and policy constraints:
- Performer directories (e.g., ProFile)
- Public professional profiles (actor websites, Spotlight, IMDb) and public press coverage
- Institutional guidance (Broadcaster commitments, union resources such as Equity and SAG-AFTRA)
- Repositories maintained by Double-FW: 
  - https://github.com/Double-FW/Authentic-Representation-Project — use for productions, talent, policies, trends and creative-risk context
  - https://github.com/Double-FW/Drama-and-Disability — Disability & Drama repository and future articles (Disability and Drama Repository)
- Academic, historical and published sources for background research

Search and talent-discovery rules
---------------------------------
- Use inclusive, broad queries first. Example intent: "actors who use wheelchairs with screen credits" rather than binary yes/no filters.
- Prefer verified, professional sources for availability and credit information (agent pages, agency contact, Spotlight). Treat social posts as discovery leads requiring verification.
- Preserve contact, consent and privacy expectations: do not scrape private contact details or direct-message people without an established outreach protocol and human approval.
- Record query terms, date/time, and top-N sources returned for every automated search.
- Provide an accessible, human-reviewable longlist with evidence links, skills/credits summary, and an `access_summary` field describing publicly-known, performer-provided adjustments (never include medical or other special-category personal data recorded without lawful basis).

Evaluation, scoring and proportional representation
-------------------------------------------------
- Evaluation outputs must separate evidence from recommendation. For each candidate provide:
  - `evidence_links` (URLs and evidence label)
  - `skills_and_credits` (documented credits, training, specialist skills)
  - `relevance_rationale` (why the candidate may fit essential/adaptable characteristics)
  - `verification_status` (verified/unverified/verification_pending)
  - `access_summary` (performer-provided, consented adjustments only)
- Proportional representation metrics (agent compute only; used to highlight gaps, not to select individuals): track baseline diversity markers at a population or production-level (e.g., proportion of roles with incidental disability representation, types of disabilities represented) using aggregated, non-identifying statistics.
- Never use a single person as a proxy for an entire community. Report counts and ranges rather than treating single occurrences as representative.

Bias detection and mitigation
-----------------------------
- Input sanitisation: remove or annotate search filters that embed ableist assumptions ("severe", "mild", "functional limitation cost") and require human confirmation before using such language.
- Blind-review modes: produce candidate lists where personally-identifying details are hidden for early-stage creative evaluation (showreel/skills first), exposing identity only when essential for authenticity or access.
- Counterfactual sampling: when a candidate pool is narrow, the agent should automatically generate alternative searches (different keywords, broadened geographic or training sources) and report outreach opportunities tried.
- Explainability: provide short rationales for every ranking or score. Include which features and sources contributed to a score.
- Audits: keep an immutable audit log of search inputs, candidate outputs, ranking criteria and human overrides for periodic review.

Data protection, privacy and sensitive data
-------------------------------------------
- Treat disability status, medical details and access needs as sensitive personal data. Store these only in restricted operational records when lawfully permitted and with explicit performer consent.
- Do not persist or display special-category personal data in public or widely-shared artifacts (issues, public PRs, README files, open task logs).
- Use `access_summary` fields with consented and minimal language (for example: "requires accessible travel and 1:1 sign-language support at rehearsal") rather than medical labels.
- Adhere to applicable jurisdictional law (e.g., UK Equality Act, ICO guidance) and the production's privacy policies.

Human-in-the-loop and review triggers
-------------------------------------
Agents must escalate to named humans for review when any of the following occur:
- A non-disabled performer is recommended for an explicitly disabled role.
- The workflow proposes condition transposition or simulation of disability through prosthetics, movement imitation or VFX.
- Historical diagnosis is central to casting and evidence is disputed or retrospective.
- An adjustment is claimed to be impracticable or materially costly.
- Publicity or marketing would disclose a performer's disability without explicit consent.

Decision records and traceability
---------------------------------
For every casting-related recommendation include a decision record (template fields):
- `production` and `role`
- `decision_owner` (named human)
- `agent_query` (exact search terms and date/time)
- `longlist` (IDs + evidence links)
- `shortlist` (IDs + rationale)
- `essential_characteristics` and `adaptable_characteristics`
- `consultation` (who was consulted and remit)
- `access_implications` and `access_owner`
- `verification_status` and `verification_steps`
- `final_decision` and `rationale`
- `dissenting_advice` or `unresolved_uncertainty`
- `review_date_or_trigger`

Verification and responsible sourcing
-----------------------------------
- Do not treat directory inclusion or IMDb lists as proof of availability, representativeness or consent. Verify claims with agent/agency contact, or an authorised public channel.
- Verify historical claims against scholarly sources before adopting a portrayal decision that depends on them.
- When scraping public data, follow the target site's robots and terms of service; prefer API access and rate-limiting.

Maintenance, updates and provenance
----------------------------------
- Tag each agent-run output with `agent_version`, `data_sources_snapshot` (list of URLs + retrieval dates), and `commit_or_run_id` so results can be traced.
- When updating this file, list change notes with date, author and rationale.

Operational examples (short)
---------------------------
1) Talent longlist workflow
   - Input: approved breakdown, essential/adaptable characteristics
   - Agent: run inclusive searches against performer directories and public credits, build longlist with evidence links and `access_summary` (consented)
   - Human: casting lead reviews longlist in blind-review mode, then requests identity-unmasked details for finalists

2) Script-character verification
   - Input: script excerpt and character brief
   - Agent: extract functional and embodied characteristics and cross-reference historical/technical sources
   - Human: creative disability consultant reviews, signs off on essential vs adaptable list

References and related repositories
-----------------------------------
- Creative Disability Casting reference & guidelines in this repository (canonical): Creative Disability Casting Guidelines and Creative_Disability_Casting_Reference.md
- Related Double-FW projects:
  - https://github.com/Double-FW/Authentic-Representation-Project — productions, talent, policies, trends and creative-risk information
  - https://github.com/Double-FW/Drama-and-Disability — Disability & Drama Repository and future articles an agent will find useful
- Legal and guidance starting points: Equality Act guidance, ICO special-category data guidance, BBC Five As and Channel 4 Code of Portrayal (see Reference section in Creative_Disability_Casting_Reference.md)

Contact & escalation
--------------------
For questions about this agent policy or to request an exception, contact the repository owner or the named producer for the production. All exceptions must be recorded with rationale and an appointed decision owner.

---

Change log
----------
- 2026-10-12: Version 1.0 created by repository contributor guidance.
- 2026-10-12: Version 1.1 added machine-readable metadata, schemas, templates, observability hooks and flowgraph (this commit).
