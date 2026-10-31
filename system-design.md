# MMAUG Insurance Claims Agent System Design and Architecture

> Version 1.1 | 23 October 2026

**MMAUG Insurance Claims Agent**

**System Design and Architecture**

Meeting revision  |  Version 1.1  |  23 October 2026

Updated design direction and Agile delivery plan

### Design intent

This design translates BRD v1.1 into an Azure-first implementation proposal after the 22 October 2026 review and supplied clarifications. It retains FR-01 to FR-15 and NFR-01 to NFR-10. Scope is internal motor accidental-damage preparation. Reviewer-requested agent and retrieval changes are incorporated as a candidate design; implementation depth, budget and outstanding choices require recorded decisions.

The system prepares evidence for human claims officers. It cannot approve or reject claims, determine final liability, authorise payments or modify policies. These constraints are enforced by available interfaces and service permissions, not merely instructions to the model.

### Recommended architecture

Use an officer web application, authenticated backend, bounded coordinator and specialist roles, private evidence storage, PostgreSQL and a synthetic read-only policy API. The candidate retrieval path indexes approved knowledge from Blob Storage into Azure AI Search. Deterministic validation, evidence references, explicit unavailable states and human review remain mandatory.

### Approval boundary

The meeting reviewed direction and advised backlog preparation. Record the customer disposition and conditions; no blanket sign-off, provisioned cloud resource or completed testing is asserted. Prepare the backlog now; resolve dependencies before affected implementation and approve budget before provisioning.

### Document map

Sections 1-8 retain topology, workflow, technology, data, AI, security, operations and verification. Sections 9-12 add agent/retrieval contracts, identity and interface detail, Agile implementation and the decision/cost register.

## 1  Architecture context and topology

Primary user: authenticated internal claims officer handling motor accidental damage. Administrator is the minimum supporting role; supervisor/auditor is optional with justification. No claimant portal is included. Figure 1 shows the candidate deployment and controlled data flow; optional production gateways/failover are not assumed MVP scope.

![Candidate Azure service topology and evidence flow](../assets/architecture-v1-1.png)

Figure 1 Candidate Azure service topology and numbered evidence flow

### Trust boundaries

The browser is untrusted: the API rechecks identity and object access for every case operation. Uploaded documents cross a quarantine/parser boundary before processing. The model receives only necessary evidence and has no direct database or storage credentials. The tool gateway checks identity and case scope outside the model before it calls the synthetic API. Evidence references return through the authorised application rather than public file URLs.

### Deployment shape

Propose frontend/API/worker containers on Azure Container Apps, private Blob containers and Azure Database for PostgreSQL. Persist run creation before Storage Queue dispatch. The bounded worker coordinates specialist roles and versioned results. A separate approved-knowledge indexing path feeds Azure AI Search; case evidence remains scoped by case. Confirm queue retry/dead-letter settings, search scope, regional availability and cost before provisioning.

## 2  Processing flow and failure behaviour

| Stage | System responsibility | Control or exception |
| --- | --- | --- |
| Intake | Authenticate, create case and associate uploaded files. | Check claim-level authorisation; stable IDs and validated metadata. |
| Clearance | Validate type/signature/size, quarantine and scan; parse in isolation. | Rejected or unreadable files remain visible as failed inputs; no unsafe parsing or AI access. |
| Extract and check | Capture fields and source positions; run mandatory-field and contradiction rules. | Preserve raw values, competing values and uncertainty; never silently resolve material conflicts. |
| Controlled lookup | Retrieve authorised policy, coverage, history and business rules. | Gateway scopes arguments; bounded calls/timeouts; no-match differs from denied/unavailable. |
| Synthesis | Compose structured draft from validated evidence and lookup results. | Verify schema and supporting citations; suppress or mark unsupported assertions. |
| Review | Officer opens evidence and records preparation review. | Preserve machine output and human annotations separately; no automated consequential action. |

### Proposed run lifecycle

