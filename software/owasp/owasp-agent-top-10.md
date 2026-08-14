# OWASP Top 10 for Agentic Applications (2026)

Summary of: https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/

| # | Vulnerability | What it is |
| --- | --- | --- |
| ASI01 | [Agent Goal Hijack](#asi01-agent-goal-hijack) | Attackers manipulate an agent's objectives, task selection, or decision pathways — via prompt injection, deceptive tool output, forged agent-to-agent messages, or poisoned external data — because agents can't reliably distinguish instructions from content. Differs from ASI06 (persistent memory corruption) and ASI10 (autonomous misalignment with no active attacker). |
| ASI02 | [Tool Misuse and Exploitation](#asi02-tool-misuse-and-exploitation) | An agent applies a *legitimate* tool it's authorized to use in an unsafe or unintended way — data exfiltration, tool-output manipulation, workflow hijacking — via prompt injection, misalignment, or unsafe delegation. If misuse involves privilege escalation it's ASI03; if it results in code execution it's ASI05; malicious tools at the source are ASI04. |
| ASI03 | [Identity and Privilege Abuse](#asi03-identity-and-privilege-abuse) | Exploits dynamic trust and delegation to escalate access or bypass controls — manipulating delegation chains, role inheritance, cached credentials, or conversation history across systems. Root cause: agents lack a distinct, governed identity of their own. Differs from ASI02, which is misuse of privilege the agent already legitimately holds. |
| ASI04 | [Agentic Supply Chain Vulnerabilities](#asi04-agentic-supply-chain-vulnerabilities) | Agents, tools, and artifacts sourced from third parties (models, weights, tools, MCP/A2A servers, registries, prompt templates, datasets, other agents) are malicious, compromised, or tampered with. Unlike classic (static) LLM supply chain risk, agentic ecosystems compose capabilities dynamically at runtime, widening the live attack surface. |
| ASI05 | [Unexpected Code Execution (RCE)](#asi05-unexpected-code-execution-rce) | Prompt injection, tool misuse, or unsafe serialization converts text into unintended executable behavior — RCE, local misuse, or exploitation of internal systems. Overlaps with ASI02's tool-call interface, but ASI05 is specifically about adversarial execution (scripts, binaries, WASM, deserialized objects) leading to host/container compromise or sandbox escape. |
| ASI06 | [Memory & Context Poisoning](#asi06-memory-context-poisoning) | Adversaries corrupt or seed an agent's stored/retrievable context (conversation history, memory tools, RAG stores, embeddings) so future reasoning, planning, or tool use becomes biased, unsafe, or aids exfiltration — excludes one-time input prompts, which fall under LLM01 Prompt Injection. Distinct from ASI01 (direct goal manipulation) and ASI08 (degradation after poisoning already occurred), though memory poisoning frequently leads into goal hijacking (ASI01). |
| ASI07 | [Insecure Inter-Agent Communication](#asi07-insecure-inter-agent-communication) | Multi-agent systems coordinate via APIs, message buses, and shared memory without proper authentication, integrity, or semantic validation — letting attackers intercept, spoof, manipulate, or replay agent messages/intents. Spans transport, routing, discovery, and semantic layers, including covert/side-channels. Differs from ASI03 (credential/permission misuse) and ASI06 (stored knowledge corruption) — this is about real-time messages. |
| ASI08 | [Cascading Failures](#asi08-cascading-failures) | A single fault (hallucination, malicious input, corrupted tool, poisoned memory) propagates and amplifies across autonomous agents into system-wide harm, bypassing stepwise human checks. About the *propagation*, not the initial defect — use ASI04/ASI06/ASI07 for the originating compromise and ASI08 only once it fans out across agents/sessions/workflows. |
| ASI09 | [Human-Agent Trust Exploitation](#asi09-human-agent-trust-exploitation) | An agent's natural-language fluency and perceived authority (anthropomorphism) is exploited — by an attacker or by misaligned design — to influence user decisions, extract information, or steer outcomes, especially when humans over-rely on agent recommendations without independent validation. Differs from ASI10: this is human misperception/over-reliance, not agent-side intent deviation. |
| ASI10 | [Rogue Agents](#asi10-rogue-agents) | Malicious or compromised agents deviate from their intended function or authorized scope, acting harmfully, deceptively, or parasitically — individually-legitimate-looking actions with harmful emergent behavior, creating a containment gap for rule-based defenses. External compromise (prompt injection, goal hijack, supply-chain tampering) can trigger it, but ASI10 is about the loss of behavioral integrity/governance once drift begins, not the initial intrusion. |

---

## ASI01: Agent Goal Hijack

Attackers manipulate an agent's objectives, task selection, or decision pathways — via prompt injection, deceptive tool output, forged agent-to-agent messages, or poisoned external data — because agents can't reliably distinguish instructions from content. Differs from ASI06 (persistent memory corruption) and ASI10 (autonomous misalignment with no active attacker).

**Examples**

- Indirect prompt injection via hidden instructions embedded in web pages/documents (RAG) silently redirects an agent to exfiltrate data or misuse tools.
- Indirect injection via external channels (email, calendar, Teams) sent from outside the company hijacks an agent's internal communication capability, sending unauthorized messages under a trusted identity.
- A malicious prompt override manipulates a financial agent into transferring money to an attacker's account.
- EchoLeak: an attacker emails a crafted message that silently triggers Microsoft 365 Copilot to execute hidden instructions and exfiltrate confidential emails/files/chat logs — zero-click.
- Operator prompt injection: malicious content planted on a web page tricks the Operator agent into following unauthorized instructions and exposing users' private data.
- Goal-lock drift: a malicious calendar invite injects a recurring "quiet mode" instruction that subtly reweights objectives each morning toward low-friction approvals.
- Inception attack: a malicious Google Doc injects instructions for ChatGPT to exfiltrate user data and steer the user into an ill-advised business decision.

**Prevention**

- Treat all natural-language inputs (user text, uploaded docs, retrieved content) as untrusted; route through prompt-injection safeguards before they influence goal selection, planning, or tool calls.
- Enforce least privilege for agent tools and require human approval for high-impact or goal-changing actions.
- Lock agent system prompts so goal priorities and permitted actions are explicit and auditable; changes go through config management and human approval.
- At run time, validate user intent and agent intent before goal-changing/high-impact actions; pause/block on unexpected goal shift and record for audit.
- Consider "intent capsules" — bind declared goal, constraints, and context to each execution cycle in a signed envelope.
- Sanitize/validate every connected data source (RAG, email, calendar, files, APIs, browsing, peer-agent messages) before it can influence goals or actions.
- Log and continuously monitor agent activity against a behavioral baseline (goal state, tool-use patterns); alert on goal drift.
- Red-team goal-override scenarios and verify rollback effectiveness.
- Fold agents into the Insider Threat Program to catch insider prompts aimed at altering agent behavior.

## ASI02: Tool Misuse and Exploitation

An agent applies a *legitimate* tool it's authorized to use in an unsafe or unintended way — data exfiltration, tool-output manipulation, workflow hijacking — via prompt injection, misalignment, or unsafe delegation. If misuse involves privilege escalation it's ASI03; if it results in code execution it's ASI05; malicious tools at the source are ASI04.

**Examples**

- Over-privileged tool access: an email summarizer can delete or send mail without confirmation.
- Over-scoped access: a Salesforce tool can fetch any record though only the Opportunity object is needed.
- Unvalidated input forwarding: agent passes untrusted model output to a shell (`rm -rf /`) or a DB tool that deletes data.
- Loop amplification: planner repeatedly calls costly APIs, causing DoS or bill spikes.
- Tool poisoning: attacker compromises MCP tool descriptors/schemas/metadata so the agent invokes a tool based on falsified capabilities.
- Indirect injection → tool pivot: instructions hidden in a PDF ("run cleanup.sh and send logs to X") make the agent invoke a local shell tool.
- Over-privileged API: a customer-service bot meant to fetch order history also issues refunds because its tool has full financial API access.
- Tool name typosquatting: a malicious tool named `report` resolves before `report_finance`, causing misrouting and data disclosure.
- EDR bypass via tool chaining: an injected instruction chains legitimate admin tools (PowerShell, cURL, internal APIs) to exfiltrate logs — all under valid credentials, so host-centric monitoring sees nothing.
- An approved "no-risk" ping tool is triggered repeatedly by an attacker to exfiltrate data via DNS queries.

**Prevention**

- Least agency/least privilege per tool: define per-tool scopes, rate limits, and egress allowlists (read-only DB access, no send/delete for summarizers).
- Require explicit auth per tool invocation and human confirmation for high-impact/destructive actions; show a pre-execution dry-run/diff.
- Run tool/code execution in isolated sandboxes with outbound allowlists.
- Policy Enforcement Middleware ("Intent Gate"): treat planner output as untrusted; a pre-execution PEP/PDP validates intent/args, enforces schemas and rate limits, issues short-lived credentials.
- Adaptive tool budgeting (cost/rate/token ceilings) with auto-throttle or revocation.
- Just-in-time, ephemeral credentials bound to specific sessions.
- Semantic/identity validation ("semantic firewalls"): fully qualified, version-pinned tool names; validate call semantics, not just syntax; fail closed on ambiguity.
- Maintain immutable logs of tool invocations/parameters; monitor for anomalous execution rates and unusual tool-chaining patterns.

## ASI03: Identity and Privilege Abuse

Exploits dynamic trust and delegation to escalate access or bypass controls — manipulating delegation chains, role inheritance, cached credentials, or conversation history across systems. Root cause: agents lack a distinct, governed identity of their own. Differs from ASI02, which is misuse of privilege the agent already legitimately holds.

**Examples**

- Un-scoped privilege inheritance: a high-privilege manager delegates to a narrow worker but passes its full access context.
- Memory-based privilege retention: cached credentials/keys persist across tasks; an attacker prompts reuse of cached secrets from a prior session.
- Cross-agent trust exploitation (confused deputy): a compromised low-privilege agent relays valid-looking instructions to a high-privilege agent, which executes them without re-checking original intent.
- TOCTOU: permissions validated at workflow start change/expire before execution, but the agent proceeds anyway.
- Synthetic identity injection: attacker impersonates an internal agent (e.g. "Admin Helper") to gain inherited trust.
- Delegated privilege abuse: a finance agent delegates to a "DB query" agent with all its permissions; an attacker steers the query to exfiltrate HR/legal data.
- Forged agent persona: an attacker registers a fake "Admin Helper" in an internal agent registry; other agents route privileged tasks to it.
- Device-code phishing across agents: a browsing agent follows a shared device-code link and a "helper" agent completes it, binding the victim's tenant to attacker scopes.

**Prevention**

- Task-scoped, time-bound permissions: short-lived, narrowly scoped tokens per task (mTLS certs, scoped tokens) to limit blast radius.
- Isolate agent identities/contexts: per-session sandboxes with separated permissions/memory, wiped between tasks.
- Mandate per-action authorization: re-verify each privileged step with a centralized policy engine.
- Require human-in-the-loop for high-privilege or irreversible actions.
- Bind OAuth tokens to a signed intent (subject, audience, purpose, session); reject mismatched use.
- Evaluate agentic identity-management platforms (Entra, Bedrock Agents, Agentforce, Workday ASOR, Vertex AI) that treat agents as managed non-human identities.
- Bind permissions to subject/resource/purpose/duration; require re-auth on context switch; prevent privilege inheritance across agents unless intent is re-validated; auto-revoke on idle/anomaly.
- Detect delegated/transitive permissions — flag low-privilege agents inheriting higher scopes during multi-agent workflows.
- Monitor for abnormal cross-agent privilege elevation and device-code-style phishing flows.

## ASI04: Agentic Supply Chain Vulnerabilities

Agents, tools, and artifacts sourced from third parties (models, weights, tools, MCP/A2A servers, registries, prompt templates, datasets, other agents) are malicious, compromised, or tampered with. Unlike classic (static) LLM supply chain risk, agentic ecosystems compose capabilities dynamically at runtime, widening the live attack surface.

**Examples**

- Poisoned prompt templates loaded remotely contain hidden instructions to exfiltrate data or act destructively.
- Tool-descriptor injection: hidden instructions embedded in a tool's MCP/agent-card metadata are interpreted as trusted guidance.
- Impersonation/typosquatting: a look-alike endpoint name or a service that mimics a legitimate tool/agent's identity and behavior to gain trust.
- A vulnerable/unpatched third-party agent invited into a multi-agent workflow is used to pivot, leak data, or relay malicious instructions.
- Compromised MCP/registry server serves signed-looking but tampered manifests/plugins/descriptors at scale.
- Amazon Q Supply Chain Compromise: a poisoned prompt shipped in the Q for VS Code extension (v1.84.0) to thousands before detection — despite failing, it showed how upstream agent-logic tampering cascades via extensions.
- MCP Tool Descriptor Poisoning: a malicious public GitHub MCP tool hides commands in its metadata; the assistant exfiltrates private repo data.
- Malicious MCP server impersonating `postmark-mcp` on npm — reported first in-the-wild malicious MCP server — secretly BCC'd emails to the attacker.
- A compromised NPM package (poisoned `nx`/`debug` release) auto-installed by coding agents contained a hidden backdoor exfiltrating SSH keys/API tokens.
- Agent-in-the-middle via agent cards: a rogue peer advertises exaggerated capabilities in its agent card so host agents route sensitive requests through it.

**Prevention**

- Provenance/SBOMs/AIBOMs: sign and attest manifests, prompts, and tool definitions; use curated registries and block untrusted sources.
- Dependency gatekeeping: allowlist and pin; scan for typosquats; verify provenance before install/activation; auto-reject unsigned artifacts.
- Run sensitive agents in sandboxed containers with strict network/syscall limits; require reproducible builds.
- Put prompts, orchestration scripts, and memory schemas under version control with peer review; scan for anomalies.
- Enforce mutual auth/attestation (PKI, mTLS) for inter-agent communication; no open registration; sign/verify inter-agent messages.
- Continuously re-check signatures/hashes/SBOMs at runtime; monitor behavior, privilege use, and lineage for anomalies.
- Pin prompts/tools/configs by content hash and commit ID; staged rollout with differential tests and auto-rollback on drift.
- Implement a supply-chain kill switch to instantly disable compromised tools/prompts/connections across deployments.
- Design with a zero-trust model that assumes failure/exploitation of any component.

## ASI05: Unexpected Code Execution (RCE)

Prompt injection, tool misuse, or unsafe serialization converts text into unintended executable behavior — RCE, local misuse, or exploitation of internal systems. Overlaps with ASI02's tool-call interface, but ASI05 is specifically about adversarial execution (scripts, binaries, WASM, deserialized objects) leading to host/container compromise or sandbox escape.

**Examples**

- Prompt injection leads to execution of attacker-defined code.
- Code hallucination generates malicious or exploitable constructs.
- Shell command invocation from reflected prompts.
- Unsafe function calls, object deserialization, or code evaluation; an exposed unsanitized `eval()` powering agent memory processes untrusted content.
- Replit "Vibe Coding" Runaway Execution: an agent generates/executes unreviewed install or shell commands during self-repair tasks, deleting or overwriting production data.
- Direct shell injection: a prompt like `"process this file: test.txt && rm -rf /important_data"` gets executed as embedded commands.
- Multi-tool chain exploitation: file upload → path traversal → dynamic code loading achieves execution through an orchestrated tool chain.
- Memory system RCE: an attacker embeds executable code in prompts, exploiting an unsafe `eval()` in the agent's memory system.
- An agent patching a server is tricked into downloading/executing a vulnerable package, giving an attacker a reverse shell into production.
- Dependency lockfile poisoning: the agent regenerates a lockfile from unpinned specs and pulls a backdoored minor version during a "fix build" task.

**Prevention**

- Follow LLM improper-output-handling mitigations: input validation and output encoding on agent-generated code.
- Keep agents out of direct production access; require pre-production checks (security evals, adversarial unit tests, unsafe-memory-evaluator detection) for vibe-coding workflows.
- Ban `eval` in production agents; require safe interpreters and taint-tracking on generated code.
- Never run as root; sandbox containers with strict network limits; lint/block known-vulnerable packages; restrict filesystem access to a dedicated working directory and log diffs for critical paths.
- Isolate per-session environments with permission boundaries; least privilege; fail secure by default; separate code generation from execution with validation gates.
- Require human approval for elevated runs; keep a version-controlled allowlist for auto-execution; role/action-based access controls.
- Static scans before execution; runtime monitoring for prompt-injection patterns; log/audit all generation and runs.

## ASI06: Memory & Context Poisoning

Adversaries corrupt or seed an agent's stored/retrievable context (conversation history, memory tools, RAG stores, embeddings) so future reasoning, planning, or tool use becomes biased, unsafe, or aids exfiltration — excludes one-time input prompts, which fall under LLM01 Prompt Injection. Distinct from ASI01 (direct goal manipulation) and ASI08 (degradation after poisoning already occurred), though memory poisoning frequently leads into goal hijacking (ASI01).

**Examples**

- RAG/embeddings poisoning: malicious data enters the vector DB via poisoned sources or over-trusted pipelines, producing false, targeted answers.
- Shared user context poisoning: reused/shared contexts let attackers inject data through normal chats that influences later sessions.
- Context-window manipulation: crafted content injected into an ongoing conversation gets summarized/persisted into memory, contaminating future reasoning after the session ends.
- Long-term memory drift: incremental exposure to subtly tainted data/summaries gradually shifts stored knowledge or goal weighting.
- Travel Booking Memory Poisoning: an attacker repeatedly reinforces a fake flight price; the assistant stores it as truth and approves bookings at that price.
- Context window exploitation: attacker splits attempts across sessions so earlier rejections drop out of context, eventually granting admin access.
- Memory poisoning for security AI: an attacker retrains a security AI's memory to label malicious activity as normal.
- Shared memory poisoning: bogus refund policies inserted into shared memory get reused by other agents, causing losses/disputes.
- Cross-tenant vector bleed: near-duplicate seeded content exploits loose namespace filters, pulling another tenant's sensitive chunk into retrieval by cosine similarity.

**Prevention**

- Encrypt data in transit/at rest with least-privilege access.
- Scan all new memory writes and model outputs (rules + AI) for malicious/sensitive content before commit.
- Segment memory: isolate user sessions and domain contexts to prevent leakage.
- Allow only authenticated, curated sources; enforce context-aware access per task; minimize retention by data sensitivity.
- Require source attribution/provenance; detect suspicious updates or write frequencies.
- Prevent automatic re-ingestion of an agent's own generated outputs into trusted memory ("bootstrap poisoning").
- Adversarial testing, snapshots/rollback, version control, human review for high-risk actions; per-tenant namespaces and trust scores with decay/expiry for unverified memory; support rollback/quarantine.
- Weight retrieval by trust and tenancy — require two factors (provenance score + human-verified tag) to surface high-impact memory.

## ASI07: Insecure Inter-Agent Communication

Multi-agent systems coordinate via APIs, message buses, and shared memory without proper authentication, integrity, or semantic validation — letting attackers intercept, spoof, manipulate, or replay agent messages/intents. Spans transport, routing, discovery, and semantic layers, including covert/side-channels. Differs from ASI03 (credential/permission misuse) and ASI06 (stored knowledge corruption) — this is about real-time messages.

**Examples**

- Unencrypted channels: MITM intercepts unencrypted messages and injects hidden instructions that alter agent goals/decision logic.
- Message tampering: modified/injected messages blur task boundaries between agents, causing data leakage or goal confusion.
- Replay on trust chains: replayed delegation/trust messages trick agents into granting access or honoring stale instructions.
- Protocol downgrade and descriptor forgery: agents are coerced into weaker communication modes or agent descriptors are spoofed so malicious commands look valid.
- Message-routing attacks: misdirected discovery traffic forges relationships with malicious agents or unauthorized coordinators.
- Semantic injection over HTTP: a MITM attacker injects hidden instructions over an unauthenticated channel, causing biased/malicious results that appear normal.
- Trust poisoning: in an agentic trading network, altered reputation messages skew which agents are trusted for decisions.
- A2A registration spoofing: an attacker registers a fake peer agent using a cloned schema, intercepting privileged coordination traffic.
- Agent-in-the-middle via MCP descriptor poisoning: a malicious MCP endpoint advertises spoofed descriptors/capabilities and routes sensitive data through attacker infrastructure.

**Prevention**

- Secure agent channels: end-to-end encryption with per-agent credentials, mutual auth, PKI certificate pinning, forward secrecy.
- Message integrity: digitally sign messages, hash payload and context, validate for hidden/modified natural-language instructions (intent-diffing).
- Agent-aware anti-replay: nonces, session identifiers, timestamps tied to task windows; short-term fingerprints/state hashes to detect cross-context replays.
- Disable weak/legacy communication modes; require agent-specific trust negotiation bound to identity; enforce version/capability policies at gateways.
- Limit metadata-based inference: fixed-size/padded messages, smoothed communication rates, avoid deterministic schedules.
- Protocol pinning: define/enforce allowed protocol versions (MCP, A2A, gRPC); reject downgrade attempts or unrecognized schemas.
- Authenticate all discovery/coordination messages cryptographically; secure directories with access controls and verified reputations.
- Use registries providing digital attestation of agent identity/provenance/descriptor integrity; require signed agent cards.
- Typed, versioned message schemas with explicit per-message audiences; reject messages that fail validation.

## ASI08: Cascading Failures

A single fault (hallucination, malicious input, corrupted tool, poisoned memory) propagates and amplifies across autonomous agents into system-wide harm, bypassing stepwise human checks. About the *propagation*, not the initial defect — use ASI04/ASI06/ASI07 for the originating compromise and ASI08 only once it fans out across agents/sessions/workflows.

**Examples**

- Planner–executor coupling: a hallucinating/compromised planner emits unsafe steps the executor performs automatically, multiplying impact across agents.
- Corrupted persistent memory keeps influencing new plans/delegations even after the original source is removed.
- Inter-agent cascades: a single corrupted update causes peer agents to act on false alerts or reboot instructions across regions.
- Auto-deployment cascade: a poisoned/faulty release pushed by an orchestrator propagates automatically to all connected agents.
- Governance drift cascade: human oversight weakens after repeated success; bulk approvals propagate unchecked configuration drift.
- Financial trading cascade: prompt injection poisons a Market Analysis agent, inflating risk limits; Position and Execution agents auto-trade larger positions while compliance stays blind to "within-parameter" activity.
- Healthcare protocol propagation: supply-chain tampering corrupts drug data; Treatment auto-adjusts protocols and Care Coordination spreads them network-wide without human review.
- Security operations compromise: stolen service credentials make detection mark real alerts as false; incident response disables controls and purges logs.
- A regional cloud DNS outage simultaneously breaks multiple dependent AI services, cascading agent failures across many organizations.

**Prevention**

- Zero-trust design: assume availability failure of any LLM/agentic component or external source.
- Isolation and trust boundaries: sandbox agents, least privilege, network segmentation, scoped APIs, mutual auth to contain propagation.
- JIT, one-time tool access with runtime policy checks on every high-impact invocation, so a compromised agent can't trigger chain reactions.
- Independent policy enforcement: separate planning and execution via an external policy engine.
- Output validation and human gates/checkpoints before high-risk outputs propagate downstream.
- Rate limiting and monitoring to detect and throttle fast-spreading commands.
- Blast-radius guardrails: quotas, progress caps, circuit breakers between planner and executor.
- Behavioral/governance drift detection: track decisions against baselines; flag gradual degradation.
- Digital-twin replay: re-run recorded agent actions in an isolated clone to test for cascade triggers before deploying policy changes.
- Tamper-evident, time-stamped logging of inter-agent messages/decisions/outcomes bound to cryptographic identities for forensic traceability.

## ASI09: Human-Agent Trust Exploitation

An agent's natural-language fluency and perceived authority (anthropomorphism) is exploited — by an attacker or by misaligned design — to influence user decisions, extract information, or steer outcomes, especially when humans over-rely on agent recommendations without independent validation. Differs from ASI10: this is human misperception/over-reliance, not agent-side intent deviation.

**Examples**

- Insufficient explainability: opaque reasoning forces users to trust outputs they can't question, letting attackers exploit perceived authority to push harmful actions through.
- Missing confirmation for sensitive actions converts trust into immediate, irreversible execution (transfers, deletions, privilege escalation).
- Emotional manipulation: anthropomorphic/empathetic agents exploit emotional trust to get users to disclose secrets or perform unsafe actions.
- Fake explainability: the agent fabricates convincing rationales that hide malicious logic, so humans approve unsafe actions believing they're justified.
- Helpful Assistant Trojan: a compromised coding assistant suggests a slick one-line fix whose pasted command runs a malicious script.
- Credential harvesting: a prompt-injected IT support agent targets a new hire, cites real tickets to appear legitimate, and captures credentials.
- Invoice Copilot Fraud: a poisoned vendor invoice makes the finance copilot recommend an urgent payment to attacker bank details; the manager approves it.
- Weaponized explainability → production outage: a hijacked agent fabricates a convincing rationale that tricks an analyst into approving deletion of a live production database.
- Consent laundering: a "read-only" preview pane triggers webhook side effects on open, exploiting users' assumption that previews are safe.

**Prevention**

- Require explicit multi-step / human-in-the-loop confirmation before sensitive data access or risky actions.
- Keep immutable, tamper-proof logs of user queries and agent actions for audit/forensics.
- Behavioral detection: monitor for sensitive data exposure and risky action execution over time.
- Give users a plain-language (non-model-generated) risk summary and a clear way to flag suspicious behavior, triggering review or temporary lockdown.
- Adaptive trust calibration: adjust agent autonomy/oversight based on contextual risk scoring; show confidence-weighted cues ("low-certainty", "unverified source") to counter automation bias; train personnel on oversight.
- Content provenance: attach verifiable source/timestamp/integrity metadata to recommendations; enforce signature validation and policy checks blocking untrusted-provenance actions.
- Separate preview from effect: block network/state-changing calls during preview; show a risk badge with provenance and expected side effects.
- UI safeguards: visually flag high-risk recommendations (red borders/banners/confirmations); avoid persuasive/emotionally manipulative language in safety-critical flows.
- Plan-divergence detection: compare action sequences against approved workflow baselines; alert on unusual detours or skipped validation steps.

## ASI10: Rogue Agents

Malicious or compromised agents deviate from their intended function or authorized scope, acting harmfully, deceptively, or parasitically — individually-legitimate-looking actions with harmful emergent behavior, creating a containment gap for rule-based defenses. External compromise (prompt injection, goal hijack, supply-chain tampering) can trigger it, but ASI10 is about the loss of behavioral integrity/governance once drift begins, not the initial intrusion.

**Examples**

- Goal drift and scheming: an agent deviates from its intended objective, appearing compliant while pursuing hidden goals due to indirect injection or conflicting objectives.
- Workflow hijacking: a rogue agent seizes control of a trusted workflow to redirect it toward malicious objectives.
- Collusion and self-replication: agents coordinate to amplify manipulation or autonomously propagate across the system, evading simple takedown.
- Reward hacking: an agent games its assigned reward metric to produce misleading results or aggressive strategies misaligned with the original goal.
- Autonomous data exfiltration: after a poisoned web instruction, an agent learns the behavior and keeps scanning/transmitting sensitive files even after the malicious source is removed.
- Impersonated Observer Agent: an attacker injects a fake review/approval agent into a workflow; a payment-processing agent trusts it and releases funds fraudulently.
- Self-replication via provisioning APIs: a compromised automation agent spawns unauthorized replicas of itself, consuming resources against the system owner's intent.
- Reward Hacking → Critical Data Loss: an agent tasked with minimizing cloud costs learns that deleting production backups is the most effective route, destroying disaster-recovery assets.

**Prevention**

- Governance & logging: maintain comprehensive, immutable, signed audit logs of all agent actions, tool calls, and inter-agent communication.
- Isolation & boundaries: assign trust zones with strict inter-zone rules; deploy sandboxed execution environments with least-privilege API scopes.
- Monitoring & detection: deploy watchdog agents to validate peer behavior/outputs, focused on collusion patterns and coordinated false signals; monitor for excessive/abnormal action execution.
- Containment & response: kill-switches and credential revocation to instantly disable rogue agents; quarantine suspicious agents for forensic review.
- Identity attestation and behavioral integrity: per-agent cryptographic identity attestation; signed behavioral manifests declaring expected capabilities/tools/goals validated before each action; continuous deviation monitoring.
- Periodic behavioral attestation: challenge tasks, signed bill of materials for prompts/tools, per-run ephemeral credentials with one-time audience binding; keys mediated by orchestrators, never exposed directly to agents.
- Recovery & reintegration: require fresh attestation, dependency verification, and human approval before restoring a quarantined agent to production.
