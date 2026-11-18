# Agent Team

This document describes the custom agent team orchestrated via GitHub Copilot CLI in a Codespace to build Mona's Project Pulse dashboard.

## Orchestrator
- **Model:** Claude Opus 5.5 (copilot)
- **Location:** `.github/agents/orchestrator.agent.md`
- **Responsibility:** Coordinates Planner, Coder, and Designer agents. Breaks down complex requests into phases, delegates specialized tasks to agents, manages execution order and dependencies, and verifies integrated results. Does not implement work directly.

## Planner
- **Model:** Claude Opus 5.5 (copilot)
- **Location:** `.github/agents/planner.agent.md`
- **Responsibility:** Creates implementation strategies by researching the codebase, documentation, dependencies, and edge cases. Produces detailed plans with ordered implementation steps, file assignments, dependencies, parallelization opportunities, and validation expectations. Does not write code.

## Coder
- **Model:** GPT-5.5 (copilot)
- **Location:** `.github/agents/coder.agent.md`
- **Responsibility:** Implements code-oriented tasks within assigned file scope. Writes code, fixes bugs, implements logic, and creates support configuration (e.g., `.vscode/launch.json` for the dashboard). Follows repository patterns, keeps code clear and testable, and validates changes before completion.

## Designer
- **Model:** Gemini 3.7 Flash (copilot)
- **Location:** `.github/agents/designer.agent.md`
- **Responsibility:** Handles UI/UX, accessibility, information architecture, interaction flow, and visual design. Creates a polished Project Pulse dashboard with visible project cards, status badges, responsive layout, and visual affordances. Prioritizes user outcomes and explains design tradeoffs.

## Execution Model

This specialized team works within the following coordination framework:

1. **Planner** analyzes requirements and creates a detailed implementation plan
2. **Orchestrator** parses the plan into phases with explicit file ownership
3. **Coder** and **Designer** work in parallel on non-overlapping file scopes
4. **Sequential phases** run when work overlaps or depends on earlier output
5. **Verification** ensures the integrated result coheres before reporting completion

## Governance

All agents operate within these constraints:

- Work remains within assigned file scopes
- Do not stage, commit, or push changes (learner controls git operations via Copilot CLI)
- Surface blockers instead of hiding them
- Validate work before reporting completion
