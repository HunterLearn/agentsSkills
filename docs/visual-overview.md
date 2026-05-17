# Visual Overview

A diagram-first tour of this repository — what's here, how the pieces fit, and how to both *contribute* to it and *use* it. Read top to bottom; each section builds on the previous.

> All diagrams are Mermaid and render inline on GitHub. Verified counts at time of writing: **78 plugins** (77 local + 1 external via git-subdir), **184 agents**, **150 skills**, **98 commands**.

---

## 1. Repo map

```mermaid
flowchart TB
    Root["claude-agents/"]
    Root --> Manifest[".claude-plugin/<br/>marketplace.json"]
    Root --> Claude["CLAUDE.md<br/>(project conventions)"]
    Root --> README["README.md"]
    Root --> Plugins["plugins/<br/>(78 plugins)"]
    Root --> Docs["docs/<br/>(6 reference files)"]
    Root --> Tools["tools/<br/>(dev utilities)"]
    Root --> GH[".github/<br/>(CI workflows)"]

    Plugins --> P1["backend-development/"]
    Plugins --> P2["security-scanning/"]
    Plugins --> P3["plugin-eval/<br/>(the evaluator)"]
    Plugins --> PN["...74 more local<br/>+ 1 git-subdir"]

    Docs --> D1["plugins.md"]
    Docs --> D2["agents.md"]
    Docs --> D3["agent-skills.md"]
    Docs --> D4["usage.md"]
    Docs --> D5["architecture.md"]
    Docs --> D6["plugin-eval.md"]

    classDef edit fill:#d4edda,stroke:#28a745,color:#000
    classDef registry fill:#fff3cd,stroke:#ffc107,color:#000
    classDef ref fill:#e7f1ff,stroke:#0d6efd,color:#000
    class Plugins,P1,P2,P3 edit
    class Manifest registry
    class Docs,D1,D2,D3,D4,D5,D6,Claude,README ref
```

**Green = you edit it. Yellow = the registry. Blue = reference docs.** Everything contributors author lives under [`plugins/`](../plugins/); the [`.claude-plugin/marketplace.json`](../.claude-plugin/marketplace.json) registers those plugins for Claude Code to discover.

---

## 2. Plugin anatomy

```mermaid
flowchart LR
    subgraph Plugin["plugins/backend-development/"]
        direction TB
        Manifest[".claude-plugin/<br/>plugin.json<br/><i>only 'name' required</i>"]
        Agents["agents/<br/>backend-architect.md<br/>security-auditor.md<br/>tdd-orchestrator.md<br/>...8 total"]
        Skills["skills/<br/>api-design-principles/<br/>architecture-patterns/<br/>cqrs-implementation/<br/>...9 total"]
        Cmds["commands/<br/>feature-development.md"]
    end

    subgraph SkillInternals["skills/api-design-principles/"]
        SkillMd["SKILL.md<br/><b>(required)</b>"]
        Refs["references/<br/><i>(optional)</i>"]
        Assets["assets/<br/><i>(optional)</i>"]
    end

    Skills -.->|"directory pattern"| SkillInternals

    classDef required fill:#f8d7da,stroke:#dc3545,color:#000
    classDef optional fill:#e2e3e5,stroke:#6c757d,color:#000
    classDef primitive fill:#d1ecf1,stroke:#0c5460,color:#000
    class Manifest,SkillMd required
    class Refs,Assets optional
    class Agents,Skills,Cmds primitive
```

Every plugin has the same shape: a [`plugin.json`](../plugins/backend-development/.claude-plugin/plugin.json) manifest plus any combination of `agents/`, `skills/`, `commands/`. Components are **auto-discovered from the directory structure** — you don't list them in `plugin.json`. Skills are folders (so they can carry `references/` and `assets/`); agents and commands are single `.md` files.

---

## 3. Marketplace wiring

