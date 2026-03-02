# AI Coding Agent Guidelines — A basic framework

---

## The Context Hierarchy

AI coding agents don't just "see your code." They build understanding from multiple sources, each with a different scope and lifecycle. Understanding these layers is the foundation of everything else in this guide.

```markdown
Layer 1: System Instructions (always active) → .github/copilot-instructions.md, AGENTS.md, .cursorrules
         You set these once per repo. They shape every interaction.

Layer 2: Workspace Context (selectively used) → @workspace searches, indexed by embeddings
         The agent's "background knowledge" of your project.

Layer 3: Explicit Attachments (you control) → #file, #selection, #symbol references
         What you intentionally bring into the conversation.

Layer 4: Conversation History (accumulated) → Prior Q&A, decisions, corrections within the session
         Builds up as you work. Can help or pollute.

Layer 5: Tool Results (dynamic) → grep, semantic search, file reads, terminal output
         Fresh data the agent retrieves during execution.
```

Layers 1–2 are always there. Layers 3–5 are *intentional* you drive them. The more you engage Layers 3–5, the better the output, especially in legacy or complex environments where Layer 2 retrieval is less reliable.

---

## Narrowing Context: 7 Standards

The quality of AI output is directly proportional to the precision of the context you provide. 

| # | Standard | Rule | Why |
| --- | --- | --- | --- |
| S1 | **Scope declaration** | Every prompt must declare what files/modules are in scope | Prevents the model from pulling irrelevant context |
| S2 | **Negative constraints** | Explicitly state what NOT to change | "Do not modify public API signatures" prevents side-effect edits |
| S3 | **Reference over paste** | Use `#file:path/to/file` instead of pasting code | Keeps token count low, model sees fresh file state |
| S4 | **One concern per chat** | Start a new chat for a new topic/task. Use proper naming conventions for sessions | Prevents context pollution from unrelated history |
| S5 | **Instruction files** | Maintain `.github/copilot-instructions.md` per repository | Ensures consistent behavior without repeating context |
| S6 | **Scoped instructions** | Use scoped instruction files for domain-specific rules (e.g. `*.instructions.md` with `applyTo` in VS Code, or folder-level `.cursorrules` in Cursor) | Narrows model behavior per component |
| S7 | **Working set discipline** | In Edits/Agent mode, only add files you want the model to consider | Large working sets dilute attention and waste tokens |

---

## The Task Framing Pattern

Every complex prompt should follow this structure. It's the single most impactful practice in this guide.

```markdown
## Objective
[Your definition of done]

## Context
- Stack: [technologies, versions, build system]
- Starting point: [which files/modules to begin with]
- Related: [#file references or documentation links]

## Constraints
- Do NOT change [specific boundaries]
- Prefer [specific patterns/libraries/approaches]
- Must maintain [backward compatibility / API stability / etc.]

## Acceptance Criteria
- [ ] Tests pass
- [ ] No new lint warnings
- [ ] Follows team coding standards
- [ ] Documentation updated
```

This structure works across tools. The format is less important than the habit of declaring intent, context, boundaries and "done" before asking the agent to work.

---

## The Checkpoint Strategy

For multi-step work, never let an agent execute everything at once. Break execution into verified phases:

```
Phase 1: Research & Plan → Agent proposes a plan → You review and adjust

Phase 2: Core Implementation → Agent writes code → You run tests

Phase 3: Test & Validate → Agent adds tests → You verify coverage

Phase 4: Documentation → Agent updates docs → You review accuracy
```

**Never let Agent Mode execute more than one phase without your review.**

An agent that makes a wrong assumption in Phase 1 will confidently build on that assumption through Phase 4. Catching it early is cheaper than debugging it later.

---

## Prompt Patterns


Rxamples of reusable templates for common tasks. Adapt to your stack.

| Pattern | Template | When |
| --- | --- | --- |
| **Explain** | "Explain what `#file:src/engine.py` lines 120-180 do, focusing on [specific aspect]" | Understanding unfamiliar code |
| **Refactor** | "Refactor `#OrderProcessor` to separate validation from execution. Keep public interface stable." | Code improvement |
| **Debug** | "This test fails with [error]. The relevant code is `#file:src/handler.py`. Identify root cause and fix." | Bug fixing |
| **Generate** | "Generate a [artifact] for `#file:config.yaml` following the pattern in `#file:template.yaml`" | Code generation |
| **Review** | "Review `#file:src/api.py` for: naming conventions, error handling, SOLID principles." | Code review |
| **Migrate** | "Migrate `#file:legacy_module` from [old] to [new]. Preserve behavior. Add unit tests." | Modernization |

---

## Token Preservation

Tokens = cost + latency. Every unnecessary token consumed affects response quality, speed and budget.

