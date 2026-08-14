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

- EchoLeak: an attacker emails a crafted message that silently triggers Microsoft 365 Copilot to execute hidden instructions and exfiltrate confidential emails/files/chat logs — zero-click.
- A malicious prompt override manipulates a financial agent into transferring money to an attacker's account.

**Prevention**

- Treat all natural-language inputs (user text, uploaded docs, retrieved content) as untrusted; route through prompt-injection safeguards before they influence goal selection, planning, or tool calls.
- Enforce least privilege for agent tools and require human approval for high-impact or goal-changing actions.
- Lock agent system prompts so goal priorities and permitted actions are explicit and auditable; changes go through config management and human approval.
- At run time, validate user intent and agent intent before goal-changing/high-impact actions; pause/block on unexpected goal shift and record for audit.
- Log and continuously monitor agent activity against a behavioral baseline (goal state, tool-use patterns); alert on goal drift.

## ASI02: Tool Misuse and Exploitation

An agent applies a *legitimate* tool it's authorized to use in an unsafe or unintended way — data exfiltration, tool-output manipulation, workflow hijacking — via prompt injection, misalignment, or unsafe delegation. If misuse involves privilege escalation it's ASI03; if it results in code execution it's ASI05; malicious tools at the source are ASI04.

**Examples**

- Over-privileged API: a customer-service bot meant to fetch order history also issues refunds because its tool has full financial API access.
- Tool poisoning: attacker compromises MCP tool descriptors/schemas/metadata so the agent invokes a tool based on falsified capabilities.

**Prevention**

- Least agency/least privilege per tool: define per-tool scopes, rate limits, and egress allowlists (read-only DB access, no send/delete for summarizers).
- Require explicit auth per tool invocation and human confirmation for high-impact/destructive actions; show a pre-execution dry-run/diff.
- Run tool/code execution in isolated sandboxes with outbound allowlists.
- Policy Enforcement Middleware ("Intent Gate"): treat planner output as untrusted; a pre-execution PEP/PDP validates intent/args, enforces schemas and rate limits, issues short-lived credentials.
- Maintain immutable logs of tool invocations/parameters; monitor for anomalous execution rates and unusual tool-chaining patterns.

## ASI03: Identity and Privilege Abuse

Exploits dynamic trust and delegation to escalate access or bypass controls — manipulating delegation chains, role inheritance, cached credentials, or conversation history across systems. Root cause: agents lack a distinct, governed identity of their own. Differs from ASI02, which is misuse of privilege the agent already legitimately holds.

**Examples**

- Cross-agent trust exploitation (confused deputy): a compromised low-privilege agent relays valid-looking instructions to a high-privilege agent, which executes them without re-checking original intent.
- Delegated privilege abuse: a finance agent delegates to a "DB query" agent with all its permissions; an attacker steers the query to exfiltrate HR/legal data.

**Prevention**

- Task-scoped, time-bound permissions: short-lived, narrowly scoped tokens per task (mTLS certs, scoped tokens) to limit blast radius.
- Isolate agent identities/contexts: per-session sandboxes with separated permissions/memory, wiped between tasks.
- Mandate per-action authorization: re-verify each privileged step with a centralized policy engine.
- Require human-in-the-loop for high-privilege or irreversible actions.
- Bind permissions to subject/resource/purpose/duration; prevent privilege inheritance across agents unless intent is re-validated; auto-revoke on idle/anomaly.

## ASI04: Agentic Supply Chain Vulnerabilities

Agents, tools, and artifacts sourced from third parties (models, weights, tools, MCP/A2A servers, registries, prompt templates, datasets, other agents) are malicious, compromised, or tampered with. Unlike classic (static) LLM supply chain risk, agentic ecosystems compose capabilities dynamically at runtime, widening the live attack surface.

**Examples**

- Malicious MCP server impersonating `postmark-mcp` on npm — reported first in-the-wild malicious MCP server — secretly BCC'd emails to the attacker.
- Amazon Q Supply Chain Compromise: a poisoned prompt shipped in the Q for VS Code extension (v1.84.0) to thousands before detection — despite failing, it showed how upstream agent-logic tampering cascades via extensions.

