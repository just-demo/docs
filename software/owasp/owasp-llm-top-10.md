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
- Trusted-surface indirect injection via MCP: attacker plants text in a low-privilege channel (GitHub issue, support ticket, npm package) and the user's own agent reads it under elevated credentials — Invariant Labs (2025) exfiltrated private repos via a poisoned GitHub issue, General Analysis (2025) dumped a production DB through Cursor's Supabase MCP server, malicious `postmark-mcp` package BCC'd email to an attacker across ~300 orgs.
- RAG corpus poisoning: as few as 5 poisoned documents reached ~90% attack success against a knowledge base of millions of texts.
- Multimodal/steganographic: instructions embedded in an image below the human visual threshold are extracted by the vision encoder and change model behavior.
- Invisible-Unicode exfiltration: zero-width/variation-selector characters smuggle instructions or exfiltrate bytes in benign-looking text (2024 M365 Copilot ASCII-smuggling PoC exfiltrated a Slack MFA code).
- Agentic destructive execution: a destructive system prompt was committed to the Amazon Q VS Code extension repo (AWS reverted it); separately a runtime injection caused Amazon Q to execute arbitrary code.

**Prevention**

- Constrain the model's role/capabilities in the system prompt (allow/deny statements) — partial control only, pair with #4.
- Define a strict output schema; validate every response in trusted app code before any downstream action.
- Filter at every modality boundary (text, image, audio, structured data), not just text.
- Hold credentials and state-change capability in application code, not the model; least privilege per operation via a deterministic policy engine that re-validates at execution time.
- Strip invisible Unicode (tag-block, variation-selector, zero-width chars) at every ingest/render boundary.
- Pass external content through a structurally separate, provenance-labeled channel so the model can distinguish data from instructions.
- Require explicit human confirmation before privileged/irreversible/externally-visible actions, showing the exact rendered action (not a summary).
- Budget agent capabilities with the "Rule of Two": simultaneous untrusted input + sensitive data + state change/external comms = high risk, needs per-action human approval.
- Treat agent memory writes as privileged operations — log, classify, and require approval before instruction-bearing memories persist across sessions.
- Pin, sign, and verify MCP servers/third-party tool packages; audit tool descriptions for hidden instructions.
- Test against adaptive attackers with the full defense disclosed — static-only testing is unreliable (static success near zero vs. >90% adaptive success in one study).

## LLM02: Sensitive Information Disclosure

An LLM-integrated system exposes confidential/regulated/proprietary data through any channel — not just the final answer, but tool-call args, reasoning traces, retrieved chunks, logs, embeddings, and inference side-channels (timing, token length, log-probs). Arises at training-time, inference-time, pipeline-time, and observation-time.

**Examples**

- Training-data extraction: the "poem" divergence attack made `gpt-3.5-turbo` emit 10,000+ unique memorized training examples for ~$200.
- The March 2023 ChatGPT Redis bug exposed payment PII for 1.2% of Plus subscribers.
- 4,500+ shared ChatGPT conversations were indexed by Google in 2025 via missing `noindex` directives.
- Prompt injection makes a support bot print its system prompt and an embedded vendor API key.
- Whisper Leak: topic inference from encrypted streaming traffic at >98% AUPRC across 28 production models, with no decryption.
- A leaked "embeddings-only" vector backup gets reclassified as a source-document breach after an inversion attack recovers the underlying content.
- DeepSeek's January 2025 ClickHouse exposure leaked 1M+ rows of logs and API keys.

**Prevention**

- **Tier 1 (every deployment):** govern corpora (provenance, classification, dedup; scrub PII at ingest); minimize context sent to providers; authorize before retrieval (enforce doc/chunk-level auth inside the index query, not after); never store secrets in system prompts; sanitize with classifiers, not regex alone; budget queries per user/session; restrict/scrub logs, encrypt in transit and at rest, enforce no-train/no-retain technically.
- **Tier 2 (regulated/high-sensitivity):** DP-SGD training; vector-store protection (encryption, ACLs, restricted export); gate log-probabilities/confidence on prod endpoints; classify and redact reasoning traces as first-class output; side-channel defenses (padding, batching, cache segregation); format-preserving encryption for identifiers; AI-aware audit logging into SIEM + continuous DLP.
- **Tier 3 (regulated/classified/high-target):** confidential computing; verifiable erasure (unlearning) validated by extraction/membership-inference probes; disclosure red-teaming as a release gate; audit synthetic data and resist distillation; documented disclosure incident-response playbook aligned to GDPR/HIPAA/EU AI Act.

