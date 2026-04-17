# Bring Your Own LLM to Alteryx GenAI Tools

This repository provides a step-by-step guide to connecting Alteryx GenAI tools to a customer-hosted OpenAI-compatible LLM endpoint.

## Overview
This setup uses:
- GCP Ubuntu VM
- Ollama (local LLM runtime)
- Caddy (reverse proxy + TLS + auth)
- DuckDNS (public hostname)
- Alteryx One (connection + orchestration)

## Architecture
Designer Desktop → AIMS → Public Endpoint → Caddy → Ollama → Model → Response

## Quick Start
See:
- examples/Caddyfile
- examples/curl-tests.md
- docs/troubleshooting.md