Case: Created → Evidence received → Review required → Preparation reviewed. ProcessingRun: Queued → Running → Completed, Partial or Failed. Document: Quarantined → Cleared or Rejected; parsing status is tracked separately. These proposed states refine BRD section 10 without creating an AI approval/rejection state. Exact permitted transitions require agreement.

### Retries and partial results

Use idempotent job/run identifiers and bounded retry with backoff for transient service faults. Do not retry authorisation denial as if it were a timeout. Persist terminal failures for support inspection; give the officer a safe retry or document replacement action. Mark unavailable sections explicitly so a partial summary cannot masquerade as complete. Retry counts, delay and expiry are open design parameters.

Reprocessing creates a new immutable run and summary version. The active version is explicit; earlier reviews remain tied to the version reviewed. Concurrent runs should not overwrite each other. Cancellation, stale-job recovery and maximum run duration are to be agreed before implementation.

## 3  Components and technology choices

| Capability | Proposed selection | Rationale and alternative |
| --- | --- | --- |
| Web experience | React with TypeScript | Supports evidence/review interface. Simpler server-rendered UI if team capacity favours it. |
| API and worker | Python/FastAPI, Container Apps | Fits document/AI libraries and isolated background work. App Service plus a suitable worker is an alternative. |
| Evidence and jobs | Private Blob Storage; Storage Queue | Separates files and dispatch from relational metadata. Queue abstraction allows an alternative such as Service Bus if justified. |
| Transactional records | Azure Database for PostgreSQL | Relations suit cases, source references and versions. Azure SQL is viable with stronger team familiarity. |
| Identity and secrets | Entra ID; managed identities; Key Vault | Least-privilege service access without embedded secrets. Confirm tenant and service compatibility. |
| Parsing and OCR | Native text parsing, Document Intelligence selectively | OCR only for agreed scans; evaluate fixture quality. Native extraction suffices for text-only scope. |
| AI orchestration | Azure-hosted model candidate; bounded coordinator and specialist roles | Separate model deployment from agent responsibilities; typed handoffs and tool permissions. Record MVP multi-agent implementation decision. |
| Delivery and monitoring | GitHub Actions, Bicep, Azure Monitor/Application Insights | Azure-aligned repeatability and correlated traces. Terraform is a portable IaC alternative. |
| Knowledge retrieval | Blob -> indexer/embeddings -> Azure AI Search | Reviewer-requested candidate path. Index approved knowledge; preserve source/version metadata and access isolation. Validate cost and need. |

### Selection conditions

Confirm subscription access, tenant, region, model availability/quota, scanner and budget. Sweden Central was suggested in the meeting; validate actual subscription access and suitability before selecting it. Benchmark available models for structured output, grounding, latency and spend. Spoken model-retirement/pricing examples are not a deployment specification.

### Scope and complexity control

Respond to the reviewer request with explicit coordinator, extraction/validation and evidence-summary responsibilities and a Blob-to-search path. Confirm whether these roles must be independently implemented in the MVP. Keep bounded handoffs and justify each service. Application Gateway, API Management and multi-region replication discussed as production examples require separate value/cost decisions.

## 4  Data design and interface boundaries

| Concept | Proposed relationship and evidence handling |
| --- | --- |
| Claim and Document | Claim owns document metadata; immutable storage version, hash, MIME, size and clearance state identify each source. |
| ProcessingRun and ExtractedField | Run records versions/status; fields retain raw/normalised values, extraction method, uncertainty and evidence locations. |
| EvidenceReference and Finding | Reference locates document page/span or captured API result. Finding records missing/conflicting data and all relevant sources. |
| ToolInvocation and lookup snapshot | Record tool/version, scoped safe arguments, result/time and correlation ID. Keep authorised snapshot provenance for reproducible summaries. |
| SummaryVersion and ReviewAction | Version links run and evidence. Review links exact version, actor, time and optional notes; never erases original AI output. |
| AuditEvent | Records actor/service, case/run, action and outcome with minimised sensitive content and restricted access. |