```mermaid
flowchart TB
    CC["Claude Code<br/>(loads marketplace at startup)"]
    Manifest[".claude-plugin/marketplace.json<br/>{ name, owner, plugins[] }"]
    CC -->|"reads"| Manifest

    Manifest -->|"plugins[].source<br/>= './plugins/X'"| Local["Local plugins<br/>77"]
    Manifest -->|"plugins[].source<br/>= { git-subdir, url }"| External["External plugins<br/>1 (qa-orchestra)"]

    Local --> PJ["plugin.json"]
    PJ -.->|"auto-discovery"| AgentsDir["agents/*.md"]
    PJ -.->|"auto-discovery"| SkillsDir["skills/&lt;name&gt;/SKILL.md"]
    PJ -.->|"auto-discovery"| CmdsDir["commands/*.md"]

    AgentsDir --> CCRuntime["Available to Claude<br/>at runtime"]
    SkillsDir --> CCRuntime
    CmdsDir --> CCRuntime

    classDef registry fill:#fff3cd,stroke:#ffc107,color:#000
    classDef runtime fill:#d4edda,stroke:#28a745,color:#000
    class Manifest,PJ registry
    class CC,CCRuntime runtime
```

`marketplace.json` is the single source of truth Claude Code reads. Each plugin entry points either at a local directory (`./plugins/X`) or a remote git repo (`source: { source: "git-subdir", url }`). Beyond that, **structure is convention** — no file enumerates the agents/skills/commands; they're picked up by directory layout.

---

## 4. Agent / Skill / Command taxonomy

```mermaid
flowchart TB
    subgraph Primitives["Three plugin primitives"]
        direction LR
        A["AGENT<br/><br/>Specialist persona<br/>Claude auto-selects<br/>based on task match"]
        S["SKILL<br/><br/>Reusable methodology<br/>Triggers on 'Use when...'<br/>conditions matching context"]
        C["COMMAND<br/><br/>User-imperative tool<br/>Invoked by typing<br/>/command-name [args]"]
    end

    subgraph Invocation["How each gets invoked"]
        direction LR
        AI["Claude reads task →<br/>matches agent description →<br/>delegates"]
        SI["Skill description trigger →<br/>auto-loaded into context<br/>when relevant"]
        CI["User types /name →<br/>Claude executes<br/>command body"]
    end

    A --> AI
    S --> SI
    C --> CI

    subgraph Tiers["Model tier legend (184 agents)"]
        direction TB
        T1["Opus 28% — architecture, security, code review"]
        T2["Inherit 27% — complex tasks, user picks"]
        T3["Sonnet 34% — docs, testing, debugging"]
        T4["Haiku 11% — fast ops, simple tasks"]
    end

    classDef agent fill:#d1ecf1,stroke:#0c5460,color:#000
    classDef skill fill:#d4edda,stroke:#28a745,color:#000
    classDef cmd fill:#fff3cd,stroke:#ffc107,color:#000
    class A,AI agent
    class S,SI skill
    class C,CI cmd
```

The three primitives differ in **who triggers them**. Agents are pulled in by Claude when a task description matches their `Use PROACTIVELY when …` clause. Skills auto-load when their `Use when …` trigger sentence matches the surrounding context. Commands run only when the user types `/<name>`. The tier breakdown shows Sonnet is the default workhorse, with Opus reserved for high-stakes specialists.

---

## 5. PluginEval pipeline

```mermaid
sequenceDiagram
    actor User
    participant CLI as plugin-eval CLI<br/>(cli.py)
    participant Parser as parser.py
    participant L1 as Layer 1: Static<br/>(static.py)
    participant L2 as Layer 2: Judge<br/>(judge.py)
    participant L3 as Layer 3: Monte Carlo<br/>(monte_carlo.py)
    participant Claude as Claude API
    participant Engine as engine.py

    User->>CLI: plugin-eval score &lt;path&gt; --depth standard
    CLI->>Parser: parse_skill(path)
    Parser-->>CLI: ParsedSkill
    CLI->>L1: analyze_skill(skill)
    Note over L1: deterministic checks<br/>~2s, free
    L1-->>Engine: LayerResult (score, anti-patterns)
    CLI->>L2: analyze_skill(skill) [async]
    L2->>Claude: 4x rubric prompts<br/>(triggering, orchestration,<br/>output, scope)
    Note over L2,Claude: Haiku + Sonnet<br/>~30s, ~4 calls
    Claude-->>L2: scores
    L2-->>Engine: LayerResult
    opt --depth deep or thorough
        CLI->>L3: analyze_skill(skill) [parallel with L2]
        L3->>Claude: N simulated activations
        Note over L3,Claude: ~2-5min, 50-100 calls
        Claude-->>L3: pass/fail per run
        L3-->>Engine: LayerResult (CI bounds)
    end
    Engine->>Engine: blend layers via<br/>DIMENSION_WEIGHTS
    Engine->>Engine: Badge.from_scores()<br/>Platinum/Gold/Silver/Bronze
    Engine-->>CLI: CompositeResult
    CLI-->>User: markdown / json / html report
```

