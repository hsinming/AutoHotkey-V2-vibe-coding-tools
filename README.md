# AutoHotkey V2 Vibe Coding Tools

A comprehensive ecosystem of AI agent prompts, context modules, and skills designed for elite AutoHotkey v2 (AHK v2) development. This project provides a structured framework for AI agents (like Roo or OpenCode) to produce production-grade, object-oriented AHK v2 code.

## 🚀 Overview

This repository organizes AHK v2 knowledge into modular "Context Modules" and "Skills" that can be dynamically loaded by AI agents. It enforces strict engineering principles, architectural review gates, and modern AHK v2 syntax standards.

## 📂 Project Structure

- **`Context Modules/`**: The core knowledge base. Contains detailed markdown files covering specific AHK v2 domains (GUI, OOP, DllCall, etc.).
- **`skills/`**: Roo-compatible skills that wrap context modules into actionable agent behaviors.
    - `get-ahk-core-context`: Core syntax, OOP, and formatting.
    - `get-ahk-ui-context`: GUI, Input, and Window management.
    - `get-ahk-system-context`: FileSystem, Network, and COM.
    - `get-ahk-logic-context`: Async, Errors, and Text processing.
- **`Roo code mode prompts/`**: Custom mode definitions for the Roo agent, implementing a multi-agent orchestration workflow.
- **`OpenCode agent prompts/`**: Specialized prompts for OpenCode agents.

## 🤖 Agent Ecosystem (Multi-Agent Workflow)

The project implements a tiered agent architecture to ensure code quality:

1.  **🪃 Orchestrator (`ahk-orchestrator`)**: The entry point. Routes requests to the appropriate specialist based on a "Design Decision Test."
2.  **🏗️ Architect (`ahk-architect`)**: Design authority. Produces architectural blueprints and success criteria.
3.  **💻 Code (`ahk-code`)**: Implementation engine. Writes AHK v2 code based on upstream blueprints.
4.  **🪲 Debug (`ahk-debug`)**: Auditor. Analyzes errors and provides verified fixes.
5.  **❓ Ask (`ahk-ask`)**: Technical mentor. Provides explanations and tutorials.

## 🛠️ Engineering Principles

All code generated through this ecosystem adheres to:
- **KISS & YAGNI**: Simplest solution that works.
- **Information Hiding**: Exposing behavior via methods, not raw data.
- **SoC & Cohesion**: Each class owns exactly one concern.
- **Progressive Disclosure**: Context is loaded on-demand, never preloaded in bulk.

## 📜 Syntax & Formatting Standards

- **AHK v2 Only**: Strict adherence to v2 syntax (no v1 residue).
- **OOP First**: Heavy emphasis on classes and object-oriented patterns.
- **K&R / OTB Style**: Opening braces on the same line.
- **4-Space Indentation**: No tabs.
- **CRLF Line Endings**: Standard Windows compatibility.

## 📖 Usage

To use these tools with a compatible AI agent:
1.  Point the agent to this workspace.
2.  The agent will automatically discover the `.roo/skills` or custom mode definitions.
3.  Start a task (e.g., "Create a GUI for managing system services") and the Orchestrator will guide the process.

---
*Developed for elite AutoHotkey v2 engineering.*