### Conceptual application interfaces

Intake interfaces create cases and accept files into quarantine. Read interfaces expose authorised case detail, evidence and run progress. Run interfaces start/retry bounded processing; review interfaces record the officer action against a specific summary version. These are interface responsibilities, not final endpoint paths or schemas. No application or tool endpoint performs adjudication, payment or policy amendment.

### Synthetic policy API contract

Authenticated read-only synthetic lookup accepts validated policy/case identifiers; caller scope comes from backend identity, never model arguments. Return policy, coverage, history, rule IDs/versions, response ID/time and provenance. Outcomes: success, no_match, denied, unavailable and invalid_request. Exact fixture rules/history depth remains D04/Q15. These facts never determine final coverage eligibility or liability.

### Consistency and retention

Persist run creation before dispatch; use idempotency keys and transactions to prevent duplicate or orphaned state. Reconcile dispatch failures safely. Deletion must account for originals, parsed derivatives, summary versions, API snapshots, logs and backups. Retention duration, evidence access and deletion timing remain approval items. Store synthetic data only; real-data governance is outside this MVP.

## 5  AI workflow and Responsible AI controls

### Deterministic steps before generation

Parse usable text, validate field schemas, normalise dates/identifiers and apply versioned mandatory-field/conflict rules. Preserve raw evidence. An LLM may extract facts from unstructured narratives and draft readable summaries; it must not invent absent values or resolve contradictory evidence without review. Do not equate model-reported confidence with calibrated probability.

### Bounded orchestration

The coordinator controls a fixed workflow through typed specialist handoffs, allowlisted tools, finite step/token/time budgets and bounded retries. Extraction/validation and evidence-summary roles cannot execute arbitrary shell, browse, query unrestricted databases or access other cases. Model/tool/document content remains untrusted; backend checks scope and validates outputs. MCP is an optional tool transport, not an authorisation control.

### Proposed summary contract

Sections: case overview; extracted facts; policy/history/rule lookup results; missing/conflicting facts; uncertainties; follow-up recommendations; citations; processing limitations. Label interpretation separately from facts. Never invent missing facts, coverage decisions or liability. Officer acknowledgement is required; notes/corrections/request-information are optional if capacity permits.

### Grounding and injection defence

Validate that every material factual assertion is supported by its cited source, not merely attached to a citation identifier. Preserve both sides of conflicts. Use instruction/evidence separation, scope enforcement, output validation and adversarial fixtures. Optional Prompt Shields can add document-attack detection; it is probabilistic defence in depth and does not replace authorisation or isolation.

### Evaluation approach

Proposed targets: >90% extraction correctness, 90% omission recall, 85% conflict recall and 60-second latency. Agree normalisation/matching, applicable denominators, timing statistic and start/end points before acceptance. Use labelled synthetic cases and report false positives, groundedness, honest failure and forbidden-tool outcomes. Keep holdout variants and record model/prompt/schema/rule versions. Results do not establish production generalisation.

### Human control

Officers inspect original sources and record review against the exact draft. Recommendations are preparation observations, not claim outcomes. If evidence cannot support a statement, remove it or explicitly label it unverified. No subsequent action is automatically triggered by review acknowledgement.

## 6  Identity security and privacy design

| Boundary or threat | Proposed control | Verification intent |
| --- | --- | --- |
| Unauthorised case access | Entra sign-in; server-side role and object checks on every case/file/summary request. | Cross-officer access and forged role/claim identifiers fail. |
| Malicious uploads | Allowlisted types, signature/size checks, quarantine/scan, resource-limited parser and private storage. | Spoofed/oversized files rejected; unscanned files unavailable to AI. |
| Document prompt injection | Separate evidence from instructions; gateway allowlist and scope checks outside model. | Embedded commands cannot reveal secrets, access other cases or execute tools outside scope. |
| Credential or data exposure | Managed identities where supported, scoped secrets in Key Vault, encrypted transport/storage, minimised logs. | Secret scans, role review, private file checks and redacted telemetry. |
| Unsupported conclusions | Evidence validation, uncertainty labels and human review; no consequential-action interfaces. | Unsupported coverage/liability statements flagged; forbidden-action tests fail closed. |
| Evidence/history alteration | Immutable document versions, versioned summaries/reviews and append-oriented events. | Citations resolve after reprocessing; reviewer edits preserve originals. |

