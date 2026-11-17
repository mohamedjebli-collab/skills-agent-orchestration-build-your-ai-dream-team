# Agent team

I am using GitHub Copilot CLI in a Codespace to orchestrate the work of building Mona's Project Pulse dashboard. The custom agents below are defined in `.github/agents/`.

| Agent | Target model | Responsibility | Definition |
| --- | --- | --- | --- |
| Orchestrator | Claude Opus 5.5 (copilot) | Breaks down the request, delegates to the specialists with explicit file scopes, runs work in parallel only when scopes don't overlap, verifies the integrated result, and reports. Does not implement. | `.github/agents/orchestrator.agent.md` |
| Planner | Claude Opus 5.5 (copilot) | Researches the repo and docs, then returns an ordered plan with file assignments, dependencies, parallel/sequential work, edge cases, and validation. Writes no code. | `.github/agents/planner.agent.md` |
| Coder | GPT-5.5 (copilot) | Implements code and logic within its assigned files, plus runnable-app support such as `.vscode/launch.json` (with `cwd` set to `${workspaceFolder}/app`). Validates before reporting. | `.github/agents/coder.agent.md` |
| Designer | Gemini 3.7 Flash (copilot) | Handles UI/UX, accessibility, and visual design: a polished, responsive dashboard with project cards, status badges, and priority treatment (`.dashboard`, `.project-card`). | `.github/agents/designer.agent.md` |

## Workflow

1. The Orchestrator asks the Planner for a plan.
1. The Orchestrator splits the plan into phases with non-overlapping file scopes.
1. The Coder and Designer implement their assigned files.
1. The Orchestrator verifies the result and reports.

All agents are told not to stage, commit, or push; I control git through Copilot CLI prompts.