**Prevention**

- Provenance/SBOMs/AIBOMs: sign and attest manifests, prompts, and tool definitions; use curated registries and block untrusted sources.
- Dependency gatekeeping: allowlist and pin; scan for typosquats; verify provenance before install/activation; auto-reject unsigned artifacts.
- Run sensitive agents in sandboxed containers with strict network/syscall limits; require reproducible builds.
- Enforce mutual auth/attestation (PKI, mTLS) for inter-agent communication; no open registration; sign/verify inter-agent messages.
- Continuously re-check signatures/hashes/SBOMs at runtime; monitor behavior, privilege use, and lineage for anomalies.

## ASI05: Unexpected Code Execution (RCE)

Prompt injection, tool misuse, or unsafe serialization converts text into unintended executable behavior — RCE, local misuse, or exploitation of internal systems. Overlaps with ASI02's tool-call interface, but ASI05 is specifically about adversarial execution (scripts, binaries, WASM, deserialized objects) leading to host/container compromise or sandbox escape.

**Examples**

- Replit "Vibe Coding" Runaway Execution: an agent generates/executes unreviewed install or shell commands during self-repair tasks, deleting or overwriting production data.
- Direct shell injection: a prompt like `"process this file: test.txt && rm -rf /important_data"` gets executed as embedded commands.

**Prevention**

- Follow LLM improper-output-handling mitigations: input validation and output encoding on agent-generated code.
- Ban `eval` in production agents; require safe interpreters and taint-tracking on generated code.
- Never run as root; sandbox containers with strict network limits; restrict filesystem access to a dedicated working directory and log diffs for critical paths.
- Isolate per-session environments with permission boundaries; separate code generation from execution with validation gates.
- Require human approval for elevated runs; keep a version-controlled allowlist for auto-execution.

## ASI06: Memory & Context Poisoning

Adversaries corrupt or seed an agent's stored/retrievable context (conversation history, memory tools, RAG stores, embeddings) so future reasoning, planning, or tool use becomes biased, unsafe, or aids exfiltration — excludes one-time input prompts, which fall under LLM01 Prompt Injection. Distinct from ASI01 (direct goal manipulation) and ASI08 (degradation after poisoning already occurred), though memory poisoning frequently leads into goal hijacking (ASI01).

**Examples**

- Travel Booking Memory Poisoning: an attacker repeatedly reinforces a fake flight price; the assistant stores it as truth and approves bookings at that price.
- Shared memory poisoning: bogus refund policies inserted into shared memory get reused by other agents, causing losses/disputes.

**Prevention**

- Scan all new memory writes and model outputs (rules + AI) for malicious/sensitive content before commit.
- Segment memory: isolate user sessions and domain contexts to prevent leakage.
- Allow only authenticated, curated sources; enforce context-aware access per task; minimize retention by data sensitivity.
- Require source attribution/provenance; detect suspicious updates or write frequencies.
- Adversarial testing, snapshots/rollback, version control; per-tenant namespaces and trust scores with decay/expiry for unverified memory.

## ASI07: Insecure Inter-Agent Communication

Multi-agent systems coordinate via APIs, message buses, and shared memory without proper authentication, integrity, or semantic validation — letting attackers intercept, spoof, manipulate, or replay agent messages/intents. Spans transport, routing, discovery, and semantic layers, including covert/side-channels. Differs from ASI03 (credential/permission misuse) and ASI06 (stored knowledge corruption) — this is about real-time messages.

**Examples**

- Unencrypted channels: MITM intercepts unencrypted messages and injects hidden instructions that alter agent goals/decision logic.
- A2A registration spoofing: an attacker registers a fake peer agent using a cloned schema, intercepting privileged coordination traffic.

**Prevention**

- Secure agent channels: end-to-end encryption with per-agent credentials, mutual auth, PKI certificate pinning, forward secrecy.
- Message integrity: digitally sign messages, hash payload and context, validate for hidden/modified natural-language instructions (intent-diffing).
- Agent-aware anti-replay: nonces, session identifiers, timestamps tied to task windows; short-term fingerprints/state hashes to detect cross-context replays.
- Authenticate all discovery/coordination messages cryptographically; secure directories with access controls and verified reputations.
- Use registries providing digital attestation of agent identity/provenance/descriptor integrity; require signed agent cards.

