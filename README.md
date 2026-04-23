# Local LLM Setup Guide

Setup and troubleshooting for running Claude Code with local models via Ollama on macOS.

## Contents

- **CLAUDE.md** — Guidance for Claude Code when maintaining or updating this repository. Covers repository structure, key concepts, critical configuration gotchas, validation methodology, and common maintenance tasks.
- **ollama-coding-guide-v3.md** — Comprehensive guide for configuring Ollama + Claude Code with tested models (glm-4.7-flash, qwen3-coder). Covers LiteLLM proxy setup, environment configuration, and startup scripts.
- **claude-code-ollama-troubleshooting-2026-04-19.md** — Troubleshooting log documenting tool-call failures with qwen models and resolution via LiteLLM proxy path.
- **mempalace-claude-code-guide.md** — Setup and integration guide for MemPalace (persistent memory system) with Claude Code on macOS ARM64. Covers installation pitfalls, Python versioning, ChromaDB workarounds, MCP registration, hooks configuration, and verification steps.

## Environment

Verified on:
- **Hardware**: M3 Pro 36 GB
- **Ollama**: 0.21.0
- **Claude Code**: 2.1.114
- **macOS**: Tahoe 26.4

## Status

Last updated: April 23, 2026. All flags and configurations tested against live installation.