## LLM03: Excessive Agency

Damaging actions get performed in response to unexpected, ambiguous, or manipulated LLM output (from hallucination or prompt injection), rooted in excessive functionality, excessive permissions, or excessive autonomy granted to the agent's tools.

**Examples**

- A tool grants read access to documents but also includes delete/modify functionality that isn't needed.
- A tool trialed in development is dropped for a better alternative but remains available to the agent.
- An open-ended shell-command tool fails to restrict which commands can be run.
- A DB tool meant only to read data connects with an identity that also has UPDATE/INSERT/DELETE permissions.
- A tool meant to act for "the current user" connects with a generic high-privileged identity that can access all users' files.
- A document-deletion tool performs deletions with no user confirmation.
- Hijacked Email Assistant: a mail-summarizing tool also has send capability; an indirect prompt injection via a malicious incoming email tricks the agent into scanning the inbox and forwarding sensitive content to the attacker.

**Prevention**

- Minimize tools — only offer what's strictly necessary.
- Minimize tool functionality (a mail-reading tool shouldn't also send/delete).
- Avoid open-ended tools (run-shell, fetch-URL); prefer granular tools with a strict, validated input schema.
- Minimize tool permissions to the minimum needed on downstream systems, enforced via real DB/API-level permissions.
- Execute tools in the user's own auth context (OAuth with minimum scope), preserving that context across chained/multi-agent calls.
- Require human approval (human-in-the-loop) for high-impact actions.
- Complete mediation: enforce authorization in deterministic logic (tool, policy decision point, or downstream system), not LLM judgment — graduated auto-approve vs. escalate policy for low- vs. high-consequence actions.
- Monitor tool use and downstream activity.
- Rate-limit tool invocations with circuit breakers on thresholds.

## LLM04: Supply Chain

Training data, models, adapters, conversion/merge/quantization pipelines, and deployment platforms are compromised via tampering, poisoning, or malicious artifact replacement — classic software supply-chain risk extended to model artifacts, LoRA adapters, and on-device models.

**Examples**

- A malicious `torchtriton` PyPI package shadowed the legitimate PyTorch-nightly dependency and exfiltrated data (Dec 2022).
- PoisonGPT: a model published under a trusted-looking name with surgically modified parameters spread misinformation while evading standard benchmark detection.
- A compromised supplier LoRA adapter is subtly altered and merged into a deployed model via a model-merge workflow.
- HiddenLayer hijacked the Safetensors conversion bot on Hugging Face to inject malicious behavior during format conversion.
- Model namespace reuse: an author's account/namespace is freed and re-registered by an attacker; pipelines that resolve models by name (not digest) pull the malicious model → RCE.
- Scanner/safe-loader bypass: corrupted pickle streams execute their payload before the scanner reaches the broken byte; ONNX computational-graph backdoors attach no executable code for scanners to flag.
- Ultralytics attack: GitHub Actions cache injection published trojanized PyPI releases of a flagship AI library, including a compromised "fix" release.
- Ollama CVE-2024-37032: RCE via a malicious model manifest pulled from a registry.
- "Slopsquatting": coding assistants hallucinate plausible but nonexistent package names, which attackers pre-register with malicious code.

**Prevention**

- Vet data/model suppliers (T&Cs, privacy policies, security posture); re-assess on changes.
- Apply classic component-management controls (scanning, patching, maintained versions); verify AI-suggested dependencies actually exist before adopting them.
- AI red-team third-party models before adoption, plus continuous anomaly detection and adversarial-robustness testing in production.
- Maintain a signed, up-to-date SBOM extended to models/adapters/datasets (AIBOM/ML-BOM, e.g. CycloneDX ML-BOM); track licenses in the same inventory.
- Use only verifiable-source models; cryptographic signing backed by a transparency log (Sigstore/OpenSSF Model Signing), immutable digest references (not mutable "latest" tags), SLSA-style policy release gates, plus behavioral evaluation — signing proves origin, not safety.
- Strictly monitor/audit collaborative model-dev environments; treat conversion/merge services as high-risk promotion points.
- Encrypt edge-deployed models with integrity checks, use vendor attestation APIs, reject unrecognized firmware/untrusted device states.

## LLM05: Data and Model Poisoning

An adversary (or unsafe process) manipulates training/fine-tuning/embedding data or model artifacts to embed harmful behavior, bias, or backdoors — can occur at pre-training, fine-tuning, embedding creation, RAG ingestion, or via continuous-learning feedback loops. Unlike a code bug, fixing it requires data revalidation or retraining.

**Examples**

- Attackers inject biased/malicious content into training data, deliberately eroding refusal behavior while preserving general accuracy (undetectable by standard evaluation).
- Fraud-detection model poisoned with mislabeled transactions (fraud labeled legitimate), so it learns to ignore real fraud.
- As few as 250 poisoned documents compromise models from 600M to 13B parameters, regardless of total dataset size.
- RAG knowledge-base poisoning: a single optimized poisoned text per targeted query overrides accurate content and resists paraphrasing/detection defenses.
- Chat-template/tokenizer backdoor (e.g. a GGUF package) with trigger-activated instructions — validated across 18 models/4 runtimes: factual accuracy drops from 90% to 15% under trigger, malicious URL emission exceeds 80% success.
- Malicious pre-trained weights uploaded to a public repo; standard safety training fails to remove the embedded backdoor.
- Persistent-memory poisoning: an attacker injects malicious instructions into an agent's memory over multiple sessions, causing long-term workflow manipulation.

**Prevention**

- Track dataset/model lineage via SBOM/ML-BOM, enforce signing and verification.
- Strict validation of all incoming data; vet third-party vendors; compare outputs against trusted sources.
- Protect RAG: enforce trust boundaries, filter retrieved content, apply source scoring, isolate system instructions from external data.
- Sandbox/isolate the model from unverified data, tools, or external systems.
- Statistical/AI anomaly detection across training, embedding, and inference pipelines; monitor loss/output/behavior for drift.
- Use curated, domain-specific datasets for fine-tuning.
- Least-privilege access and network segmentation against unauthorized data injection.
- Data version control (e.g. DVC) for rollback and forensic analysis.
- Control automated retraining/feedback loops: validate incoming data, require human oversight, rate-limit against gradual poisoning.
- Continuously red-team for hidden backdoors/triggers — safety alignment does not reliably remove them.
- Validate retrieved content before it's allowed to influence outputs.
- Treat chat templates, tokenizer configs, LoRA/PEFT adapters, and quantization artifacts as security-relevant code (sign, hash-verify, diff, static-analyze).

## LLM06: Unbounded Consumption

The app allows excessive/uncontrolled inference, letting attackers disrupt availability, inflict cost (denial of wallet), or steal IP via model cloning — driven by cost asymmetry between cheap attacker requests and expensive compute, amplified by reasoning models, multimodal inputs, and agentic tool-use fan-out.

**Examples**

- Variable-length input flood / output explosion: many varying-length inputs exploit processing inefficiencies, or a poisoned fine-tuning sample breaks end-of-sequence behavior, pushing every output to max length.
- Denial of Wallet (DoW): a high volume of operations exploits pay-per-use cloud AI pricing to inflict unsustainable cost.
- Reasoning-loop exhaustion: short, benign-looking prompts force extended-thinking models into prolonged/non-terminating reasoning, burning thinking-token budgets while bypassing input-size filters.
- "Sponge examples" — adversarial inputs optimized specifically to maximize compute cost.
- Model extraction: querying the API with crafted inputs to replicate a partial model or distill a functional equivalent; exposed logits/log-probabilities accelerate this.
- A malicious tool (e.g. published as an open-source Claude Skill) instructs an agent into recursive/cyclical tool calls, driving excessive token consumption and service instability.
- Growing agentic-session context: per-turn cost climbs from ~$0.001 (turn 1) to ~$0.50 (turn 100) as the full accumulated context is re-processed each turn — no single request trips rate limits, but the aggregate across sessions reaches hundreds of dollars.

**Prevention**

- Rate limiting + token-aware quotas (tokens/minute, tokens/day, estimated cost per request); pre-flight token estimation to reject before inference begins.
- Hard, non-overridable spending caps per API key/user/team/cloud account — enforcement, not just alerting.
- Dynamic resource-allocation management so no single request dominates.
- Sandbox the LLM's network/API access to limit exfiltration of extracted data.
- Design for graceful degradation under heavy load.
- Limit queued/total actions; scale and load-balance dynamically.
- Scan visual inputs (LVLMs) for adversarial perturbations.
- Monitor agent-tool interactions against a normal-behavior baseline to catch recursive/resource-intensive patterns.
- Agentic circuit breakers: step limits, recursion-depth limits, time limits, per-run cost ceilings, state hashing to detect loops.
- Harden inference infrastructure: keep serving frameworks patched, disable unsafe deserialization, restrict special-token passthrough, authenticate all endpoints.

## LLM07: Misinformation

An LLM or LLM-enabled app produces incorrect, incomplete, or misleading output that appears credible enough to be trusted and acted on by a human, workflow, or agent — a system-level failure once outputs drive tool calls, code, or cross-agent decisions. Root causes like prompt injection or poisoning are covered elsewhere; this is about the resulting false representation.

**Examples**

- Hallucinated dependency: a coding assistant recommends a plausible but nonexistent package that an attacker has pre-registered with malicious code.
- A customer-service agent misreads policy and approves a refund that violates the terms, causing financial loss.
- A clinical summary omits a drug contraindication and a clinician acts on the incomplete recommendation.
- An attacker seeds a support forum with false remediation steps that a troubleshooting agent retrieves and repeats as a trusted recommendation.
- A security agent misclassifies normal traffic as an intrusion and automatically blocks a production network segment, causing an outage.
- Cross-agent trust failure: a retrieval agent wrongly reports a customer as identity-verified, and a downstream payment agent trusts that state and releases funds.
- An agent reports that a nightly database backup completed when it never ran; a later restore fails because no backup exists.

**Prevention**

- Ground claims in authoritative, current sources before acting.
- Claim-check-act pattern: separate generation from execution, verify claims before acting.
- Validate tool-call arguments, authorization, preconditions, and current state before execution.
- Use verification signals (groundedness/consistency checks), not just model confidence.
- Require approval workflows and system checks for high-impact actions.
- Require structured outputs with mandatory fields to catch silent omissions.
- Limit blast radius with least privilege, sandboxing, and rate limits.
- Log claims, evidence, and outcomes; test adversarial scenarios.
- Calibrate human/system trust — distinguish verified facts from assumptions.
- Continuous adversarial evaluation against misleading scenarios.

## LLM08: Hidden Context Exposure

Unauthorized extraction, inference, or reconstruction of hidden, non-user-facing system instructions/context (system prompt, developer instructions, retrieved policy text, tool schemas). Security-relevant when it contains secrets, policy logic, or trust-boundary details — hidden context should never itself be relied on as a security boundary.

**Examples**

- A system prompt contains credentials for a tool it has access to; the leaked prompt lets an attacker reuse those credentials elsewhere.
- Conversational probing extracts the hidden tool list and parameter schemas, giving the attacker concrete targets for follow-on prompt injection — no credential leaked, no policy overtly bypassed, but reconnaissance is complete.
- A leaked system prompt reveals the exact conditions/exceptions behind refusal behavior, letting an attacker craft inputs that dodge known refusal patterns.
- A leaked prompt reveals role/permission requirements for a tool (e.g. "requires the developer role"), inviting further probing.
- Leaked output-formatting/JSON-schema rules let an attacker craft responses that pass downstream validation while embedding manipulated values.

**Prevention**

- Never put secrets, credentials, or security-critical config in the system prompt or hidden context — assume everything in context is discoverable by users.
- Enforce critical behaviors (content filtering, safety) via deterministic external guardrails, not instructions embedded in the prompt.
- Enforce authorization, privilege separation, and policy decisions independently of the LLM, in deterministic and auditable systems — never delegate them to the model.

## LLM09: Vector and Embedding Weaknesses

Security risks specific to any system that converts content into embeddings and uses similarity search to decide what the model sees (RAG, agent memory, semantic caches) — exploits embedding-space geometry and similarity-search mechanics, not the model's instruction-following. Frame: poisoning makes it wrong, inversion makes it leak, jamming makes it silent, access-control failure makes it indiscriminate.

**Examples**

- Cross-tenant leakage: similarity search runs across the full index before per-app access control is applied, letting an attacker infer other tenants' document existence/topic/volume from result counts and timing — works even with correct tagging (compounded by real CVEs like Milvus CVE-2025-64513 and RAGFlow CVE-2025-69286).
- Embedding inversion: Vec2Text recovers ~50–70% of words from sentence embeddings, up to 92% exact reconstruction of short 32-token inputs; newer zero-shot methods work cross-domain/black-box and survive differential-privacy noise — vector-DB backups/leaks should be treated as source-document leaks.
- Retrieval-time poisoning: attacker-crafted content is embedded close to a target query so it's retrieved as trusted context; a handful of poisoned documents succeed even in multi-million-document corpora.
- Retrieval jamming: a "blocker" document engineered to be retrieved for a specific query causes the LLM to refuse or claim ignorance — an availability attack with no malicious instructions involved.
- Membership inference: raw similarity scores returned to clients turn the index into a direct oracle for whether a specific document exists.
- Semantic cache/dedup poisoning: content crafted to land just above/below a cosine-similarity threshold poisons a cache entry served to all semantically-equivalent queries, or causes legitimate new content to be silently dropped as a duplicate.
- Multimodal embedding poisoning: an attacker-crafted image (via CLIP/ColPali-style cross-modal encoders) embeds close to a sensitive text query and gets retrieved as trusted context; a single image can suffice.

**Prevention**

- Enforce tenant scoping inside the index query itself (server-side, chunk-level), not as a post-retrieval filter; authenticate embedding/similarity endpoints with per-tenant rate limits; physically separate indexes for high-sensitivity workloads.
- Normalize content before embedding (strip zero-width characters/homoglyphs); track provenance per embedding; human-review externally-sourced content; vet the embedding model itself.
- Segregate mixed-trust content by index, not just classification tags on a shared index.
- Anomaly detection at ingest/retrieval: flag vectors unusually close to many queries, don't return raw similarity scores to clients, add noise/diversification, rate-limit, use cross-encoder re-ranking.
- Bound embedding storage lifecycle: delete embeddings on source-document deletion, encrypt at rest with separately managed keys, re-embed the full corpus on model rotation, treat embedding-API keys as secrets.
- Keep immutable retrieval logs, monitor for tenant-bypass attempts, and treat "embeddings-only" leaks as source-data breaches for incident-response/notification purposes.

## LLM10: Improper Output Handling

Insufficient validation, sanitization, and handling of LLM-generated output before it's passed downstream — output should be treated like untrusted user input, regardless of whether its content is itself correct. Can lead to XSS/CSRF in browsers, or SSRF/privilege escalation/RCE on backend systems.

**Examples**

- LLM output passed directly to a shell/`exec`/`eval` → remote code execution.
- LLM-generated JavaScript or Markdown is returned to and rendered by the browser → XSS.
- LLM-generated SQL is executed without parameterization → SQL injection (an unscrutinized chat-driven query feature could `DROP` every table).
- LLM output is used to construct file paths without sanitization → path traversal.
- LLM-generated content is used in email templates without escaping → phishing.
- ANSI escape sequences/control characters in LLM output are rendered by a terminal, log viewer, or IDE → visual spoofing, clipboard hijacking (OSC 52), terminal-emulator exploits.
- A chat UI auto-renders a Markdown image or link preview referenced in model output → exfiltrates conversation data via the image URL's hostname/query string.

**Prevention**

- Treat model output as untrusted input (zero-trust) with proper validation before it reaches backend functions.
- Follow OWASP ASVS guidance on input validation and sanitization.
- Encode output back to users to prevent unintended JS/Markdown execution.
- Apply context-aware output encoding based on the sink (HTML, JavaScript, SQL, etc.).
- Use parameterized queries/prepared statements for any DB operation involving LLM output.
- Employ a strict Content Security Policy to blunt XSS from LLM-generated content.
- Log and monitor for unusual output patterns indicating exploitation attempts.
- Sanitize control characters (ANSI escapes, BEL, OSC, backspace, carriage return) before writing output to terminals/logs; encode them visibly if they must be preserved.
- In client renderers, disable auto-fetch of external resources (Markdown images, link previews, iframes) by default, or restrict to an allowlist / proxy through a stripping server-side fetcher.