### Service identities and network access

Separate application, worker and deployment identities where practical. Give the worker only the storage/model/API permissions it needs; the model receives no infrastructure credentials. Use private storage and authenticated backend file access. Private endpoints and network isolation are proposed where service tier and budget permit; confirm detailed ingress/egress design before provisioning. Prefer managed identities for supported service-to-service access.

### Privacy and retention decisions

Use approximately twenty synthetic customers/policies and synthetic claim documents only. Confirm region, AI provider data handling, audit access, retention, deletion and backup lifecycle before any real personal data. Do not log full documents, tokens or credentials. Encryption alone does not establish GDPR compliance. Production readiness would require separate governance, security review and support approval.

### Scanner contingency

Apply proposed PDF/JPEG/PNG rules, 10 MB/file and five attachments/claim only after agreement. Validate signature/type/size/count; quarantine and scan before parsing or AI access. If scanning is unavailable, restrict the demo to vetted fixtures with recorded acceptance. Confirm full clipped Q14 response and supported OCR formats; visual damage interpretation is excluded.

## 7  Delivery deployment and operations

### Environment and release topology

Use local/isolated development, separate test/UAT resources and a controlled demo or production-like environment with distinct identities, configuration and secrets. Propose Bicep-managed infrastructure and immutable container images. A real production environment is deferred pending service levels, capacity, governance and support. Confirm that the production-like demo satisfies bootcamp expectations.

### Proposed CI and CD flow

Reviewed pull request -> static/unit/integration checks -> agent fixture evaluation -> dependency/secret/container/IaC checks -> immutable build in Azure Container Registry -> test deployment -> UAT evidence -> authorised demo release -> post-deployment verification. Protect main; agree security thresholds and release exception authority.

Use one repository with frontend, backend, agents, data, infra, tests and docs folders; one ordered project backlog and short-lived feature branches. Version prompts, tool schemas, rules and model configuration with application code. Prefer federated deployment identity over long-lived keys. Retain release manifest and evidence. Owners/reviewers and estimates are assigned in refinement.

### Observability and support

Emit safe structured logs with case/run correlation and step outcomes. Track queue age, processing failures, API denials/timeouts, extraction errors, grounding failures, latency, token use and estimated spend. Show operational status without exposing raw claim content. Define alerts, support owner and escalation route before UAT; thresholds and budget alerts need agreement.

### Recovery and rollback

Persist job status, identify abandoned runs and provide safe retries. Verify sign-in, upload, processing, controlled lookup, citations and review after release. If verification fails, stop promotion and roll back image and compatible prompt/rule/schema versions. Prefer backward-compatible database changes; destructive changes need tested recovery. A separate restore test covers case data and evidence; image rollback alone cannot recover deleted files.

### Operational handover

Deliver environment/deployment instructions, access model, configuration inventory, model/prompt/rule versions, synthetic fixture catalogue, test/UAT report, known limitations and runbooks for failed jobs, unavailable services, incidents and rollback. Recovery objectives, operating hours and production support commitments remain open; do not imply a 24-hour support service.

## 8  Verification decisions and design approval

| BRD requirement group | Design coverage | Acceptance evidence to collect |
| --- | --- | --- |
| FR-01–03; NFR-01–03 | Sections 1–4 and 6: intake, access, files, records | Authenticated journey and access/file negatives. |
| FR-04–06; NFR-05–07 | Sections 2/4/5: extraction, checks, sources | Labelled fields/omissions/conflicts; timings; keyboard review. |
| FR-07–08/12; NFR-04 | Sections 2–4: scoped lookup and resilience | Controlled API call, denied/no-match/timeout and safe retry. |
| FR-09–11; NFR-06/08/09 | Sections 4–6: cited summary, review, audit | Source support, versioned review, injection and authority tests. |
| NFR-08–10 | Section 7: CI/CD, IaC, monitoring, recovery | Passing pipeline, release approval, smoke test, alerts and rollback. |
| FR-13–15 | Case navigation, optional annotations/export | Implement Should/Could scope only if approved and capacity allows. |

