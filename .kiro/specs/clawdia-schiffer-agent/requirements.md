# Requirements Document

## Introduction

Clawdia Schiffer is a policy-governed activist AI agent that campaigns against bottom trawling. The system demonstrates authorization embedded inside the agent loop — not bolted on afterward. Clawdia researches facts from approved sources, generates campaign media, drafts captions, and publishes to Instagram, while Cedar policies continuously constrain what it can research, what claims it can make, what tools it can use, what subagents it can spawn, and whether it can publish.

The core insight: Clawdia is safe not because of prompting, but because authority is bounded at every layer — browser, agent loop, subagents, tool calls, APIs, and publishing.

## Glossary

- **MainAgent**: The primary orchestrating agent (Clawdia) that plans, delegates, and assembles campaign packages
- **ResearchSubAgent**: A bounded subagent permitted only to search, fetch, and summarize from approved domains
- **MediaSubAgent**: A bounded subagent permitted only to generate images, video, and captions
- **PDP**: Policy Decision Point — the Cedar policy engine that evaluates authorization requests; in this system, implemented as an embedded Cedarling instance within each component
- **Cedarling**: An embeddable Cedar policy engine that runs in-process (as a native library or WASM module). Each component (MainAgent, ResearchSubAgent, MediaSubAgent, PublishGate) embeds its own Cedarling instance. The browser UI also loads Cedarling as a WASM module for real-time policy visibility.
- **DelegationRecord**: An explicit, scoped, time-bounded authorization grant from a parent agent to a subagent
- **CampaignPackage**: The assembled output containing claims, citations, media, and a confidence score
- **PublishGate**: The policy-controlled component that decides whether a campaign is published as draft or live
- **TPE**: Typed Partial Evaluation — Cedar's mechanism for querying applicable constraints before acting
- **ConfidenceScore**: A numeric value (0.0–1.0) representing the verified confidence in campaign claims
- **ApprovedDomain**: A domain explicitly listed in policy as a permitted research source
- **ClaimCitation**: A source reference attached to a factual claim, required for publishing
- **ApproverIdentity**: The validated identity of a human approver, constructed from IDP-issued `id_token` and `userinfo_token` claims
- **ClawdiaAdmin**: A human user with `role: clawdia-admin` in their `userinfo_token`, authenticated by `account.gluu.org`, authorized to approve live publishing

---

## Requirements

### Requirement 1: Policy-Constrained Agent Planning

**User Story:** As a campaign operator, I want Clawdia to query applicable policy constraints before planning any action, so that the agent plans within policy rather than reacting to denials.

#### Acceptance Criteria

1. WHEN the MainAgent receives a campaign goal, THE MainAgent SHALL invoke `query_authorization_constraints` before constructing any execution plan
2. THE MainAgent SHALL incorporate the returned constraints (allowed sources, permitted tools, publishing requirements) into its plan before delegating to subagents
3. IF `query_authorization_constraints` returns an error or empty constraint set, THEN THE MainAgent SHALL halt planning and return a structured error to the caller
4. THE MainAgent SHALL NOT propose any action in its plan that is not permitted by the retrieved constraints

---

### Requirement 2: Policy Enforcement on Every Tool Call

**User Story:** As a system operator, I want every tool invocation to be authorized by an embedded Cedar policy engine, so that no tool executes without explicit policy approval.

#### Acceptance Criteria

1. WHEN any agent (MainAgent, ResearchSubAgent, or MediaSubAgent) invokes a tool, THE component's embedded Cedarling instance SHALL evaluate an authorization request inline before execution
2. WHEN Cedarling returns `Allow`, THE component SHALL forward the tool call to the tool implementation
3. WHEN Cedarling returns `Deny`, THE component SHALL return a structured denial response to the calling agent without executing the tool
4. WHEN a denial is returned to the MainAgent, THE MainAgent SHALL treat the denial as a replanning signal and attempt an alternative action within policy
5. THE component SHALL log every authorization decision (principal, action, resource, decision, timestamp) to the append-only audit log

---

### Requirement 3: Bounded Subagent Delegation

**User Story:** As a system operator, I want subagents to operate only within explicitly granted, time-bounded delegations, so that subagents cannot exceed their assigned authority regardless of prompt content.

#### Acceptance Criteria

