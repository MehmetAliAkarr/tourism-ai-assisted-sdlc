# Tourism AI-Assisted SDLC — Model Selection Strategy

> Adapted from [squad-sdk response-tiers](squad/packages/squad-sdk/src/coordinator/response-tiers.ts) and [model-selector](squad/packages/squad-sdk/src/agents/model-selector.ts).

## Model Tiers

| Tier | Models (fallback order) | Use Cases |
|------|------------------------|-----------|
| **Premium** | claude-opus-4.6 → claude-opus-4.5 → claude-sonnet-4.5 | Architecture proposals, security audits, PRD generation, complex design |
| **Standard** | claude-sonnet-4.5 → gpt-4.1 → claude-sonnet-4 | Code generation, refactoring, test writing, task decomposition |
| **Fast** | claude-haiku-4.5 → gpt-4.1-mini → gpt-4.1-nano | Boilerplate, changelogs, simple fixes, code cleanup, renames |

## Agent → Tier Mapping

| Agent | Tier | Model | Rationale |
|-------|------|-------|-----------|
| Hotel PRD Agent | Premium | claude-opus-4.6 | Complex domain analysis, multi-stakeholder requirements |
| SE: Architect | Premium | claude-opus-4.6 | Architecture review, security analysis, system design |
| Senior Cloud Architect | Premium | claude-opus-4.6 | Infrastructure design, NFR analysis |
| Hotel Task Planner | Standard | claude-sonnet-4.5 | Structured decomposition, follows PRD template |
| C#/.NET Janitor | Fast | claude-haiku-4.5 | Mechanical code cleanup, pattern application |

## Response Tier Selection

Incoming requests are classified using keyword patterns to determine the appropriate tier:

### Full (Premium) — Complex, multi-step, high-impact
- `refactor|redesign|migrate|rewrite` + `entire|all|whole|system|codebase`
- `implement|build|create` + `feature|module|system|service`
- `security audit|vulnerability scan`
- `PRD|requirements document|architecture review`
- `full review|full analysis`

### Standard — Typical coding and planning tasks
- Default tier for most development requests
- Code implementation, bug fixes, test writing
- Task planning and decomposition

### Lightweight (Fast) — Simple, mechanical, low-risk
- `list|show|display|get|find|search`
- `rename|move|delete|remove`
- `cleanup|format|lint|fix typo`
- `status|info|check`

### Direct — No agent needed
- Greetings: `hi|hello|hey|thanks`
- Status queries: `what is your name|help|usage`

## Full Policy Reference

See [model-policy.md](../model-policy.md) for the complete auto-model selection policy including:
- Four-layer resolution order
- Fallback chains and cross-tier protection
- Complexity escalation rules
- Delegation handoff conventions

## Agent Tooling Baseline

When creating or updating `.agent.md` files in this repository:

- Include `vscode/memory` in the frontmatter `tools` list by default.
- Only omit `vscode/memory` if there is a specific security or scope constraint.
- Keep existing tool naming conventions (`read`, `edit`, `search`, `agent`, `todo`, MCP namespaces, etc.) and add `vscode/memory` without removing required tools.