### Verification of the meeting changes

Intake: demonstrate an internal officer creating a motor accidental-damage case with the agreed minimum fields. Include an intentionally incomplete evidence set; missing analysis fields remain visible and are not invented. Reject theft/third-party workflow selection and unauthorised case access.

Files and sources: demonstrate agreed type, size and attachment-count boundaries, spoofed content, quarantine and cleared-only parsing. Every fact resolves to an immutable document page/span or captured lookup response. Confirm full Q14 OCR wording before finalising format coverage.

Agent and retrieval path: trace a run through each selected role and typed handoff. Test exhausted budgets, malformed outputs, specialist failure and attempts to access another case. If search is selected, verify source/version metadata, relevant retrieval, no-match behaviour and knowledge-only ingestion.

Human control: show success, no-match, denied and unavailable lookup outcomes; preserve both sides of material contradictions. Acknowledge an exact summary version and confirm that this does not trigger claim approval, rejection, policy change or payment.

Delivery: retain labelled fixture results and the measured environment, model/prompt/schema/rule versions. Demonstrate reviewed CI, immutable image, authorised deployment, post-release smoke test, monitoring and rollback. Record failed checks and known limitations before acceptance.

Evidence owners: QA coordinates the acceptance matrix; component owners supply results; the product owner/customer proxy records the review outcome. Open decisions and cost assumptions are tracked in section 12 and BRD section 26.

## 9 Agent roles and retrieval contracts

The reviewer requested multiple agents around a coordinator. The following roles are a bounded candidate design. D03 records whether distinct hosted agents are mandatory for the MVP or whether explicit modules with the same contracts satisfy the review. They share a single case/run context and cannot expand their own authority.

| Role | Input and output | Permitted operations |
| --- | --- | --- |
| Coordinator | Case/run IDs, authorised context and step status -> next typed step or explicit failure | Dispatch allowlisted steps; maintain run state and budgets; no final claim decisions. |
| Extraction and validation | Cleared scoped sources -> fields, citations, uncertainty, missing/conflict findings | Read case evidence; invoke extraction/OCR as allowed; deterministic checks preserve both conflict values. |
| Evidence summary | Validated facts/findings + scoped lookup results -> versioned cited draft | Read-only policy/history/rule tool and scoped knowledge retrieval; schema/source checks before persistence. |

Handoff envelope: schema_version, case_id, run_id, step_id, source_ids, result_status, output, evidence_refs, limitations and error_code. Validate case/run binding and schema outside the model. Specialist failure yields partial or failed status, never invented facts. Configure finite calls, retries, tokens and deadlines in a versioned ADR; enforce them at runtime.

Knowledge path: approved synthetic policy/rule documents -> dedicated Blob knowledge container -> text extraction/chunks with immutable source/version/page metadata -> embedding deployment -> Azure AI Search index -> scoped retrieval tool -> evidence-summary role. Exact chunking, embedding model, index schema and freshness strategy are D04 decisions. Test relevant/irrelevant and missing-source cases before selecting settings.

Claim uploads follow quarantine -> clearance -> case-bound extraction. They are not automatically added to a shared knowledge index. If case evidence indexing is later selected, enforce case scope on ingestion and every query in the backend, test cross-case negatives and include index deletion in retention. Search results supplement evidence; authoritative policy/history facts remain in the read-only synthetic API.

## 10 Identity interfaces and data validation