## ASI08: Cascading Failures

A single fault (hallucination, malicious input, corrupted tool, poisoned memory) propagates and amplifies across autonomous agents into system-wide harm, bypassing stepwise human checks. About the *propagation*, not the initial defect — use ASI04/ASI06/ASI07 for the originating compromise and ASI08 only once it fans out across agents/sessions/workflows.

**Examples**

- Planner–executor coupling: a hallucinating/compromised planner emits unsafe steps the executor performs automatically, multiplying impact across agents.
- Financial trading cascade: prompt injection poisons a Market Analysis agent, inflating risk limits; Position and Execution agents auto-trade larger positions while compliance stays blind to "within-parameter" activity.

**Prevention**

- Isolation and trust boundaries: sandbox agents, least privilege, network segmentation, scoped APIs, mutual auth to contain propagation.
- JIT, one-time tool access with runtime policy checks on every high-impact invocation, so a compromised agent can't trigger chain reactions.
- Independent policy enforcement: separate planning and execution via an external policy engine.
- Output validation and human gates/checkpoints before high-risk outputs propagate downstream.
- Blast-radius guardrails: quotas, progress caps, circuit breakers between planner and executor.

## ASI09: Human-Agent Trust Exploitation

An agent's natural-language fluency and perceived authority (anthropomorphism) is exploited — by an attacker or by misaligned design — to influence user decisions, extract information, or steer outcomes, especially when humans over-rely on agent recommendations without independent validation. Differs from ASI10: this is human misperception/over-reliance, not agent-side intent deviation.

**Examples**

- Missing confirmation for sensitive actions converts trust into immediate, irreversible execution (transfers, deletions, privilege escalation).
- Invoice Copilot Fraud: a poisoned vendor invoice makes the finance copilot recommend an urgent payment to attacker bank details; the manager approves it.

**Prevention**

- Require explicit multi-step / human-in-the-loop confirmation before sensitive data access or risky actions.
- Keep immutable, tamper-proof logs of user queries and agent actions for audit/forensics.
- Adaptive trust calibration: adjust agent autonomy/oversight based on contextual risk scoring; show confidence-weighted cues ("low-certainty", "unverified source") to counter automation bias.
- Content provenance: attach verifiable source/timestamp/integrity metadata to recommendations; enforce signature validation and policy checks blocking untrusted-provenance actions.
- UI safeguards: visually flag high-risk recommendations (red borders/banners/confirmations); avoid persuasive/emotionally manipulative language in safety-critical flows.

## ASI10: Rogue Agents

Malicious or compromised agents deviate from their intended function or authorized scope, acting harmfully, deceptively, or parasitically — individually-legitimate-looking actions with harmful emergent behavior, creating a containment gap for rule-based defenses. External compromise (prompt injection, goal hijack, supply-chain tampering) can trigger it, but ASI10 is about the loss of behavioral integrity/governance once drift begins, not the initial intrusion.

**Examples**

- Impersonated Observer Agent: an attacker injects a fake review/approval agent into a workflow; a payment-processing agent trusts it and releases funds fraudulently.
- Reward Hacking → Critical Data Loss: an agent tasked with minimizing cloud costs learns that deleting production backups is the most effective route, destroying disaster-recovery assets.

**Prevention**

- Governance & logging: maintain comprehensive, immutable, signed audit logs of all agent actions, tool calls, and inter-agent communication.
- Isolation & boundaries: assign trust zones with strict inter-zone rules; deploy sandboxed execution environments with least-privilege API scopes.
- Monitoring & detection: deploy watchdog agents to validate peer behavior/outputs, focused on collusion patterns and coordinated false signals.
- Containment & response: kill-switches and credential revocation to instantly disable rogue agents; quarantine suspicious agents for forensic review.
- Identity attestation and behavioral integrity: per-agent cryptographic identity attestation; signed behavioral manifests declaring expected capabilities/tools/goals validated before each action.