| # | Rule | 
| --- | --- |
| T1 | **Reference, don't paste** — Use `#file` and `#symbol` instead of copy-pasting |
| T2 | **Scoped reads** — "Read lines 120-180" instead of whole file | 
| T3 | **Fresh chat for new topic** — Don't carry 20 turns of history into a new question | 
| T4 | **Instruction files over repetition** — Put standards in `.github/copilot-instructions.md` |
| T5 | **Precise prompts** — "Add null check to line 45 of [handler.py](http://handler.py/)" vs. "make it safer" | 
| T6 | **Filter tool output** — In agent mode: "show only failing tests" | 
| T7 | **Use .copilotignore** — Exclude build artifacts, generated files, vendored code | 
| T8 | **Minimize working set** — In Edits mode, only add files that need changes |
| T9 | **Batch related questions** — 3 related questions in one prompt, not 3 separate chats |
| T10 | **Exit when done** — Don't continue chatting "just to check" |

---

## Multi-Agent Design

Instead of one general-purpose agent, create **specialized agents** with narrow domains. Each agent gets only the context it needs.

### Instruction Levels

| Level | Mechanism | Scope | Example |
| --- | --- | --- | --- |
| **Global** | `.github/copilot-instructions.md` | All chats in the repo | "Always use camelCase. Target Python 3.11+. Follow PEP 8." |
| **Folder** | Scoped instruction files (e.g. `*.instructions.md` with `applyTo`) | Chats touching matching files | "This module handles payments. Validate all inputs." |
| **Agent** | Custom agent definitions (e.g. `.agent.md` in `.github/agents/`) | When agent is active | "You are a security review specialist." |
| **Prompt** | Reusable prompt files (e.g. `.prompt.md` in `.github/prompts/`) | Invoked per task via `/` | "Generate a React component following our patterns." |
| **Session** | User's opening prompt | Current conversation | Task framing pattern (see above) |

### Suggested agent personas examples

| Agent | Focus | Tools |
| --- | --- | --- |
| **Code Review** | Style, SOLID principles, test coverage, naming | search, read |
| **Debug** | Stack traces, logs, dependencies, root cause | search, read, run_in_terminal |
| **Documentation** | API docs, README, architecture decisions | search, read, fetch |
| **Security** | Vulnerabilities, input validation, secrets, dependencies | search, read |

### Agent Definition Template

Adapt to your tool. In VS Code, agents use `.agent.md` files in `.github/agents/` with tools like `search`, `fetch`, `read`, `edit`, `run_in_terminal`. Other platforms have their own conventions.
The structure below captures the intent:

```markdown
---
name: "[Agent name for picker]"
description: "[One-line description]"
tools: ["search", "read", "fetch"]
---

# [Agent Name]

## Your Role
You are a [specific role] specialized in [narrow domain].

## Core Capabilities
1. [Specific capability with boundaries]
2. [Specific capability with boundaries]

## What You Do NOT Do
- [Explicit exclusion to prevent scope creep]

## Interaction Pattern
1. [How the agent should start — what context to gather first]
2. [What analysis to perform]
3. [How to present findings]

## Output Standards
- [Format rule]
- [Quality rule]
```

---

## .copilotignore

Controls what gets excluded from **Layer 2** (workspace indexing). This does NOT affect Layers 3–5 — if you explicitly `#reference` an ignored file, the agent still sees it.

```
# Build artifacts
build/
dist/
*.o
*.obj

# Generated files
*_generated.*
*.pb.h
*.pb.cc

# Vendored / third-party
vendor/
third_party/
node_modules/

# Large data files
*.dat
*.bin
*.log
data/

# Secrets and environment
.env
.env.*
**/secrets/

# IDE artifacts
.vs/
.idea/
```

---

## Security Guardrails

AI agents read your code, your configs, your environment and they can leak what they learn. These guardrails are the minimum to enforce before any agent touches production-adjacent code.

### The Threat Model in 30 Seconds

Your AI coding agent is a **confusable deputy**: it cannot reliably separate "instructions" from "data" in the text it reads. This means:

- A malicious comment in a pulled dependency can become an instruction
- A crafted error message in a log can redirect agent behavior
- Your `.env` file, API keys, or internal URLs can surface in agent output
- An agent with write access to your terminal can execute destructive commands from a hallucinated plan

### Guard Rules