| Caller or boundary | Access and enforcement | Verification |
| --- | --- | --- |
| Officer -> API | Validate sign-in token/session, role and case assignment on every route. Client-supplied scope is untrusted. | No session and forged/cross-case IDs fail. |
| API identity | Scoped case DB operations, quarantine upload and permitted evidence access; secrets only where needed. | No public Blob access; direct ID access remains authorised. |
| Worker identity | Read cleared case sources; call selected model and read-only tools; persist scoped results through controlled persistence. | Cannot parse quarantined files or mutate policy/payment records. |
| Indexer identity | Read approved knowledge container; write selected search index through scoped deployment/runtime identities. | Cannot ingest quarantine/case evidence implicitly; permission review. |
| CI identity | Federated deploy access to the approved environment; release privileges separated from runtime. | No long-lived credential in repo/log; protected release approval. |

Private storage means no anonymous access. Define permitted ingress/egress, firewall rules and any private endpoints with the selected tier and hosting topology. Public HTTPS ingress may be needed for the officer UI; downstream access remains constrained. Gateway/WAF, API Management and multi-region failover are optional costed extensions, not implied by Figure 1. Verify exact cloud role names/actions for the chosen SDK operations before assigning permissions.

| Proposed interface | Behaviour and failure contract |
| --- | --- |
| Create/read case | Stable case ID; authenticated internal officer; enforce assignment; validate agreed intake fields. |
| Upload/read evidence | Bind immutable source ID to case; enforce agreed PDF/JPEG/PNG and proposed 10 MB/five-file limits; reject/quarantine invalid uploads. |
| Start/read run | Idempotent start; durable queued/running/completed/partial/failed state; safe repeat requests; explicit error and retry guidance. |
| Read-only lookup | Case/policy identifiers + backend caller context; success/no_match/denied/unavailable/invalid_request; versioned provenance. |
| Read summary/acknowledge | Authorised access; citations resolve to permitted sources; acknowledgement records exact version, actor and time. |

Proposed evidence fields: policy number, insured/claimant name, incident date, reported date, vehicle registration, incident description and supporting source; claim ID is system generated. Preserve raw and normalised values and provenance. D02 distinguishes fields required to create a case from absent evidence fields that must be flagged. Do not reject an intentionally incomplete claim merely because analysis must detect its omissions.

## 11 Agile implementation and verification

Next step: a 90-minute scope/decision/backlog refinement workshop, suggested within 2-3 working days, followed by Sprint Planning. Nominate product owner/proxy, use actual capacity and bootcamp dates, and record implementation owners/reviewers. This preparation timebox is not a mandatory Sprint 0. Refine design and test plans throughout delivery.

Sprint 1 Goal: an authenticated internal officer creates a motor accident case, uploads a permitted synthetic document, reopens it securely and sees processing status, while unauthorised access is blocked. Candidate scope is EN-02 and US-01 to US-03 from BRD section 25; settle D01/D02/D05 dependencies first. Add one-format extraction only if capacity supports it.

| Workstream | First increment and subsequent evidence |
| --- | --- |
| Frontend | Sign-in, case creation, upload/rejection/status and reopen; later evidence viewer, cited summary and exact-version acknowledgement. |
| Backend and data | Identity/case checks, IDs, schema, private uploads and durable run state; later synthetic API, provenance, summary/review persistence. |
| AI and retrieval | Agree contracts/fixtures and bounded agent design; then extraction/checks, controlled lookup, approved knowledge retrieval and grounded summary. |
| Platform and security | Monorepo/CI, isolated config and safe access from Sprint 1; then Bicep/registry, deployments, telemetry, cost alerts and rollback. |
| QA | Intake/access/file negatives immediately; then labelled extraction/omission/conflict, API failures, injection, groundedness and UAT scenarios. |

Quality gates apply within each story: reviewed integrated change, acceptance/security checks, passing CI, demonstrable behaviour and documentation. AI work adds fixture and source-grounding evaluation. Review the integrated increment with stakeholders each sprint, adapt the ordered backlog and hold a retrospective. Board: Backlog -> Ready -> In progress -> Review/test -> Done, with blocked flag.

