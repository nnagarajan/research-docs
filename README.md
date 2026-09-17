# Research Docs

This workspace contains a small set of technical design notes and implementation guides for research and architecture exploration.

## Documents

### 1. Cedar Authorization Proposal

- File: [cedar-authorization-proposal.md](cedar-authorization-proposal.md)
- Focus: Centralized authorization using Cedar policies, relational metadata, and table-backed grants.
- Covers: policy-driven authorization for pages, APIs, and domain actions; schema design; guardrails; and a proposal for keeping authorization metadata in database tables while using Cedar for evaluation.

### 2. Deep Agents + assistant-ui Build Guide

- File: [deep-agents-assistant-ui.md](deep-agents-assistant-ui.md)
- Focus: Building a self-hosted chat interface over a LangChain deep-agent backend.
- Covers: architecture options, manual FastAPI server design, multi-user scoping, structured outputs, and frontend integration with assistant-ui.

## Purpose

These notes are intended to capture architecture decisions, implementation patterns, and design trade-offs related to:

- policy-based authorization
- agent orchestration and deep research workflows
- self-hosted frontend/backend integration
- practical deployment patterns without managed platform dependencies

## Notes

The documents are currently draft/reference material and may be expanded or refined as implementation work progresses.