[`plugins/plugin-eval/`](../plugins/plugin-eval/) is itself a plugin — and the quality gate for every other plugin. Three layers run in sequence (Static is always-on; Judge runs at `standard`+; Monte Carlo only at `deep`/`thorough`). Each layer emits a `LayerResult` which [`engine.py`](../plugins/plugin-eval/src/plugin_eval/engine.py) blends into a composite score using per-dimension weights, then [`models.py`](../plugins/plugin-eval/src/plugin_eval/models.py) assigns a badge.

### Scoring dimensions and badges (reference)

| Dimension | Weight | | Badge | Min composite | Min Elo |
|---|---|---|---|---|---|
| triggering_accuracy | 25% | | Platinum | ≥ 90 | ≥ 1600 |
| orchestration_fitness | 20% | | Gold | ≥ 80 | ≥ 1500 |
| output_quality | 15% | | Silver | ≥ 70 | ≥ 1400 |
| scope_calibration | 12% | | Bronze | ≥ 60 | ≥ 1300 |
| progressive_disclosure | 10% | | | | |
| token_efficiency | 6% | | | | |
| robustness | 5% | | | | |
| structural_completeness | 3% | | | | |
| code_template_quality | 2% | | | | |
| ecosystem_coherence | 2% | | | | |

Anti-patterns (caught by Layer 1 in [`static.py`](../plugins/plugin-eval/src/plugin_eval/layers/static.py)): `OVER_CONSTRAINED` (>15 MUST/ALWAYS/NEVER), `EMPTY_DESCRIPTION` (<20 chars), `MISSING_TRIGGER` (no "Use when…"), `BLOATED_SKILL` (>800 lines, no `references/`), `ORPHAN_REFERENCE`, `DEAD_CROSS_REF`.

---

## 6. Authoring lifecycle — adding a new plugin

```mermaid
flowchart TB
    Start(["I want to add a new plugin"]) --> Step1
    Step1["1. Scaffold directory<br/>plugins/&lt;name&gt;/"] --> Step2
    Step2["2. Create plugin.json<br/>plugins/&lt;name&gt;/.claude-plugin/plugin.json<br/>{ 'name': '&lt;name&gt;' }"] --> Step3
    Step3["3. Add primitives<br/>agents/*.md, skills/&lt;name&gt;/SKILL.md, commands/*.md<br/>(at least one)"] --> Step4
    Step4["4. Register in marketplace<br/>edit .claude-plugin/marketplace.json<br/>add entry with source, description, category"] --> Step5
    Step5["5. Evaluate quality<br/>cd plugins/plugin-eval<br/>uv run plugin-eval score path/to/plugin"] --> Decide{"Badge ≥ Bronze?"}
    Decide -->|"No"| Fix["Fix anti-patterns<br/>(OVER_CONSTRAINED, MISSING_TRIGGER, etc.)"]
    Fix --> Step5
    Decide -->|"Yes"| Step6["6. Update docs/<br/>plugins.md, agents.md, agent-skills.md"]
    Step6 --> End(["7. PR + ship"])

    classDef step fill:#e7f1ff,stroke:#0d6efd,color:#000
    classDef check fill:#fff3cd,stroke:#ffc107,color:#000
    classDef done fill:#d4edda,stroke:#28a745,color:#000
    class Step1,Step2,Step3,Step4,Step5,Step6 step
    class Decide,Fix check
    class End done
```

Steps 1–3 are pure file scaffolding under [`plugins/<name>/`](../plugins/). Step 4 — registering in [`.claude-plugin/marketplace.json`](../.claude-plugin/marketplace.json) — is the moment the plugin becomes installable. Step 5 is the quality gate: PluginEval will fail the loop until anti-patterns are gone and the composite score crosses Bronze.

---

## 7. Consumer lifecycle — using a plugin

