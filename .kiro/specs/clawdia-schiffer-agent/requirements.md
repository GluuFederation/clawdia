# Requirements Document

## Introduction

Clawdia Schiffer is a policy-governed activist AI agent that campaigns against bottom trawling. The system demonstrates authorization embedded inside the agent loop — not bolted on afterward. Clawdia researches facts from approved sources, generates campaign media, drafts captions, and publishes to Instagram, while Cedar policies continuously constrain what it can research, what claims it can make, what tools it can use, what subagents it can spawn, and whether it can publish.

The core insight: Clawdia is safe not because of prompting, but because authority is bounded at every layer — browser, agent loop, subagents, tool calls, APIs, and publishing.

## Glossary

- **MainAgent**: The primary orchestrating agent (Clawdia) that plans, delegates, and assembles campaign packages
- **ResearchSubAgent**: A bounded subagent permitted only to search, fetch, and summarize from approved domains
- **MediaSubAgent**: A bounded subagent permitted only to generate images, video, and captions
- **PEP**: Policy Enforcement Point — intercepts every tool call and consults the PDP before allowing execution
- **PDP**: Policy Decision Point — the Cedar policy engine that evaluates authorization requests
- **Cedarling**: The WASM-compiled Cedar policy engine running in the browser
- **DelegationRecord**: An explicit, scoped, time-bounded authorization grant from a parent agent to a subagent
- **CampaignPackage**: The assembled output containing claims, citations, media, and a confidence score
- **PublishGate**: The policy-controlled component that decides whether a campaign is published as draft or live
- **TPE**: Typed Partial Evaluation — Cedar's mechanism for querying applicable constraints before acting
- **ConfidenceScore**: A numeric value (0.0–1.0) representing the verified confidence in campaign claims
- **ApprovedDomain**: A domain explicitly listed in policy as a permitted research source
- **ClaimCitation**: A source reference attached to a factual claim, required for publishing

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

**User Story:** As a system operator, I want every tool invocation to be intercepted and authorized by Cedar policy, so that no tool executes without explicit policy approval.

#### Acceptance Criteria

1. WHEN any agent (MainAgent, ResearchSubAgent, or MediaSubAgent) invokes a tool, THE PEP SHALL intercept the call and submit an authorization request to the PDP before execution
2. WHEN the PDP returns `Allow`, THE PEP SHALL forward the tool call to the tool implementation
3. WHEN the PDP returns `Deny`, THE PEP SHALL return a structured denial response to the calling agent without executing the tool
4. WHEN the PEP returns a denial, THE MainAgent SHALL treat the denial as a replanning signal and attempt an alternative action within policy
5. THE PEP SHALL log every authorization decision (principal, action, resource, decision, timestamp) to an append-only audit log

---

### Requirement 3: Bounded Subagent Delegation

**User Story:** As a system operator, I want subagents to operate only within explicitly granted, time-bounded delegations, so that subagents cannot exceed their assigned authority regardless of prompt content.

#### Acceptance Criteria

1. WHEN the MainAgent spawns a subagent, THE MainAgent SHALL invoke `delegate_authorization` to create a DelegationRecord specifying the permitted actions, allowed resources, and TTL expiration timestamp
2. THE PDP SHALL evaluate all subagent tool calls against the subagent's DelegationRecord in addition to base policy
3. WHEN a subagent's DelegationRecord TTL has expired, THE PDP SHALL deny all further tool calls from that subagent
4. THE ResearchSubAgent SHALL be delegated only the actions: `search_sources`, `fetch_article`, `extract_claims`
5. THE MediaSubAgent SHALL be delegated only the actions: `generate_campaign_image`, `draft_instagram_caption`
6. IF a ResearchSubAgent attempts to invoke any publish action, THEN THE PDP SHALL deny the request and THE PEP SHALL return a confinement violation response
7. IF a MediaSubAgent attempts to invoke any search or fetch action, THEN THE PDP SHALL deny the request and THE PEP SHALL return a confinement violation response

---

### Requirement 4: Research Restricted to Approved Domains

**User Story:** As a campaign operator, I want research to be restricted to policy-approved sources, so that Clawdia's claims are grounded in credible, pre-vetted information.

#### Acceptance Criteria

