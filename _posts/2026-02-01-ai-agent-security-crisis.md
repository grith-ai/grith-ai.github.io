---
layout: post
title: "The AI Agent Security Crisis: 24 CVEs and Counting"
date: 2026-02-01
canonical_url: "https://grith.ai/blog/ai-agent-security-crisis"
description: "IDEsaster found 24 critical vulnerabilities across major AI coding assistants  - with a 100% exploitation rate. Here's what that means for developers."
tags:
  - security
  - research
  - CVE
---


The AI coding assistant ecosystem has a security problem  - and it's worse than most developers realise.

## The IDEsaster findings

In late 2025, security researchers published IDEsaster, a systematic audit of the most popular AI coding assistants. The results were sobering: **24 critical CVEs** across multiple products, with a **100% exploitation rate** in controlled testing.

These weren't theoretical vulnerabilities. Every single one was exploitable in default configurations, requiring no special setup or unusual conditions.

## What went wrong

The root cause is architectural. Most AI coding agents follow the same pattern: start with full system access, then try to restrict dangerous operations after the fact. This is the "allow by default, deny selectively" model  - and it has a fundamental flaw.

When an agent has ambient authority (unrestricted access to files, network, and shell), the attack surface is the entire system. Every tool call is a potential vector. Every prompt injection is a potential exploit.

## The grith approach

grith inverts this model entirely. Instead of hoping agents only do safe things, grith intercepts every system call  - file opens, network connections, process spawns  - and evaluates each one against 10+ independent security filters before it executes.

Every tool call passes through a multi-filter security proxy that produces a composite score:

- **Score < 3.0**: Auto-allow (80-90% of calls)
- **Score 3.0-8.0**: Queue for human review
- **Score > 8.0**: Auto-deny

This isn't a permission prompt. It's a scoring system that makes intelligent decisions about thousands of tool calls without interrupting your workflow.

## What developers should do

1. **Audit your current tools.** Check if your AI coding assistant has been affected by IDEsaster or similar findings.
2. **Assume breach.** If you've been using an affected tool, review your SSH keys, environment variables, and any secrets that were accessible.
3. **Consider architectural security.** Permission prompts are not security. Look for tools that provide defence in depth.

The AI agent security crisis is real. The question is whether we address it with architecture or hope.