```mermaid
flowchart TB
    Start(["I want to use a plugin"]) --> Install
    Install["1. Install marketplace<br/>/plugin marketplace add hunterlearn/agentsskills"] --> Enable
    Enable["2. Enable plugin(s)<br/>/plugin install &lt;plugin-name&gt;"] --> Browse
    Browse{"How will I invoke it?"}

    Browse -->|"Direct command"| Cmd["Type /command-name [args]<br/>e.g. /market-opportunity 'SaaS HR'"]
    Browse -->|"Let Claude pick agent"| Agent["Describe task naturally<br/>e.g. 'Design my postgres schema'<br/>→ Claude routes to database-architect"]
    Browse -->|"Trigger a skill"| Skill["Ask in matching context<br/>e.g. 'Do STRIDE on this API'<br/>→ stride-analysis-patterns auto-loads"]

    Cmd --> Result(["Result delivered"])
    Agent --> Result
    Skill --> Result

    classDef start fill:#e7f1ff,stroke:#0d6efd,color:#000
    classDef invoke fill:#d1ecf1,stroke:#0c5460,color:#000
    classDef done fill:#d4edda,stroke:#28a745,color:#000
    class Install,Enable start
    class Cmd,Agent,Skill invoke
    class Result done
```

After installing the marketplace and enabling one or more plugins, you have three distinct ways to invoke functionality. Commands are imperative (you type them). Agents are delegational (Claude routes to them based on how you describe the task). Skills are contextual (they load automatically when the surrounding conversation matches their trigger). See [`docs/usage.md`](usage.md) for a deeper walkthrough.

---

## 8. Decision guide — agent, skill, or command?

```mermaid
flowchart TB
    Start(["I want to add new functionality"]) --> Q1{"Who initiates it?"}
    Q1 -->|"User types it explicitly"| Q2{"Does it have args<br/>or a procedural body?"}
    Q1 -->|"Claude decides<br/>based on task"| Q3{"Is it a reusable<br/>methodology or<br/>a persona/specialist?"}

    Q2 -->|"Yes — argument-hint, multi-step"| Cmd["COMMAND<br/>commands/&lt;name&gt;.md<br/>Invoked via /name"]
    Q2 -->|"No — simple one-shot"| Cmd

    Q3 -->|"Persona / specialist<br/>(does the work)"| Ag["AGENT<br/>agents/&lt;name&gt;.md<br/>Claude delegates to it"]
    Q3 -->|"Methodology / pattern<br/>(informs the work)"| Sk["SKILL<br/>skills/&lt;name&gt;/SKILL.md<br/>Loaded when trigger matches"]

    Ag --> M{"What model tier?"}
    M -->|"Architecture, security,<br/>high-stakes review"| Opus["model: opus"]
    M -->|"Complex, user-dependent"| Inh["model: inherit"]
    M -->|"Docs, testing,<br/>debugging"| Son["model: sonnet"]
    M -->|"Fast ops,<br/>simple tasks"| Hai["model: haiku"]

    classDef question fill:#fff3cd,stroke:#ffc107,color:#000
    classDef answer fill:#d4edda,stroke:#28a745,color:#000
    classDef tier fill:#e7f1ff,stroke:#0d6efd,color:#000
    class Q1,Q2,Q3,M question
    class Cmd,Ag,Sk answer
    class Opus,Inh,Son,Hai tier
```

Use this when scoping a new contribution. The first split is *who triggers it*: if the user has to type it, it's a command. If Claude picks it, the second split is *what kind of thing it is*: a doer is an agent; a how-to is a skill. Agents additionally pick a model tier matching the stakes of the task.

---

## Where to go next

Read in this order to build mental model fast:

1. [**`docs/architecture.md`**](architecture.md) — design philosophy and *why* the primitives are split this way.
2. [**`docs/usage.md`**](usage.md) — concrete examples of invoking plugins as a consumer.
3. [**`docs/plugin-eval.md`**](plugin-eval.md) — full PluginEval framework details (rubrics, Elo, anti-patterns).
4. [**`docs/plugins.md`**](plugins.md) — catalog of all 78 plugins to browse what already exists before authoring something new.
5. [**`docs/agents.md`**](agents.md) and [**`docs/agent-skills.md`**](agent-skills.md) — reference indexes when you need a specific capability.