| # | Guard | Rule | Why |
| --- | --- | --- | --- |
| G1 | **Secrets exclusion** | Never paste tokens, API keys, passwords, or connection strings into chat. Use `#file` references to config *templates*, not actual `.env` files | Once in the context window, secrets can appear in completions, logs, and cached embeddings |
| G2 | **`.copilotignore` for secrets** | Add `.env`, `.env.*`, `**/secrets/`, `*.pem`, `*.key` to `.copilotignore` | Prevents secrets from entering Layer 2 (workspace indexing) automatically |
| G3 | **Read-only by default** | When using Agent Mode, restrict terminal access. No `rm -rf`, no `DROP`, no `kubectl delete` without human confirmation | An agent that hallucinates a plan can execute it. Least privilege is your blast-radius limiter |
| G4 | **No untrusted content in context** | Do not paste raw user input, external issue bodies, or unreviewed PR descriptions into agent prompts | Untrusted text can contain indirect prompt injections — instructions disguised as data |
| G5 | **Review before commit** | Agent-generated code must go through the same review process as human code. No exceptions | Agents write plausible code that compiles and passes basic tests but may introduce subtle vulnerabilities |
| G6 | **Scope tool permissions** | Each agent persona should only have access to the tools it needs. A docs agent doesn't need `run_in_terminal` | Excessive agency turns a model mistake into an operational incident |
| G7 | **Output validation** | Before shipping agent output, check for: leaked internal URLs, hardcoded IPs, embedded credentials, PII | The model may include context it shouldn't in its generated code or documentation |
| G8 | **Session isolation** | Don't mix security-sensitive work (infra config, IAM policies, secret rotation) with general coding in the same chat session | Conversation history (Layer 4) accumulates — sensitive context from turn 3 can leak into the completion at turn 15 |

### Practical `.copilotignore` for Security

On top of the `.copilotignore` described above, add these patterns to keep sensitive files out of workspace indexing:

```
# Certificates and keys (beyond .env already in general config)
*.pem
*.key
*.p12
*.pfx
**/credentials/

# Cloud infrastructure state
*.tfstate
*.tfstate.backup
.terraform/

# CI/CD secrets
.github/secrets/
.gitlab-ci-secrets/

# Kubernetes secrets
**/k8s/*secret*
**/helm/*values-prod*

# Database dumps and backups
*.sql
*.dump
*.bak
```

### What to Put in Instruction Files (and What Not To)

Your `.github/copilot-instructions.md` is committed to the repo. Treat it like code that anyone can read.

**Safe to include:**

- Coding standards, naming conventions, architectural patterns
- Technology stack and version constraints
- Test framework and coverage requirements
- Negative constraints ("do not use library X", "do not modify module Y")

**Never include information that cannot be shared**

### The Checkpoint Strategy (for security)

When an agent works on security-sensitive code:

```
Phase 0: Threat Assessment → Before the agent writes anything, declare:
           - What data is in scope
           - What must NOT be exposed
           - What destructive operations are forbidden

Phase 1: Plan Review → Agent proposes approach → You verify it doesn't introduce new attack surface

Phase 2: Implementation → Agent writes code → You review EVERY file diff, not just the summary

Phase 3: Security Validation
         → Run SAST/dependency scan on changes
         → Check for hardcoded secrets (git-secrets, gitleaks, trufflehog)
         → Verify no new permissions or network exposure

Phase 4: Peer Review
         → Security-sensitive agent code gets a human reviewer who wasn't in the loop. Fresh eyes catch what familiarity misses.
```

### MCP and External Tool Connections

If your agent connects to external services via MCP (Model Context Protocol) or similar:

- **Assume every tool call is visible.** The agent may send more context to an external service than you intended. Audit what gets sent.
- **Require explicit consent for write operations.** Read is cheap to undo; write is not.
- **Validate tool responses.** A compromised or malicious MCP server can inject instructions into tool results that redirect agent behavior (Toxic Agent Flow).
- **Short-lived credentials only.** 

---

## Model Selection

AI coding tools increasingly support multiple models. Each has different strengths. This is a general guide models evolve very fast, so validate against your own use cases.

| Scenario | Recommended Approach | Why |
| --- | --- | --- |
| Daily coding (completions, small edits) | Fastest available model | Speed matters most for flow |
| Complex multi-file refactoring | Strong instruction-following model (e.g. Claude Sonnet) | Needs to respect constraints across files |
| Architectural analysis | Deep reasoning model (e.g. Claude Opus) | Handles nuance and trade-offs |
| Debugging production issues | Methodical model with good context handling | Needs to consider edge cases |
| Writing documentation | Strong writing model | Structured output, clear prose |
| Security review | Most thorough model available | Must not miss subtle issues |
| Quick inline suggestions | Default/fastest model | Latency over depth |

---

## Working with Legacy Code

If your codebase has years (or decades) of history, generic prompting won't work. Be explicit about:

1. **Language standards** — "This is Python 2.7 with some Python 3 modules. Do not modernize Python 2 code unless asked."
2. **Build system** — "This builds with Maven 3.6. Dependencies are in pom.xml."
3. **Platform constraints** — "Target: RHEL 8, x86_64. No features requiring kernel 5.x+."
4. **Data formats** — "This processes [protocol/format] messages. Follow the specification strictly."
5. **Surrounding context** — Always provide the files around the code you're asking about, not just the target file.

The agent has no memory of your architecture. Every session, you need to reconstruct enough context for it to make safe changes. 

---

## Contributing

This is a living document. If you find something wrong, missing, or that could be better:

1. Open an issue describing the problem
2. Submit a PR with your improvement
3. Share your own instruction files or agent definitions

---

*Opinions are my own. For the thinking behind this guide, read my article: [The context your AI agent needs is not a file, but a lifecycle](https://www.notion.so/YOUR_MEDIUM_LINK).*