1. WHEN the MainAgent spawns a subagent, THE MainAgent SHALL invoke `delegate_authorization` to create a DelegationRecord specifying the permitted actions, allowed resources, and TTL expiration timestamp
2. THE embedded Cedarling instance in each subagent SHALL evaluate all tool calls against the subagent's DelegationRecord in addition to base policy
3. WHEN a subagent's DelegationRecord TTL has expired, THE embedded Cedarling SHALL deny all further tool calls from that subagent
4. THE ResearchSubAgent SHALL be delegated only the actions: `search_sources`, `fetch_article`, `extract_claims`
5. THE MediaSubAgent SHALL be delegated only the actions: `generate_campaign_image`, `draft_instagram_caption`
6. IF a ResearchSubAgent attempts to invoke any publish action, THEN THE embedded Cedarling SHALL deny the request and return a confinement violation response
7. IF a MediaSubAgent attempts to invoke any search or fetch action, THEN THE embedded Cedarling SHALL deny the request and return a confinement violation response

---

### Requirement 4: Research Restricted to Approved Domains

**User Story:** As a campaign operator, I want research to be restricted to policy-approved sources, so that Clawdia's claims are grounded in credible, pre-vetted information.

#### Acceptance Criteria

1. WHEN the ResearchSubAgent invokes `search_sources`, THE embedded Cedarling SHALL verify the target domain is in the ApprovedDomain list before allowing execution
2. WHEN the ResearchSubAgent invokes `fetch_article`, THE embedded Cedarling SHALL verify the article URL's domain is in the ApprovedDomain list before allowing execution
3. IF a requested domain is not in the ApprovedDomain list, THEN THE embedded Cedarling SHALL deny the request and return the list of approved domains to the ResearchSubAgent
4. THE ResearchSubAgent SHALL attach a ClaimCitation to every factual claim extracted via `extract_claims`, referencing the source URL and domain

#### Approved Domain List

The following domains are approved for research. Any domain not in this list is denied by policy.

| Domain | Tier | Label |
|---|---|---|
| `fisheries.noaa.gov` | Tier 1 — Government/Scientific | NOAA Fisheries |
| `fao.org` | Tier 1 — Government/Scientific | FAO |
| `ices.dk` | Tier 1 — Government/Scientific | ICES |
| `ipbes.net` | Tier 1 — Government/Scientific | IPBES |
| `oceana.org` | Tier 2 — NGO/Advocacy | Oceana |
| `wwf.org` | Tier 2 — NGO/Advocacy | WWF |
| `mcsuk.org` | Tier 2 — NGO/Advocacy | MCS UK |
| `edf.org` | Tier 2 — NGO/Advocacy | EDF |
| `scholar.google.com` | Tier 3 — Academic | Google Scholar |
| `sciencedirect.com` | Tier 3 — Academic | ScienceDirect |

---

### Requirement 5: Campaign Package Assembly

**User Story:** As a campaign operator, I want all research, media, and metadata assembled into a structured campaign package, so that the publish gate has a complete, verifiable artifact to evaluate.

#### Acceptance Criteria

1. WHEN the MainAgent invokes `assemble_campaign_package`, THE Campaign_Assembler SHALL combine all extracted claims, ClaimCitations, generated media references, and a computed ConfidenceScore into a single CampaignPackage
2. THE Campaign_Assembler SHALL compute the ConfidenceScore as a value between 0.0 and 1.0 based on the number of claims with valid citations divided by total claims
3. IF any claim in the CampaignPackage has no associated ClaimCitation, THEN THE Campaign_Assembler SHALL flag that claim as unverified in the CampaignPackage
4. THE Campaign_Assembler SHALL include the DelegationRecord identifiers of the subagents that produced each component in the CampaignPackage metadata

---

### Requirement 6: Policy-Governed Publishing (Publish Gate)

**User Story:** As a campaign operator, I want publishing to be controlled by deterministic Cedar policy rather than LLM judgment, so that no post goes live without meeting verifiable quality and approval thresholds.

#### Acceptance Criteria

1. WHEN the MainAgent invokes `publish_instagram_live`, THE PublishGate SHALL evaluate the CampaignPackage against the publish policy before allowing the Instagram API call
2. THE PublishGate SHALL deny live publishing if the CampaignPackage ConfidenceScore is below 0.8
3. THE PublishGate SHALL deny live publishing if any claim in the CampaignPackage is flagged as unverified
4. WHERE human approval is required by policy, THE PublishGate SHALL deny live publishing if a valid ApproverIdentity is not present (see Requirement 11)
5. WHEN live publishing is denied, THE MainAgent SHALL invoke `publish_instagram_draft` to save the campaign as a draft
6. WHEN `publish_instagram_draft` is invoked, THE PublishGate SHALL allow the draft save without requiring human approval or a minimum ConfidenceScore
7. WHEN a campaign is published (draft or live), THE PublishGate SHALL record the publish decision, the policy evaluation result, and the CampaignPackage identifier in the audit log

