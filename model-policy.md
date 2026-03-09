# Hotel Domain — Auto-Model Selection Policy

Purpose: Dynamically select the optimal LLM model per agent invocation based on task type, agent role, and complexity — minimizing token cost while preserving output quality.

> **Scope:** Tourism AI-Assisted SDLC — Hotel vertical agents.  
> **Source:** Adapted from squad-sdk [response-tiers](squad/packages/squad-sdk/src/coordinator/response-tiers.ts) and [model-selector](squad/packages/squad-sdk/src/agents/model-selector.ts).

## 1) Model Tiers

| Tier | Models (preference order) | Cost | Speed | Use Case |
|------|--------------------------|------|-------|----------|
| **premium** | `claude-opus-4.6`, `claude-opus-4.5`, `claude-sonnet-4.5` | High | Slow | Architecture decisions, security audits, PRD generation, complex multi-service analysis |
| **standard** | `claude-sonnet-4.5`, `gpt-4.1`, `claude-sonnet-4` | Medium | Medium | Code generation, test writing, task decomposition, implementation planning |
| **fast** | `claude-haiku-4.5`, `gpt-4.1-mini`, `gpt-4.1-nano` | Low | Fast | Docs, changelogs, boilerplate, mechanical edits, code cleanup, routing |

## 2) Four-Layer Resolution Order

Model is resolved top-down. First match wins.

### Layer 1: User Override (highest priority)
If the user explicitly specifies a model (`model=<MODEL_ID>`) or explicitly names a target agent (`@Janitor`, `@PRD`), use the override or the agent's charter model unconditionally.

### Layer 2: Agent Charter Preference
Each agent's frontmatter `model:` field declares a preferred model. If `model: auto` or absent, skip to Layer 3.

### Layer 3: Task-Type Auto-Selection
Automatically select model based on the task's nature:

| Task Type | Model Tier | Rationale |
|-----------|-----------|-----------|
| `prd-generation` | premium | Complex domain analysis, multi-stakeholder requirements |
| `architecture` | premium | Complex reasoning, multi-service system analysis |
| `security-audit` | premium | Critical correctness requirement |
| `code-implementation` | standard | Balanced quality for code generation |
| `code-review` | standard | Needs nuance for review quality |
| `test-generation` | standard | Needs code understanding for quality tests |
| `task-decomposition` | standard | Structured breakdown following PRD template |
| `planning` | standard | Sprint planning, implementation sequencing |
| `code-cleanup` | fast | Mechanical modernization, pattern application |
| `documentation` | fast | Boilerplate-heavy, low complexity |
| `orchestration` | fast | Routing decisions, no code generation |
| `git-operations` | fast | Mechanical commands, branch/commit |
| `rename-move-delete` | fast | Simple mechanical file operations |

### Layer 4: Default Fallback (lowest priority)
If no layer matches: use **fast** tier (`claude-haiku-4.5`). This ensures cost-first behavior by default.

## 3) Fallback Chains

When a model is unavailable or errors out, fall back within the same tier:

```
premium:  claude-opus-4.6 → claude-opus-4.5 → claude-sonnet-4.5
standard: claude-sonnet-4.5 → gpt-4.1 → claude-sonnet-4
fast:     claude-haiku-4.5 → gpt-4.1-mini → gpt-4.1-nano
```

### Cross-Tier Protection
- A task assigned to `standard` tier MUST NOT escalate to `premium` during fallback.
- Fallback only moves sideways (same tier) or downward (cheaper tier).
- Exception: `User Override` (Layer 1) bypasses this rule.

## 4) Agent Role → Default Task Type Mapping

Each agent role maps to a default task type when the request doesn't specify one:

| Agent | Default Task Type | Default Tier | Charter Model |
|-------|------------------|--------------|---------------|
| `Hotel PRD Agent` | prd-generation | premium | `claude-opus-4.6` |
| `SE: Architect` | architecture | premium | `claude-opus-4.6` |
| `Senior Cloud Architect` | architecture | premium | `claude-opus-4.6` |
| `Hotel Task Planner` | task-decomposition | standard | `claude-sonnet-4.5` |
| `C#/.NET Janitor` | code-cleanup | fast | `claude-haiku-4.5` |

## 5) Complexity Escalation Rules

Any delegating agent MAY escalate a worker's tier based on task signals:

| Signal | Action |
|--------|--------|
| Task references > 3 different services/repositories | Escalate standard → premium |
| Task involves security/auth/encryption keywords | Escalate to premium |
| PRD contains > 5 functional requirements spanning multiple projects | Escalate task-decomposition to premium |
| Task file contains > 8 implementation steps | Escalate fast → standard |
| Task is a simple rename/move/config change | Downgrade to fast |

## 6) Delegation Handoff Convention

When one agent hands off to another, it SHOULD specify the resolved tier:

```
Tier: <premium|standard|fast|auto>
Reason: <brief rationale>
Context: <Jira ID, PRD path, or task description>
```

- `Tier: auto` → Target agent resolves via Layer 3 (task-type auto-selection).
- `Tier: <tier>` → Target agent uses specified tier.
- If tier is absent → Target agent falls back to its charter model (Layer 2) then Layer 3.

## 7) Response Tier (Message Complexity Gate)

Before delegating, the coordinating agent classifies the incoming message:

| Message Pattern | Response Tier | Model Tier | Action |
|----------------|---------------|------------|--------|
| Greeting, status query, help | **direct** | none | Answer inline, no agent spawn |
| `listele\|göster\|bul\|rename\|temizle\|lint` | **lightweight** | fast | → C#/.NET Janitor |
| `task\|plan\|decompose\|görev\|implement` | **standard** | standard | → Hotel Task Planner |
| `PRD\|gereksinim\|mimari\|security\|refactor entire` | **full** | premium | → PRD Agent / Architect |

## 8) How Agents Use This Policy

Every agent in this project MUST:

1. **Read `model-policy.md`** at session start to understand the tier system.
2. **Know its own default tier** from Section 4 (Agent Role mapping).
3. **Accept tier overrides** from the coordinating agent or user (Layer 1).
4. **Apply task-type auto-selection** (Layer 3) when the request nature differs from its default — e.g., if the Task Planner is asked to do a security review, it should recognize this as `premium` and inform the user.
5. **Never escalate cross-tier** during fallback without explicit user override.