Evaluation design: approximately 20 synthetic customers/policies and a proposed 12-claim suite (four complete, four incomplete, four conflicting), plus denied/unavailable lookup, unsupported file, injection and cross-case access variants. Label expected fields/omissions/conflicts/source support before tuning. Extraction correctness divides correct applicable fields by applicable labelled fields; omission/conflict recall divides detected seeded issues by all seeded issues. Also report false positives. Agree treatment of nulls, multiple values and normalisation before scoring.

For the proposed 60-second metric, agree start/end boundary, statistic, concurrency, warm/cold behaviour and fixture limits. Record all run timings and environment/model versions. Do not convert a proposed benchmark into a contractual SLA. Deployment evidence includes authentication, upload, extraction, lookup, citations, review, monitoring, failure recovery and tested rollback.

## 12 Decisions costs and review record

| Decision | Owner role | Required output |
| --- | --- | --- |
| D01 Scope and authority | Product owner / customer proxy | Record review disposition and conditions; nominate acceptance authority and case assignment rule. |
| D02 Fields and files | Claims representative + backend | Agree intake/evidence distinction, field dictionary, Q04 limits and full Q14 OCR wording. |
| D03 Backend and orchestration | Technical lead + AI lead | Confirm Python/FastAPI recommendation; reviewer requested multiple agents; agree MVP implementation versus target design. |
| D04 Retrieval and API | AI/data + backend | Confirm Blob-to-search scope, isolation and API schema; settle Q15 rule/history depth. |
| D05 Environment and budget | Platform + budget authority | Confirm access, region, model quota, scanner, cost cap and deployment bill of materials. |
| D06 Evaluation and retention | QA + product owner | Agree measurement methods, target acceptance, retention duration and deletion scope. |
| D07 Delivery capacity | Project manager + team | Name owners/reviewers, available capacity and bootcamp milestone dates; order first-sprint work. |

All named assignments and due dates are set during refinement. Record proposed/accepted/deferred status, authority, date, rationale, requirement links and conditions. The supplied clarification screenshot confirms scope but leaves proposed limits/targets, retention and Q15 depth open; confirm complete Q14 wording.

| Cost group | Estimate inputs |
| --- | --- |
| Compute and data | Region, frontend/API/worker capacity, active hours, DB tier/storage/backups, Blob volume/transactions and queue usage. |
| AI and retrieval | Model deployment, input/output tokens, embedding volume, OCR pages, search tier/index size and indexer frequency. |
| Delivery and operations | Registry, monitoring ingestion/retention, secrets, network/egress and optional gateway/private endpoints. |
| Commercial record | Currency/date, calculator evidence, monthly estimate, demo-duration estimate, assumptions, approved cap, owner and alerts. |

No price total is committed in this revision. Platform owner completes a current calculator bill of materials and checks subscription/model access before provisioning. If proposed retrieval or network controls exceed the cap, record an agreed reduction with acceptance impact rather than silently removing a requirement.

Review disposition: approve, approve with recorded conditions, or revise. Customer/bootcamp delegate, product owner and technical/security lead record names, dates and conditions; implementation and release evidence are collected separately. Revision 1.1 updates the service diagram, reviewer-requested agent/search direction, fields/file rules, review contract, metrics, monorepo, Agile plan and cost/decision register.

Source record: 22 October 2026 review call, supplied clarification screenshot and supplied BRD/System Design PDFs. Companion: BRD v1.1 sections 20 and 24-26. Scrum reference: https://scrumguides.org/scrum-guide.html. Cloud product availability, exact permissions and prices must be verified during the affected technical decision.

Technical references checked 23 October 2026: Blob indexing https://learn.microsoft.com/en-us/azure/search/search-how-to-index-azure-blob-storage and Container Apps identities https://learn.microsoft.com/en-us/azure/container-apps/managed-identity. Blob indexing imports content and metadata; managed identities authenticate supported service access with scoped permissions. These references support the proposed paths, not untested configuration claims.