1. WHEN the ResearchSubAgent invokes `search_sources`, THE PDP SHALL verify the target domain is in the ApprovedDomain list before allowing execution
2. WHEN the ResearchSubAgent invokes `fetch_article`, THE PDP SHALL verify the article URL's domain is in the ApprovedDomain list before allowing execution
3. IF a requested domain is not in the ApprovedDomain list, THEN THE PEP SHALL deny the request and return the list of approved domains to the ResearchSubAgent
4. THE ResearchSubAgent SHALL attach a ClaimCitation to every factual claim extracted via `extract_claims`, referencing the source URL and domain

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
4. WHERE human approval is required by policy, THE PublishGate SHALL deny live publishing if the CampaignPackage does not carry a valid human approval token
5. WHEN live publishing is denied, THE MainAgent SHALL invoke `publish_instagram_draft` to save the campaign as a draft
6. WHEN `publish_instagram_draft` is invoked, THE PublishGate SHALL allow the draft save without requiring a human approval token or minimum ConfidenceScore
7. WHEN a campaign is published (draft or live), THE PublishGate SHALL record the publish decision, the policy evaluation result, and the CampaignPackage identifier in the audit log

---

### Requirement 7: Browser-Side Policy Visibility via Cedarling

**User Story:** As a developer or demo observer, I want to see Cedar policy evaluations rendered in the browser UI in real time, so that the policy-inside-the-loop architecture is visible and auditable.

#### Acceptance Criteria

1. THE Browser_UI SHALL load the Cedarling WASM module on startup and display its initialization status
2. WHEN any authorization decision is made during a campaign run, THE Browser_UI SHALL display the principal, action, resource, decision (Allow/Deny), and timestamp for that decision
3. THE Browser_UI SHALL display the current agent phase (Planning, Researching, Generating, Assembling, Publishing) updated in real time as the MainAgent progresses
4. WHEN a tool call is denied by the PEP, THE Browser_UI SHALL highlight the denial in the authorization log with the denial reason
5. THE Browser_UI SHALL display the final CampaignPackage summary including ConfidenceScore, claim count, citation count, and publish status

---

### Requirement 8: Demo Scenario — Overreach Blocked

**User Story:** As a demo operator, I want to demonstrate that a prompt requesting immediate live publishing is blocked by policy and replanned as a draft, so that the audience sees policy enforcement in action.

#### Acceptance Criteria

1. WHEN the MainAgent receives a goal containing an instruction to publish immediately without meeting confidence or approval thresholds, THE PEP SHALL deny the `publish_instagram_live` call
2. WHEN `publish_instagram_live` is denied, THE MainAgent SHALL replan and invoke `publish_instagram_draft` instead
3. THE Browser_UI SHALL display the denial event and the subsequent replanning step as distinct, labeled events in the authorization log

---

### Requirement 9: Demo Scenario — Subagent Confinement Proof

**User Story:** As a demo operator, I want to demonstrate that a ResearchSubAgent attempting to publish is denied, so that the audience sees subagent confinement enforced by policy rather than by instruction.

#### Acceptance Criteria

1. WHEN the ResearchSubAgent attempts to invoke `publish_instagram_draft` or `publish_instagram_live`, THE PDP SHALL deny the request because the ResearchSubAgent's DelegationRecord does not include publish actions
2. THE PEP SHALL return a confinement violation response to the ResearchSubAgent
3. THE Browser_UI SHALL display the confinement violation as a labeled denial event in the authorization log, identifying the ResearchSubAgent as the principal and the attempted publish action as the denied action

---

### Requirement 10: Audit Log Integrity

**User Story:** As a system operator, I want a complete, tamper-evident audit log of all authorization decisions and publish events, so that the system's behavior can be reviewed and verified after the fact.

#### Acceptance Criteria

1. THE Audit_Log SHALL record every PEP authorization decision with: principal identifier, action, resource, Cedar policy decision, denial reason (if denied), and ISO 8601 timestamp
2. THE Audit_Log SHALL record every publish event (draft or live) with: CampaignPackage identifier, publish type, ConfidenceScore, claim count, citation count, and timestamp
3. THE Audit_Log SHALL be append-only — no entry SHALL be modified or deleted after creation
4. WHEN the Browser_UI requests the audit log, THE Audit_Log SHALL return all entries in chronological order