---

### Requirement 7: Browser-Side Policy Visibility via Cedarling

**User Story:** As a developer or demo observer, I want to see Cedar policy evaluations rendered in the browser UI in real time, so that the policy-inside-the-loop architecture is visible and auditable.

#### Acceptance Criteria

1. THE Browser_UI SHALL load the Cedarling WASM module on startup and display its initialization status
2. WHEN any authorization decision is made during a campaign run, THE Browser_UI SHALL display the principal, action, resource, decision (Allow/Deny), and timestamp for that decision
3. THE Browser_UI SHALL display the current agent phase (Planning, Researching, Generating, Assembling, Publishing) updated in real time as the MainAgent progresses
4. WHEN a tool call is denied by the embedded Cedarling, THE Browser_UI SHALL highlight the denial in the authorization log with the denial reason
5. THE Browser_UI SHALL display the final CampaignPackage summary including ConfidenceScore, claim count, citation count, and publish status

---

### Requirement 8: Demo Scenario — Overreach Blocked

**User Story:** As a demo operator, I want to demonstrate that a prompt requesting immediate live publishing is blocked by policy and replanned as a draft, so that the audience sees policy enforcement in action.

#### Acceptance Criteria

1. WHEN the MainAgent receives a goal containing an instruction to publish immediately without meeting confidence or approval thresholds, THE embedded Cedarling SHALL deny the `publish_instagram_live` call
2. WHEN `publish_instagram_live` is denied, THE MainAgent SHALL replan and invoke `publish_instagram_draft` instead
3. THE Browser_UI SHALL display the denial event and the subsequent replanning step as distinct, labeled events in the authorization log

---

### Requirement 9: Demo Scenario — Subagent Confinement Proof

**User Story:** As a demo operator, I want to demonstrate that a ResearchSubAgent attempting to publish is denied, so that the audience sees subagent confinement enforced by policy rather than by instruction.

#### Acceptance Criteria

1. WHEN the ResearchSubAgent attempts to invoke `publish_instagram_draft` or `publish_instagram_live`, THE embedded Cedarling SHALL deny the request because the ResearchSubAgent's DelegationRecord does not include publish actions
2. THE component SHALL return a confinement violation response to the ResearchSubAgent
3. THE Browser_UI SHALL display the confinement violation as a labeled denial event in the authorization log, identifying the ResearchSubAgent as the principal and the attempted publish action as the denied action

---

### Requirement 10: Audit Log Integrity

**User Story:** As a system operator, I want a complete, tamper-evident audit log of all authorization decisions and publish events, so that the system's behavior can be reviewed and verified after the fact.

#### Acceptance Criteria

1. THE Audit_Log SHALL record every authorization decision with: principal identifier, action, resource, Cedar policy decision, denial reason (if denied), and ISO 8601 timestamp
2. THE Audit_Log SHALL record every publish event (draft or live) with: CampaignPackage identifier, publish type, ConfidenceScore, claim count, citation count, and timestamp
3. THE Audit_Log SHALL be append-only — no entry SHALL be modified or deleted after creation
4. WHEN the Browser_UI requests the audit log, THE Audit_Log SHALL return all entries in chronological order

---

### Requirement 11: Approver Identity Verification

**User Story:** As a system operator, I want live publishing to require a human approver authenticated by `account.gluu.org` with the `clawdia-admin` role, so that approval cannot be forged or bypassed by prompt injection.

#### Acceptance Criteria

1. WHEN live publishing requires human approval, THE PublishGate SHALL require the caller to present both an `id_token` and a `userinfo_token` issued by `account.gluu.org`
2. THE PublishGate SHALL deny live publishing if the `iss` claim in the `id_token` is not exactly `https://account.gluu.org`
3. THE PublishGate SHALL deny live publishing if the `role` claim in the `userinfo_token` is not exactly `clawdia-admin`
4. THE PublishGate SHALL deny live publishing if either the `id_token` or the `userinfo_token` has expired (i.e., the `exp` claim is in the past at the time of the publish request)
5. THE PublishGate SHALL deny live publishing if either token is absent
6. WHEN token validation succeeds, THE PublishGate SHALL construct an `ApproverIdentity` from the validated token claims and pass it to the Cedar authorization context
7. THE Audit_Log SHALL record the `sub` claim from the `id_token` and the token validation result (`valid`, `invalid_issuer`, `invalid_role`, `expired`, or `missing`) for every live publish attempt
