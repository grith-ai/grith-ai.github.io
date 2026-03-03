---
layout: post
title: We Audited the Security of 7 Open-Source AI Agents  - Here Is What We Found
date: 2026-02-23
canonical_url: "https://grith.ai/blog/security-audit-seven-ai-agents"
description: A comparative teardown of the sandbox, permissions model, and untrusted input handling in OpenClaw, Claude Code, Codex, Cursor, Cline, Aider, and Open Interpreter. Real CVEs, real attack chains.
tags:
  - security
  - research
  - comparison
  - AI-agents
---


<figure>
  <a href="https://grith.ai/blog/security-audit-seven-ai-agents/hero-audit-scorecard-1600x900.png" target="_blank"><img src="https://grith.ai/blog/security-audit-seven-ai-agents/hero-audit-scorecard-1600x900.png" alt="Security audit scorecard for seven AI agents: OpenClaw, Claude Code, Codex, Cursor, Cline, Aider, and Open Interpreter, rated across sandbox, permissions, and untrusted input handling" /></a>
  <figcaption>Seven agents, three security dimensions. Most fail on at least two.</figcaption>
</figure>

Every AI coding agent needs the same three things: file access, shell execution, and network requests. The question is whether anything evaluates those operations before they execute.

We looked at seven open-source or widely-used AI agents and assessed each on three dimensions: **what sandbox** (if any) isolates the agent from the host system, **what permissions model** gates dangerous operations, and **what happens when the agent processes untrusted input**  - a poisoned README, a malicious plugin, a crafted email.

The results are not encouraging.

## The audit criteria

For each tool, we evaluated:

| Dimension            | What we looked for                                                                                |
| -------------------- | ------------------------------------------------------------------------------------------------- |
| **Sandbox**          | OS-level isolation (containers, seccomp, Seatbelt), filesystem boundaries, network restrictions   |
| **Permissions**      | How the tool decides what requires approval  - per-operation, per-session, or not at all            |
| **Untrusted input**  | What happens when the agent reads a file, plugin, or message containing a prompt injection         |

We referenced published CVEs, vendor security advisories, and independent research from Mindgard, HiddenLayer, Embrace The Red, and the IDEsaster project.

## 1. OpenClaw

**Sandbox:** None. OpenClaw runs with full system access  - terminal, filesystem, network, and application control. No container, no seccomp, no filesystem boundary.

**Permissions:** Localhost trusted by default with no authentication. Deployments behind a reverse proxy (nginx, Caddy) forward all traffic as `127.0.0.1`, so external requests bypass authentication entirely. CVE-2026-25253 (CVSS 8.8) allowed full operator access via a crafted WebSocket handshake with an omitted `scopes` field[^1].

**Untrusted input:** A Shodan scan found ~1,000 exposed instances with zero auth. Researchers accessed API keys, Telegram tokens, Slack accounts, and full chat histories[^2]. The ClawHavoc campaign poisoned 12% of ClawHub's plugin registry (341 of 2,857 skills) with stealer malware[^3]. A security audit found 512 vulnerabilities, 8 critical[^2].

**Verdict:** No security architecture exists. Everything is trusted, nothing is evaluated.

## 2. Claude Code

**Sandbox:** None by default. Claude Code runs as a CLI tool with the user's full filesystem and shell access. An optional Docker-based sandbox exists but is not the default[^4].

**Permissions:** Coarse allowlist model. The `Read` tool never prompts for file reads in the project directory. Bash commands use a subjective allowlist  - `whoami`, `ls`, and similar commands execute without confirmation. This is better than no model at all, but it operates at the command level, not the syscall level.

**Untrusted input:** CVE-2025-55284 demonstrated that hidden prompts in project files could trigger `.env` reads and DNS-based data exfiltration, bypassing the allowlist entirely[^5]. The permissive default allowlist meant the exfiltration commands didn't require user approval. Fixed in v1.0.4, but the architectural pattern  - trusted reads feeding into unmonitored shell execution  - remains.

**Verdict:** Has a permissions model, but it's coarse-grained and was bypassed by the first serious test. No per-syscall evaluation.

## 3. Codex (OpenAI)

**Sandbox:** The strongest in this list. On macOS, Codex uses Seatbelt policies. On Linux, seccomp + Landlock. Cloud deployments run in isolated containers with network disabled by default[^6]. Write access is limited to the active workspace.

**Permissions:** Configurable approval policy with three modes. Network access is off by default but can be enabled per-session. Filesystem writes are scoped to the workspace.

