# OWASP GenAI/LLM Top 10 (2026 v1.0)

Summary of: https://genai.owasp.org/resource/owasp-genai-llm-top-10-2026/

| # | Vulnerability | What it is |
| --- | --- | --- |
| LLM01 | [Prompt Injection](#llm01-prompt-injection) | Input (direct user input, or indirect via retrieved content/tool output/images/memory) alters model behavior in unintended ways. LLMs make no architectural distinction between instructions and data, so there's no clean equivalent to parameterized queries — defense is architectural (bound the blast radius), not interceptive. |
| LLM02 | [Sensitive Information Disclosure](#llm02-sensitive-information-disclosure) | An LLM-integrated system exposes confidential/regulated/proprietary data through any channel — not just the final answer, but tool-call args, reasoning traces, retrieved chunks, logs, embeddings, and inference side-channels (timing, token length, log-probs). Arises at training-time, inference-time, pipeline-time, and observation-time. |
| LLM03 | [Excessive Agency](#llm03-excessive-agency) | Damaging actions get performed in response to unexpected, ambiguous, or manipulated LLM output (from hallucination or prompt injection), rooted in excessive functionality, excessive permissions, or excessive autonomy granted to the agent's tools. |
| LLM04 | [Supply Chain](#llm04-supply-chain) | Training data, models, adapters, conversion/merge/quantization pipelines, and deployment platforms are compromised via tampering, poisoning, or malicious artifact replacement — classic software supply-chain risk extended to model artifacts, LoRA adapters, and on-device models. |
| LLM05 | [Data and Model Poisoning](#llm05-data-and-model-poisoning) | An adversary (or unsafe process) manipulates training/fine-tuning/embedding data or model artifacts to embed harmful behavior, bias, or backdoors — can occur at pre-training, fine-tuning, embedding creation, RAG ingestion, or via continuous-learning feedback loops. Unlike a code bug, fixing it requires data revalidation or retraining. |
| LLM06 | [Unbounded Consumption](#llm06-unbounded-consumption) | The app allows excessive/uncontrolled inference, letting attackers disrupt availability, inflict cost (denial of wallet), or steal IP via model cloning — driven by cost asymmetry between cheap attacker requests and expensive compute, amplified by reasoning models, multimodal inputs, and agentic tool-use fan-out. |
| LLM07 | [Misinformation](#llm07-misinformation) | An LLM or LLM-enabled app produces incorrect, incomplete, or misleading output that appears credible enough to be trusted and acted on by a human, workflow, or agent — a system-level failure once outputs drive tool calls, code, or cross-agent decisions. Root causes like prompt injection or poisoning are covered elsewhere; this is about the resulting false representation. |
| LLM08 | [Hidden Context Exposure](#llm08-hidden-context-exposure) | Unauthorized extraction, inference, or reconstruction of hidden, non-user-facing system instructions/context (system prompt, developer instructions, retrieved policy text, tool schemas). Security-relevant when it contains secrets, policy logic, or trust-boundary details — hidden context should never itself be relied on as a security boundary. |
| LLM09 | [Vector and Embedding Weaknesses](#llm09-vector-and-embedding-weaknesses) | Security risks specific to any system that converts content into embeddings and uses similarity search to decide what the model sees (RAG, agent memory, semantic caches) — exploits embedding-space geometry and similarity-search mechanics, not the model's instruction-following. Frame: poisoning makes it wrong, inversion makes it leak, jamming makes it silent, access-control failure makes it indiscriminate. |
| LLM10 | [Improper Output Handling](#llm10-improper-output-handling) | Insufficient validation, sanitization, and handling of LLM-generated output before it's passed downstream — output should be treated like untrusted user input, regardless of whether its content is itself correct. Can lead to XSS/CSRF in browsers, or SSRF/privilege escalation/RCE on backend systems. |

---

## LLM01: Prompt Injection

Input (direct user input, or indirect via retrieved content/tool output/images/memory) alters model behavior in unintended ways. LLMs make no architectural distinction between instructions and data, so there's no clean equivalent to parameterized queries — defense is architectural (bound the blast radius), not interceptive.

**Examples**

- Direct injection: attacker prompts a customer-support chatbot to ignore its guidelines, query private data stores, and send emails.
- Indirect via retrieved web content: a page's hidden instructions make the model insert a markdown image whose URL exfiltrates the private conversation to an attacker-controlled domain; user only sees the rendered image.

**Prevention**

- Constrain the model's role/capabilities in the system prompt (allow/deny statements) — partial control only, pair with the rest of these controls.
- Define a strict output schema; validate every response in trusted app code before any downstream action.
- Hold credentials and state-change capability in application code, not the model; least privilege per operation via a deterministic policy engine that re-validates at execution time.
- Pass external content through a structurally separate, provenance-labeled channel so the model can distinguish data from instructions.
- Require explicit human confirmation before privileged/irreversible/externally-visible actions, showing the exact rendered action (not a summary).

## LLM02: Sensitive Information Disclosure

An LLM-integrated system exposes confidential/regulated/proprietary data through any channel — not just the final answer, but tool-call args, reasoning traces, retrieved chunks, logs, embeddings, and inference side-channels (timing, token length, log-probs). Arises at training-time, inference-time, pipeline-time, and observation-time.

**Examples**

- Training-data extraction: the "poem" divergence attack made `gpt-3.5-turbo` emit 10,000+ unique memorized training examples for ~$200.
- The March 2023 ChatGPT Redis bug exposed payment PII for 1.2% of Plus subscribers.

**Prevention**

- **Baseline (every deployment):** govern corpora (provenance, classification, scrub PII at ingest); minimize context sent to providers; authorize before retrieval (doc/chunk-level, not a post-filter); never store secrets in system prompts; budget queries per user/session; restrict/scrub logs, encrypt in transit and at rest, enforce no-train/no-retain technically.
- **Regulated/high-sensitivity:** encrypt and ACL the vector store; gate log-probabilities/confidence on prod endpoints; classify and redact reasoning traces; confidential computing; verifiable erasure (unlearning); disclosure red-teaming as a release gate.

## LLM03: Excessive Agency

Damaging actions get performed in response to unexpected, ambiguous, or manipulated LLM output (from hallucination or prompt injection), rooted in excessive functionality, excessive permissions, or excessive autonomy granted to the agent's tools.

**Examples**

- A DB tool meant only to read data connects with an identity that also has UPDATE/INSERT/DELETE permissions.
- Hijacked Email Assistant: a mail-summarizing tool also has send capability; an indirect prompt injection via a malicious incoming email tricks the agent into scanning the inbox and forwarding sensitive content to the attacker.

**Prevention**

- Minimize tools — only offer what's strictly necessary.
- Minimize tool permissions to the minimum needed on downstream systems, enforced via real DB/API-level permissions.
- Execute tools in the user's own auth context (OAuth with minimum scope), preserving that context across chained/multi-agent calls.
- Require human approval (human-in-the-loop) for high-impact actions.
- Complete mediation: enforce authorization in deterministic logic (tool, policy decision point, or downstream system), not LLM judgment.

## LLM04: Supply Chain

Training data, models, adapters, conversion/merge/quantization pipelines, and deployment platforms are compromised via tampering, poisoning, or malicious artifact replacement — classic software supply-chain risk extended to model artifacts, LoRA adapters, and on-device models.

**Examples**

- A malicious `torchtriton` PyPI package shadowed the legitimate PyTorch-nightly dependency and exfiltrated data (Dec 2022).
- PoisonGPT: a model published under a trusted-looking name with surgically modified parameters spread misinformation while evading standard benchmark detection.

**Prevention**

- Vet data/model suppliers (T&Cs, privacy policies, security posture); re-assess on changes.
- Apply classic component-management controls (scanning, patching, maintained versions); verify AI-suggested dependencies actually exist before adopting them.
- Maintain a signed, up-to-date SBOM extended to models/adapters/datasets (AIBOM/ML-BOM, e.g. CycloneDX ML-BOM).
- Use only verifiable-source models; cryptographic signing backed by a transparency log (Sigstore/OpenSSF Model Signing), immutable digest references (not mutable "latest" tags).
- Strictly monitor/audit collaborative model-dev environments; treat conversion/merge services as high-risk promotion points.

## LLM05: Data and Model Poisoning

An adversary (or unsafe process) manipulates training/fine-tuning/embedding data or model artifacts to embed harmful behavior, bias, or backdoors — can occur at pre-training, fine-tuning, embedding creation, RAG ingestion, or via continuous-learning feedback loops. Unlike a code bug, fixing it requires data revalidation or retraining.

**Examples**

- As few as 250 poisoned documents compromise models from 600M to 13B parameters, regardless of total dataset size.
- Persistent-memory poisoning: an attacker injects malicious instructions into an agent's memory over multiple sessions, causing long-term workflow manipulation.

**Prevention**

- Track dataset/model lineage via SBOM/ML-BOM, enforce signing and verification.
- Strict validation of all incoming data; vet third-party vendors; compare outputs against trusted sources.
- Protect RAG: enforce trust boundaries, filter retrieved content, apply source scoring, isolate system instructions from external data.
- Statistical/AI anomaly detection across training, embedding, and inference pipelines; monitor loss/output/behavior for drift.
- Continuously red-team for hidden backdoors/triggers — safety alignment does not reliably remove them.

## LLM06: Unbounded Consumption

The app allows excessive/uncontrolled inference, letting attackers disrupt availability, inflict cost (denial of wallet), or steal IP via model cloning — driven by cost asymmetry between cheap attacker requests and expensive compute, amplified by reasoning models, multimodal inputs, and agentic tool-use fan-out.

**Examples**

- Denial of Wallet (DoW): a high volume of operations exploits pay-per-use cloud AI pricing to inflict unsustainable cost.
- Growing agentic-session context: per-turn cost climbs from ~$0.001 (turn 1) to ~$0.50 (turn 100) as the full accumulated context is re-processed each turn — no single request trips rate limits, but the aggregate across sessions reaches hundreds of dollars.

**Prevention**

- Rate limiting + token-aware quotas (tokens/minute, tokens/day, estimated cost per request); pre-flight token estimation to reject before inference begins.
- Hard, non-overridable spending caps per API key/user/team/cloud account — enforcement, not just alerting.
- Sandbox the LLM's network/API access to limit exfiltration of extracted data.
- Monitor agent-tool interactions against a normal-behavior baseline to catch recursive/resource-intensive patterns.
- Agentic circuit breakers: step limits, recursion-depth limits, time limits, per-run cost ceilings, state hashing to detect loops.

## LLM07: Misinformation

An LLM or LLM-enabled app produces incorrect, incomplete, or misleading output that appears credible enough to be trusted and acted on by a human, workflow, or agent — a system-level failure once outputs drive tool calls, code, or cross-agent decisions. Root causes like prompt injection or poisoning are covered elsewhere; this is about the resulting false representation.

**Examples**

- Hallucinated dependency: a coding assistant recommends a plausible but nonexistent package that an attacker has pre-registered with malicious code.
- Cross-agent trust failure: a retrieval agent wrongly reports a customer as identity-verified, and a downstream payment agent trusts that state and releases funds.

**Prevention**

- Ground claims in authoritative, current sources before acting.
- Claim-check-act pattern: separate generation from execution, verify claims before acting.
- Validate tool-call arguments, authorization, preconditions, and current state before execution.
- Require approval workflows and system checks for high-impact actions.
- Limit blast radius with least privilege, sandboxing, and rate limits.

## LLM08: Hidden Context Exposure

Unauthorized extraction, inference, or reconstruction of hidden, non-user-facing system instructions/context (system prompt, developer instructions, retrieved policy text, tool schemas). Security-relevant when it contains secrets, policy logic, or trust-boundary details — hidden context should never itself be relied on as a security boundary.

**Examples**

- A system prompt contains credentials for a tool it has access to; the leaked prompt lets an attacker reuse those credentials elsewhere.
- Conversational probing extracts the hidden tool list and parameter schemas, giving the attacker concrete targets for follow-on prompt injection — no credential leaked, no policy overtly bypassed, but reconnaissance is complete.

**Prevention**

- Never put secrets, credentials, or security-critical config in the system prompt or hidden context — assume everything in context is discoverable by users.
- Enforce critical behaviors (content filtering, safety) via deterministic external guardrails, not instructions embedded in the prompt.
- Enforce authorization, privilege separation, and policy decisions independently of the LLM, in deterministic and auditable systems — never delegate them to the model.

## LLM09: Vector and Embedding Weaknesses

Security risks specific to any system that converts content into embeddings and uses similarity search to decide what the model sees (RAG, agent memory, semantic caches) — exploits embedding-space geometry and similarity-search mechanics, not the model's instruction-following. Frame: poisoning makes it wrong, inversion makes it leak, jamming makes it silent, access-control failure makes it indiscriminate.

**Examples**

- Cross-tenant leakage: similarity search runs across the full index before per-app access control is applied, letting an attacker infer other tenants' document existence/topic/volume from result counts and timing — works even with correct tagging (compounded by real CVEs like Milvus CVE-2025-64513 and RAGFlow CVE-2025-69286).
- Embedding inversion: Vec2Text recovers ~50–70% of words from sentence embeddings, up to 92% exact reconstruction of short 32-token inputs; vector-DB backups/leaks should be treated as source-document leaks.

**Prevention**

- Enforce tenant scoping inside the index query itself (server-side, chunk-level), not as a post-retrieval filter; physically separate indexes for high-sensitivity workloads.
- Normalize content before embedding (strip zero-width characters/homoglyphs); track provenance per embedding; vet the embedding model itself.
- Segregate mixed-trust content by index, not just classification tags on a shared index.
- Anomaly detection at ingest/retrieval: flag vectors unusually close to many queries, don't return raw similarity scores to clients, rate-limit.
- Bound embedding storage lifecycle: delete embeddings on source-document deletion, encrypt at rest, re-embed the full corpus on model rotation.

## LLM10: Improper Output Handling

Insufficient validation, sanitization, and handling of LLM-generated output before it's passed downstream — output should be treated like untrusted user input, regardless of whether its content is itself correct. Can lead to XSS/CSRF in browsers, or SSRF/privilege escalation/RCE on backend systems.

**Examples**

- LLM output passed directly to a shell/`exec`/`eval` → remote code execution.
- A chat UI auto-renders a Markdown image or link preview referenced in model output → exfiltrates conversation data via the image URL's hostname/query string.

**Prevention**

- Treat model output as untrusted input (zero-trust) with proper validation before it reaches backend functions.
- Apply context-aware output encoding based on the sink (HTML, JavaScript, SQL, etc.).
- Use parameterized queries/prepared statements for any DB operation involving LLM output.
- Employ a strict Content Security Policy to blunt XSS from LLM-generated content.
- In client renderers, disable auto-fetch of external resources (Markdown images, link previews, iframes) by default, or restrict to an allowlist / proxy through a stripping server-side fetcher.