**Untrusted input:** A sandbox bypass vulnerability (patched in v0.39.0) allowed writes outside the intended boundary due to a path configuration bug. CVE-2025-61260 exposed a command injection flaw where MCP server entries executed at startup without user approval[^7]. Container-level isolation doesn't help when the injection point is inside the container.

**Verdict:** Best sandbox in the group, but container-level isolation can't catch prompt injection that operates within the container's scope. Security is perimeter-based, not per-operation.

## 4. Cursor

**Sandbox:** None. Cursor runs as an IDE extension with full access to the user's filesystem, terminal, and network.

**Permissions:** Allowlist-based command approval. Commands like `ls` are pre-approved. But the allowlist enforcement could be bypassed using brace expansion  - if `ls` was allowlisted, `ls $({rm,./test})` executed without confirmation[^8].

**Untrusted input:** HiddenLayer demonstrated README-driven prompt injection (CurXecute, CVE-2025-54135, CVSS 8.6) that used Cursor's own `read_file` and `create_diagram` tools to steal SSH keys and exfiltrate them to an attacker URL[^8]. The IDEsaster project found additional flaws (CVE-2025-49150)  - 100% of tested AI IDEs were vulnerable to prompt injection attack chains[^9]. Patched in Cursor v1.3.

**Verdict:** No sandbox, bypassable permissions, and demonstrated credential theft via prompt injection in project files.

## 5. Cline

**Sandbox:** None. Cline operates as a VS Code extension with direct CLI execution, file read/write, and network access.

**Permissions:** A `requires_approval` boolean set by the model itself. The model subjectively decides whether a command is "safe" or "potentially impactful." An "Auto-Approve" setting disables all confirmation prompts. A `.clinerules` directory can override the model's safety judgement entirely[^10].

**Untrusted input:** Mindgard found four vulnerabilities in a brief audit[^10]:

- DNS-based API key exfiltration via prompt injection in Python docstrings
- Arbitrary code execution by overriding `requires_approval` via `.clinerules`
- TOCTOU attacks where sequential file analysis circumvents safety checks
- Model information leakage via error messages

A separate supply chain attack ("Clinejection") turned Cline's own GitHub Actions issue triage bot into an attack vector, resulting in an unauthorised npm publish of `cline@2.3.0` that silently installed OpenClaw[^11].

**Verdict:** The permissions model is the model's own judgement, which can be overridden by the input it's processing. This is circular.

## 6. Aider

**Sandbox:** None. Aider runs as a CLI tool with the user's full terminal and filesystem access. No container, no filesystem boundary, no network restriction.

**Permissions:** Aider asks for confirmation before applying file edits and running shell commands. But it has no allowlist, no scoring, and no evaluation of *what* the command does  - just a yes/no prompt. After dozens of confirmations per session, developers stop reading.

**Untrusted input:** No published CVEs specific to Aider, but it is subject to the same prompt injection patterns as every other agent that reads project files and executes shell commands. The IDEsaster research found that 100% of tested AI coding tools were vulnerable to prompt injection chains[^9]. Aider's lack of any sandbox or programmatic permissions model means the attack surface is the user's entire system.

**Verdict:** Relies entirely on user vigilance for security. No architectural defence.

## 7. Open Interpreter

**Sandbox:** None by default. Open Interpreter was explicitly designed to run code directly on the user's machine  - "an open-source, locally running implementation of OpenAI's Code Interpreter." A `--safe-mode` flag exists but is opt-in and limits functionality.

**Permissions:** Asks for confirmation before code execution by default. A `--auto-run` flag (commonly used) disables this entirely. No allowlisting, no scoring, no operation-level evaluation.

**Untrusted input:** Open Interpreter processes natural language instructions and converts them to code that runs on the host system. Any prompt injection in input data (files, web content, messages) that the model processes becomes executable code with full system privileges. No published CVEs, but the architecture is designed to execute arbitrary code by intention  - the security boundary is the user pressing Enter.

**Verdict:** By design, there is no security boundary between the model's output and the operating system. This is the most permissive architecture in the group.

## The pattern across all seven

<figure>
  <a href="https://grith.ai/blog/security-audit-seven-ai-agents/figure-comparison-matrix-1400x840.png" target="_blank"><img src="https://grith.ai/blog/security-audit-seven-ai-agents/figure-comparison-matrix-1400x840.png" alt="Comparison matrix showing sandbox, permissions, and untrusted input handling across all seven AI agents" /></a>
  <figcaption>The security landscape: most agents have no sandbox, coarse or model-dependent permissions, and no defence against prompt injection.</figcaption>
</figure>

The common failure mode is the same across all seven: **the agent has broad capabilities, processes untrusted input, and nothing evaluates the resulting operations at the system call level**.

Codex comes closest with container-level isolation, but containers don't help when the prompt injection operates within the container's scope  - the attacker doesn't need to escape the sandbox if the agent does their work for them inside it.

## Comparison

| Agent              | Sandbox                          | Permissions                        | Untrusted input defence            | Known CVEs                |
| ------------------ | -------------------------------- | ---------------------------------- | ---------------------------------- | ------------------------- |
| **OpenClaw**       | None                             | None (localhost trusted)           | None                               | CVE-2026-25253, ClawHavoc |
| **Claude Code**    | None (optional Docker)           | Coarse allowlist                   | None (bypassed by DNS exfil)       | CVE-2025-55284            |
| **Codex**          | Seatbelt / seccomp + Landlock    | Configurable 3-mode policy         | Container-scoped only              | CVE-2025-61260            |
| **Cursor**         | None                             | Allowlist (bypassable)             | None (README-driven exfil)         | CVE-2025-54135, CVE-2025-49150 |
| **Cline**          | None                             | Model-decided (`requires_approval`)| None (overridable via .clinerules) | Clinejection, 4 Mindgard  |
| **Aider**          | None                             | Yes/no prompt only                 | None                               | None published             |
| **Open Interpreter**| None (opt-in `--safe-mode`)     | Yes/no prompt (bypassable)         | None (by design)                   | None published             |

Only one agent (Codex) has OS-level sandboxing. None have per-syscall evaluation. Every agent in the table processes untrusted input and converts it into operations that execute with the user's full privileges.

## What would actually work

The missing layer is per-syscall evaluation. Not a yes/no prompt. Not a command allowlist. Not container isolation. A security proxy that intercepts every file read, shell command, and network request  - evaluates it against multiple independent filters  - and makes a deterministic decision before the operation executes.

That's the architecture grith implements. Wrap any of these seven agents with `grith exec`, and every system call passes through 10+ security filters. Path matching catches `~/.ssh/id_rsa` reads. Taint tracking catches data flowing from sensitive sources to network egress. Destination reputation catches connections to unknown external hosts. The composite score determines: allow, queue for review, or deny.

The agent doesn't need to be modified. The security evaluation happens at the OS level, below the agent, where it can't be bypassed by prompt injection.

This is exactly what we are building at [grith.ai](https://grith.ai).

[^1]: [Security Boulevard: Securing OpenClaw Against ClawHavoc](https://securityboulevard.com/2026/02/securing-openclaw-againstclawhavoc/)
[^2]: [Kaspersky: New OpenClaw AI Agent Found Unsafe for Use](https://www.kaspersky.com/blog/openclaw-vulnerabilities-exposed/55263/)
[^3]: [CrowdStrike: What Security Teams Need to Know About OpenClaw](https://www.crowdstrike.com/en-us/blog/what-security-teams-need-to-know-about-openclaw-ai-super-agent/)
[^4]: [Claude Code Docs: Sandboxing](https://code.claude.com/docs/en/sandboxing)
[^5]: [Embrace The Red: Claude Code Data Exfiltration via DNS](https://embracethered.com/blog/posts/2025/claude-code-exfiltration-via-dns-requests/)
[^6]: [OpenAI Codex: Security](https://developers.openai.com/codex/security/)
[^7]: [The Hacker News: 30+ Flaws in AI Coding Tools](https://thehackernews.com/2025/12/researchers-uncover-30-flaws-in-ai.html)
[^8]: [HiddenLayer: How Hidden Prompt Injections Can Hijack AI Code Assistants Like Cursor](https://hiddenlayer.com/innovation-hub/how-hidden-prompt-injections-can-hijack-ai-code-assistants-like-cursor/)
[^9]: [MaccariTA: IDEsaster  - A Novel Vulnerability Class in AI IDEs](https://maccarita.com/posts/idesaster/)
[^10]: [Mindgard: Cline Bot AI Coding Agent Vulnerabilities](https://mindgard.ai/blog/cline-coding-agent-vulnerabilities)
[^11]: [Snyk: How Clinejection Turned an AI Bot into a Supply Chain Attack](https://snyk.io/blog/cline-supply-chain-attack-prompt-injection-github-actions/)
