# Gemma 4 on Ollama — A Working Playbook for C#/.NET + Angular

**Scope:** getting reliable architecture plans, PlantUML, design descriptions, and production-shaped C# / Angular code out of a locally hosted Gemma 4.

**Version basis:** Gemma 4 family (E2B, E4B, 12B, 26B A4B MoE, 31B Dense), Apache 2.0, as published on the Ollama library and Google's model docs.

---

## 0. Thirty-second start

```bash
ollama pull gemma4:31b          # 20 GB · 256K ctx · best coder in the family
ollama run  gemma4:31b
```

If you only take three things from this document:

1. **Use `31b` for architecture and code, `26b` when you need speed, `12b` on a 16 GB laptop.**
2. **Set `num_ctx` explicitly.** Ollama will not give you the 256K window by default, and silent truncation is the #1 cause of "the model ignored my repo."
3. **Never ask for architecture and code in the same turn.** Separate the plan, the diagram, and the implementation into distinct calls with distinct system prompts. Quality roughly doubles.

---

## 1. Choosing the model

### 1.1 The family

| Tag | Download | Context | Architecture | Modalities |
|---|---|---|---|---|
| `gemma4:e2b` | 7.2 GB | 128K | Dense, ~2.3B effective (5.1B w/ embeddings) | Text, Image, Audio |
| `gemma4:e4b` | 9.6 GB | 128K | Dense, ~4.5B effective (8B w/ embeddings) | Text, Image, Audio |
| `gemma4:12b` | 7.6 GB | 256K | Dense, encoder-free unified | Text, Image, Audio |
| `gemma4:26b` | 18 GB | 256K | MoE, 25.2B total / 3.8B active (8 of 128 experts + 1 shared) | Text, Image |
| `gemma4:31b` | 20 GB | 256K | Dense, 30.7B, 60 layers | Text, Image |
| `gemma4:31b-cloud` | — | 256K | Hosted on Ollama's cloud | Text, Image |
| `gemma4:*-mlx` | varies | — | MLX builds for Apple Silicon | Text, Image |

Note the counterintuitive line: **`12b` downloads smaller than `e4b`.** The E-models carry large Per-Layer Embedding tables that inflate on-disk weight size well past their "effective" parameter count.

### 1.2 What the benchmarks say about *coding specifically*

| Metric | 31B | 26B A4B | E4B | E2B | Gemma 3 27B |
|---|---|---|---|---|---|
| LiveCodeBench v6 | **80.0%** | 77.1% | 52.0% | 44.0% | 29.1% |
| Codeforces ELO | **2150** | 1718 | 940 | 633 | 110 |
| MMLU Pro | 85.2% | 82.6% | 69.4% | 60.0% | 67.6% |
| Tau2 (agentic, avg of 3) | **76.9%** | 68.2% | 42.2% | 24.5% | 16.2% |
| MRCR v2, 8-needle @128K | **66.4%** | 44.1% | 25.4% | 19.1% | 13.5% |

Read those last two rows carefully — they matter more than LiveCodeBench for your use case:

- **Tau2** is the agentic / tool-use score. If you plan to drive an agent harness (Claude Code, OpenCode) against a real `.sln`, the gap between 31B (76.9%) and 26B (68.2%) is a gap in *how often the loop completes without you babysitting it*.
- **MRCR** is long-context retrieval. At 128K with 8 needles, the best model in the family finds two-thirds of what it's looking for. **The 256K window is real, but it is not a substitute for curation.** More on this in §7.

### 1.3 Selection rules

| Your situation | Use |
|---|---|
| Workstation, ≥ 24 GB VRAM or unified memory | `gemma4:31b` |
| 24–32 GB, but you need low latency (interactive completion, agent loops) | `gemma4:26b` — 3.8B active params means it generates roughly at 4B speed |
| Laptop, 16 GB VRAM / unified | `gemma4:12b` |
| Air-gapped, secret-cleared, or otherwise offline | Any of the above. Apache 2.0, weights are yours. |
| Quick file rename / commit message / regex | `gemma4:e4b` |
| You need the answer to be *right* and you can wait | `gemma4:31b` with `think: "high"` |

**For serious .NET and Angular work: `gemma4:31b`.** The 26B MoE is the sensible fallback and is genuinely fast, but its long-context retrieval (44.1% MRCR) makes it a poor fit for "here is my Clean Architecture solution, add a feature slice."

---

## 2. Baseline setup

### 2.1 Pull and verify

```bash
ollama pull gemma4:31b
ollama show gemma4:31b            # confirm capabilities: vision, tools, thinking
ollama ps                         # after a run: confirm 100% GPU, not partial CPU offload
```

If `ollama ps` shows a CPU/GPU split, you are about to have a bad time. Drop to `26b`, or reduce `num_ctx`.

### 2.2 Environment variables that actually matter

```bash
# Long context without exploding VRAM
export OLLAMA_FLASH_ATTENTION=1
export OLLAMA_KV_CACHE_TYPE=q8_0     # q8_0 is the safe quality/size tradeoff; q4_0 is lossy

# Default context for every model, if you don't want to set num_ctx per-request
export OLLAMA_CONTEXT_LENGTH=65536

# Keep the model resident between prompts (avoid 20 GB reload on every call)
export OLLAMA_KEEP_ALIVE=30m
```

`OLLAMA_KV_CACHE_TYPE` requires flash attention to be on. Without these two, a 64K context on a 31B model will eat far more memory than the 20 GB of weights suggests.

### 2.3 Context budgeting

The KV cache grows with tokens, not with weights. Practical targets:

| `num_ctx` | Use for |
|---|---|
| 8192 | Single-file edits, code review of one class |
| 32768 | A feature slice — command, handler, validator, endpoint, tests |
| 65536 | Cross-layer architecture work; a bounded context |
| 131072+ | Only when you have measured the memory and accepted the retrieval degradation |

**Do not set `num_ctx` to 262144 "just in case."** You will allocate the cache whether or not you fill it.

---

## 3. Sampling and thinking mode

### 3.1 Official sampling configuration

Google publishes one standardized config for **all** use cases:

```
temperature = 1.0
top_p       = 0.95
top_k       = 64
```

This will feel wrong. Every instinct built on other model families says "temperature 0 for code." Resist it initially. Gemma 4's post-training assumes this distribution, and clamping temperature to 0 tends to produce repetition loops and degenerate output rather than determinism.

**If you must reduce variance for codegen**, do it in this order:

1. First, constrain the *output shape* with structured outputs (§6) — this gives you determinism of form without touching sampling.
2. Then, if still needed, step temperature down in increments: `1.0 → 0.8 → 0.6`. Evaluate at each step against your own task, don't jump to 0.
3. Ollama's own structured-output guidance suggests `temperature: 0` for pure JSON extraction. That's fine for *extraction*. It is not fine for *generation*.

### 3.2 Thinking mode

All Gemma 4 models are reasoners with configurable thinking. This is the single highest-leverage switch for architecture work.

**Via the API** (the documented, portable way):

```json
{
  "model": "gemma4:31b",
  "messages": [ ... ],
  "think": "high",
  "stream": false,
  "options": { "temperature": 1.0, "top_p": 0.95, "top_k": 64, "num_ctx": 65536 }
}
```

`think` accepts `true`/`false` or a level: `"low"`, `"medium"`, `"high"`, `"max"`.

**Under the hood** (useful if you're writing your own client against llama.cpp, or debugging a template):

- Thinking is triggered by a `<|think|>` control token at the **start of the system prompt**. Remove it to disable.
- When enabled, output is structured as `<|channel>thought\n` … internal reasoning … `<channel|>` followed by the final answer.
- With thinking disabled, every model **except E2B/E4B** still emits the tags with an empty thought block. Your parser must tolerate this.

Ollama handles this templating for you. Don't hand-roll the tokens unless you're bypassing Ollama.

### 3.3 The multi-turn rule you will get wrong

> **In multi-turn conversations, historical assistant messages must contain only the final response. Never feed a previous turn's thinking content back into the next turn.**

This is explicit in the model card. If you're building a chat loop, a RAG pipeline, or a Yokefellow-style agent, strip the thought channel from history before the next user turn. Leaving it in degrades subsequent reasoning.

### 3.4 Thinking levels by task

| Task | `think` |
|---|---|
| Boilerplate DTO, mapping profile, barrel file | `false` |
| Angular component scaffold from a clear spec | `"low"` |
| Write a CQRS handler + unit tests | `"medium"` |
| Design a bounded context; choose between two aggregate boundaries | `"high"` |
| Trace a concurrency bug across SignalR hub → Redis backplane → EF Core | `"max"` |

---

## 4. Modelfiles: build three specialists, not one generalist

Ollama Modelfiles let you bake system prompt + parameters into a named model. This beats pasting a system prompt into every session, and it makes your setup reproducible across machines.

### 4.1 `arch-gemma` — the architect

```dockerfile
# Modelfile.arch
FROM gemma4:31b

PARAMETER temperature 1.0
PARAMETER top_p 0.95
PARAMETER top_k 64
PARAMETER num_ctx 65536
PARAMETER repeat_penalty 1.0

SYSTEM """
You are a principal software architect specializing in .NET 8+ and Angular.
You design systems using Clean Architecture with CQRS, MediatR-style request
pipelines, and Domain-Driven Design tactical patterns.

Your output discipline:
- You produce DESIGN, never implementation code, unless explicitly asked.
- Every design decision is stated as: Decision / Rationale / Consequences / Rejected Alternatives.
- You name concrete types, projects, and namespaces. "A service layer" is a
  failure; "Application/Orders/Commands/PlaceOrder/PlaceOrderCommandHandler.cs"
  is an answer.
- You state your assumptions in a numbered list BEFORE the design, and you
  flag any assumption whose falsity would change the design.
- When a requirement is ambiguous, you propose the two most likely readings
  and design for the more constrained one, noting the fork.
- You do not invent NuGet packages. If you are not certain a package exists
  at a given version, you say "verify: <package>" instead of asserting it.

Constraints you always honour:
- Dependencies point inward. Domain references nothing. Application references
  Domain. Infrastructure and Api reference Application.
- No infrastructure types in Application signatures. No EF Core in Domain.
- Every command handler has a corresponding validator and a failure path.
- Every cross-boundary contract is a DTO, never an entity.
"""
```

```bash
ollama create arch-gemma -f Modelfile.arch
```

### 4.2 `dotnet-gemma` — the implementer

```dockerfile
# Modelfile.dotnet
FROM gemma4:31b

PARAMETER temperature 0.8
PARAMETER top_p 0.95
PARAMETER top_k 64
PARAMETER num_ctx 32768

SYSTEM """
You are a senior .NET engineer. You write C# for .NET 8+ targeting a Clean
Architecture solution.

Rules:
- Nullable reference types are enabled. File-scoped namespaces. Primary
  constructors where they clarify. `sealed` by default on classes that are
  not designed for inheritance.
- Async all the way. Every async method takes CancellationToken and passes it on.
  No `async void`. No `.Result`, no `.Wait()`, no `.GetAwaiter().GetResult()`.
- Return Result<T> or a discriminated outcome for expected failures. Exceptions
  are for the exceptional.
- No `using` directives you do not use. No regions. No commented-out code.
- Tests use xUnit + FluentAssertions + NSubstitute unless told otherwise.
  Arrange/Act/Assert, named Method_Scenario_ExpectedBehaviour.

Output format:
- One fenced block per file, and the first line of the block is a comment
  containing the full relative path.
- If a change touches an existing file, output the complete file, not a diff,
  unless asked for a diff.
- After the code, list every assumption you made in one line each.

You do NOT explain what the code does line by line. The code is the explanation.
"""
```

### 4.3 `ng-gemma` — the Angular implementer

```dockerfile
# Modelfile.ng
FROM gemma4:31b

PARAMETER temperature 0.8
PARAMETER top_p 0.95
PARAMETER top_k 64
PARAMETER num_ctx 32768

SYSTEM """
You are a senior Angular engineer working in a modern Angular workspace.

Baseline assumptions (correct me in your assumption list if the provided code
contradicts these):
- Standalone components. No NgModules.
- Signals for local state; `input()` / `output()` functions, not decorators.
- `inject()` over constructor injection.
- `ChangeDetectionStrategy.OnPush` on every component.
- New control flow: @if / @for / @switch. Never *ngIf, *ngFor.
- RxJS only where genuinely asynchronous and multi-valued; otherwise signals.
  Every subscription is either `takeUntilDestroyed()` or an `async` pipe.
- Typed reactive forms.

Output format:
- One fenced block per file, first line is a comment with the full relative path.
- Component, template, and spec together. A component without a spec is incomplete.
- SignalR clients belong in a service, never in a component.

State any assumption about Angular version, and if the user's code reveals the
version, adjust and say so.
"""
```

**Why three models and not one:** the architect prompt forbids code; the implementer prompts forbid architecture prose. Each constraint set is short enough that a 31B model actually holds it. A single 2,000-token "do everything" system prompt gets diluted and ignored.

---

## 5. The five-phase workflow

Local models fail when asked to plan and build in one breath. Split it. Each phase is a separate call, and **the artifact from phase N is the input to phase N+1** — not the conversation history.

```
① FRAME → ② ARCHITECT → ③ DIAGRAM → ④ IMPLEMENT → ⑤ VERIFY
```

### ① Frame — extract the requirement

Model: `gemma4:26b`, `think: "low"`. Fast, cheap.

```
Read the following requirement. Produce:
1. The bounded context(s) implied.
2. The aggregate roots and their invariants.
3. The commands (state-changing) and queries (read-only).
4. The integration points (external systems, real-time channels, caches).
5. Every ambiguity, as a question.

Do not design. Do not write code.

REQUIREMENT:
<paste>
```

Answer the questions it raises. This is the single highest-return five minutes in the whole loop.

### ② Architect — get the plan

Model: `arch-gemma`, `think: "high"`.

```
Using the framing below, produce an architecture plan for a .NET 8 Clean
Architecture solution with an Angular frontend.

Cover, in this order:
1. Assumptions (numbered; flag load-bearing ones).
2. Solution structure: every project, its purpose, its dependencies.
3. Domain model: aggregates, entities, value objects, domain events.
4. Application layer: every command and query, with its handler, validator,
   and failure modes.
5. Persistence: EF Core configuration approach, concurrency strategy, indexes.
6. Real-time: SignalR hubs, groups, backplane, reconnection semantics.
7. Cross-cutting: authn/authz, logging, caching, idempotency.
8. Frontend: feature structure, state ownership, the API contract.
9. Decision log: Decision / Rationale / Consequences / Rejected Alternatives.
10. Risks, ordered by likelihood × cost.

FRAMING:
<paste output of phase ①>
```

Take the output. **Edit it.** The model is a strong first draft, not an authority. Your edited plan is now the source of truth for everything downstream.

### ③ Diagram — PlantUML from the plan

Model: `arch-gemma`, `think: "medium"`. See §8 for the full treatment.

### ④ Implement — one slice at a time

Model: `dotnet-gemma` or `ng-gemma`, `think: "medium"`.

```
Implement the PlaceOrder command slice from the plan below.

Deliver, as separate files:
- PlaceOrderCommand.cs
- PlaceOrderCommandHandler.cs
- PlaceOrderCommandValidator.cs
- The endpoint registration
- PlaceOrderCommandHandlerTests.cs

Use only types that appear in the plan or in the provided existing code.
If you need a type that does not exist, stop and list it under "MISSING TYPES"
rather than inventing it.

PLAN (relevant excerpt only):
<paste>

EXISTING CODE (contracts and base types only):
<paste>
```

**"Relevant excerpt only" is the whole trick.** Do not paste the full plan. Do not paste the full repo. The model's retrieval degrades measurably past ~64K, and precision falls long before recall does.

### ⑤ Verify — close the loop with a compiler

The model does not know whether its code compiles. Your build does.

```bash
dotnet build 2>&1 | head -50    # feed failures back
dotnet test
ng lint && ng test --watch=false
```

Feed the *first* compiler error back, not all of them. Fixing error #1 usually cascades. Pasting 40 errors invites the model to rewrite everything.

---

## 6. Structured outputs — the reliability lever

Ollama constrains generation to a JSON Schema via the `format` parameter. For architecture work this is transformative: you stop parsing prose and start receiving data.

### 6.1 An architecture plan as a schema

```bash
curl -s http://localhost:11434/api/chat -d '{
  "model": "arch-gemma",
  "stream": false,
  "think": "high",
  "messages": [
    {"role": "user", "content": "Design the ordering bounded context. Return as JSON."}
  ],
  "format": {
    "type": "object",
    "properties": {
      "assumptions": { "type": "array", "items": { "type": "string" } },
      "projects": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "name":         { "type": "string" },
            "layer":        { "type": "string", "enum": ["Domain","Application","Infrastructure","Api","Web"] },
            "purpose":      { "type": "string" },
            "dependsOn":    { "type": "array", "items": { "type": "string" } }
          },
          "required": ["name","layer","purpose","dependsOn"]
        }
      },
      "aggregates": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "name":       { "type": "string" },
            "invariants": { "type": "array", "items": { "type": "string" } },
            "entities":   { "type": "array", "items": { "type": "string" } },
            "valueObjects": { "type": "array", "items": { "type": "string" } }
          },
          "required": ["name","invariants","entities","valueObjects"]
        }
      },
      "useCases": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "name":         { "type": "string" },
            "kind":         { "type": "string", "enum": ["command","query"] },
            "handler":      { "type": "string" },
            "validator":    { "type": "string" },
            "failureModes": { "type": "array", "items": { "type": "string" } }
          },
          "required": ["name","kind","handler","validator","failureModes"]
        }
      },
      "decisions": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "decision":     { "type": "string" },
            "rationale":    { "type": "string" },
            "consequences": { "type": "array", "items": { "type": "string" } },
            "rejected":     { "type": "array", "items": { "type": "string" } }
          },
          "required": ["decision","rationale","consequences","rejected"]
        }
      }
    },
    "required": ["assumptions","projects","aggregates","useCases","decisions"]
  }
}'
```

### 6.2 Why this matters beyond tidiness

Once the plan is JSON, it is **machine-checkable**:

- Assert `dependsOn` never points outward from Domain. That's a lint rule, not a code review.
- Assert every `useCase` of kind `command` has a non-null `validator`.
- Feed `projects[]` into a scaffolding script that emits the `.sln` and `.csproj` files. The model designs; a deterministic tool builds.
- Diff two plans across iterations to see what actually changed.

This is the same instinct behind an architecture-description style guide with enforcement tooling: **make the description structured enough that a machine can hold you to it.** Structured outputs are how you get an LLM to participate in that discipline instead of undermining it.

### 6.3 Notes and gotchas

- Include *"Return as JSON"* in the prompt text as well as setting `format`. The schema constrains the grammar; the prompt aligns the intent. Both help.
- Lower `temperature` for pure extraction. Keep it at 1.0 when the JSON fields contain generative prose (rationale, consequences).
- **Ollama's hosted cloud models do not currently support structured outputs.** This is a local-only capability. For an air-gapped workstation that's a feature, not a limitation.
- Python: define with Pydantic, pass `Model.model_json_schema()` to `format`, validate with `model_validate_json()`. TypeScript: define with Zod, pass `z.toJSONSchema()`. Validation on the way back is not optional — the grammar constrains shape, not semantics.

---

## 7. Context strategy: the 256K trap

You have 256K tokens. You should rarely use more than 64K.

**The evidence:** on MRCR v2 with 8 needles at 128K, the 31B scores 66.4%. The 26B MoE scores 44.1%. That is a retrieval benchmark under ideal conditions. Your codebase is worse than ideal — it's full of near-duplicate names, `IRepository<T>` in four places, and three classes called `OrderService`.

### What to actually put in context

**Tier 1 — always include:**
- The edited plan excerpt for the current slice
- Type signatures and interfaces the code must satisfy
- One exemplar: an existing slice done correctly, as a style anchor

**Tier 2 — include when relevant:**
- The `.editorconfig` / lint rules
- The relevant EF Core configuration
- The failing test or compiler error

**Tier 3 — almost never:**
- Whole files where only the signature matters
- Generated code, migrations, `.csproj` boilerplate
- The full plan when you're building one slice of it

### The exemplar technique

The single most effective thing you can put in context is **one file from your own codebase that does the thing correctly.** A model shown `PlaceOrderCommandHandler.cs` will produce a `CancelOrderCommandHandler.cs` that matches your conventions far better than any system prompt describing those conventions. Conventions are easier to demonstrate than to specify.

### Repo packing

If you must give it breadth, give it a map rather than the territory:

```bash
# A signature-only view of the solution
find ./src -name "*.cs" -exec grep -H -E "^\s*(public|internal)\s+(sealed\s+)?(class|record|interface|enum)" {} \; \
  | sed 's/{.*//' > solution-map.txt
```

A 3K-token map of every public type beats a 90K-token dump of every file, and it leaves room for the model to think.

---

## 8. PlantUML generation

Gemma 4 writes PlantUML well. It writes *valid* PlantUML somewhat less well. Structure the request so that validity is checkable.

### 8.1 The diagram prompt

```
Produce a PlantUML C4 Container diagram for the architecture below.

Hard constraints:
- Start with @startuml, end with @enduml. Nothing outside those markers.
- Use only the C4-PlantUML standard library:
    !include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml
- Every Container() and System() must have a unique alias in snake_case.
- Every Rel() must reference aliases that you have already declared.
- Do not use skinparam. Do not use !theme. Do not use sprites.
- Add LAYOUT_WITH_LEGEND().

Output ONLY the PlantUML source in a single fenced block. No prose.

ARCHITECTURE:
<paste plan §2 and §6>
```

Then, for the same plan, generate in separate calls:

| Diagram | Ask for |
|---|---|
| C4 Context | Actors, the system, external systems |
| C4 Container | Projects, database, cache, SignalR, SPA |
| C4 Component | One project's internals — the Application layer |
| Sequence | One use case end to end, including the failure path |
| Class | One aggregate: root, entities, value objects, invariants as notes |
| State | An aggregate's lifecycle if it has one |
| Deployment | Azure resources and network boundaries |

**One diagram per call.** Asking for six diagrams in one response reliably produces six mediocre diagrams.

### 8.2 The validation loop

```bash
# Syntax check without rendering
plantuml -checkonly diagram.puml

# Or render and inspect
plantuml -tsvg diagram.puml
```

Wire this into a script. Generate → check → on failure, feed `plantuml`'s stderr back with:

> `The PlantUML failed to parse with this error. Return the corrected source only.`

Two iterations clears nearly everything. This is exactly the shape of a Trellis-style editor loop: generate, render, correct — with the renderer, not the model, as the arbiter of correctness.

### 8.3 Common Gemma 4 PlantUML failure modes

| Symptom | Fix |
|---|---|
| Prose before `@startuml` | Add "Output ONLY the fenced block. No preamble." Or strip everything outside the markers programmatically. |
| `Rel()` to an undeclared alias | Ask it to emit all `Container()` declarations first, then all `Rel()` calls. Enforce ordering in the prompt. |
| Invented C4 macros (`Container_Boundary_Ext`) | Whitelist the macros in the prompt: "You may use only: Person, System, System_Ext, Container, ContainerDb, Container_Boundary, Rel, Rel_Back, LAYOUT_WITH_LEGEND." |
| Mermaid syntax bleeding in (`-->` with labels in `\|\|`) | Say "This is PlantUML, not Mermaid" explicitly. Yes, really. |
| Diagram sprawls to 40 containers | Constrain: "At most 12 containers. Group by bounded context." |

### 8.4 The description that accompanies the diagram

Diagrams without prose are half an artifact. Generate the narrative **from the diagram source**, not from the plan:

```
Here is a PlantUML C4 Container diagram. Write the architecture description
that accompanies it.

Structure:
- Purpose: one paragraph, what the system does and for whom.
- Container walkthrough: one paragraph per container, in dependency order.
- Key flows: for each primary use case, the path through the containers.
- Quality attributes: how the structure serves availability, security,
  observability, and change. Name the tradeoff each one cost.
- Constraints and assumptions.

Write for an engineer joining the team next week. No marketing language.
No "leverages", no "robust", no "seamlessly".

DIAGRAM:
<paste .puml source>
```

Generating the description from the diagram rather than the plan catches drift — if the model can't describe the diagram coherently, the diagram is wrong.

---

## 9. Agent harnesses

Ollama ships first-class launchers that point existing agent tooling at a local model:

```bash
ollama launch claude   --model gemma4:31b    # Claude Code, local backend
ollama launch opencode --model gemma4:31b
ollama launch codex    --model gemma4:31b
```

This gives you a full agentic loop — read files, edit files, run `dotnet test` — with weights on your own hardware. For work under clearance, that distinction is the whole point.

### Making the agent loop actually work

- **Use `31b`.** The Tau2 gap (76.9% vs 68.2%) is precisely the gap in agentic reliability. A 26B agent will more often stall or take a wrong tool action.
- **Set `num_ctx` to at least 32768.** Agent harnesses burn context on tool schemas and file reads before you've said anything.
- **Give the agent a `CLAUDE.md` / `AGENTS.md`** with your architecture constraints. Repetition in the repo beats repetition in the prompt.
- **Cap the blast radius.** Local models are more prone to over-editing. Ask for one file at a time. Commit between steps.
- Gemma 4 has native function calling and structured JSON output, so tool use is a first-class path rather than a prompt hack.

Google also publishes an official **Gemma Skills repository** — a library of skills aimed at agents building with Gemma. Worth a look before you write your own harness glue.

### IDE integration

Continue, Cline, and Roo all speak to `http://localhost:11434`. Configure:

```json
{
  "model": "gemma4:31b",
  "provider": "ollama",
  "apiBase": "http://localhost:11434",
  "completionOptions": { "temperature": 1.0, "topP": 0.95, "topK": 64 },
  "contextLength": 32768
}
```

For inline autocomplete specifically, use `gemma4:e4b` or `12b` — a 31B dense model is too slow for keystroke-latency completion, and completion doesn't need a reasoner.

---

## 10. Known failure modes

| Failure | Why | Mitigation |
|---|---|---|
| **Hallucinated NuGet packages / versions** | Training cutoff + plausible-name generation | Put "do not assert package versions; write `verify: <pkg>`" in the system prompt. Check every package against nuget.org before `dotnet add`. |
| **Angular version drift** — emits `*ngIf`, NgModules, decorator `@Input()` | Older Angular dominates training data | Pin the version in the system prompt AND paste one current component as an exemplar. The exemplar does more work than the instruction. |
| **EF Core API drift** | Same | Paste your `DbContext` and one existing `IEntityTypeConfiguration<T>`. |
| **Silent context truncation** | Ollama's default `num_ctx` is far below the model's window | Always set `num_ctx` explicitly. Verify with `ollama ps`. |
| **Repetition loops** | Temperature clamped too low, or `repeat_penalty` fighting the sampler | Return to `temperature 1.0`, `repeat_penalty 1.0`. Gemma dislikes aggressive repetition penalties. |
| **Thinking content leaking into history** | Your client is echoing the thought channel back | Strip the `<|channel>thought` block from assistant messages before the next turn. Explicitly required by the model card. |
| **Architecture that ignores the plan** | Whole plan pasted; model attends to the wrong section | Paste only the relevant excerpt. |
| **Confident wrongness about Azure SDKs** | Fast-moving surface area, thin training coverage | Local models are the wrong tool for "what's the current API for X." Use them for structure; verify surface details against docs. |
| **Degradation past ~64K context** | MRCR: 66.4% @ 128K for the best model in the family | Curate. A map beats a dump. |

---

## 11. Where local Gemma 4 is genuinely strong — and where it isn't

**Strong:**
- Architecture reasoning and decision logs. `think: "high"` produces real tradeoff analysis, not a listicle.
- PlantUML and structural artifacts, especially inside a generate → validate → correct loop.
- Idiomatic C# once anchored by an exemplar.
- Test generation. Give it the implementation, ask for xUnit + FluentAssertions. This is where it earns its keep.
- Anything where the weights being on your machine is a hard requirement.

**Weak:**
- Current API surfaces of fast-moving libraries.
- Very long-horizon agentic tasks without checkpoints.
- Whole-repo reasoning. It will confidently answer; it will not have read everything.
- Precise package/version facts.

**The honest framing:** a 31B model at LiveCodeBench 80% and Codeforces ELO 2150 is a genuinely capable engineer with an imperfect memory and no internet. Structure the work the way you'd structure work for exactly that person — clear specs, one task at a time, a compiler as the arbiter, and review before merge.

---

## 12. Quick reference

```bash
# Setup
export OLLAMA_FLASH_ATTENTION=1
export OLLAMA_KV_CACHE_TYPE=q8_0
export OLLAMA_KEEP_ALIVE=30m
ollama pull gemma4:31b

# Build the specialists
ollama create arch-gemma   -f Modelfile.arch
ollama create dotnet-gemma -f Modelfile.dotnet
ollama create ng-gemma     -f Modelfile.ng

# Architecture, thinking hard, structured
curl -s localhost:11434/api/chat -d '{
  "model":"arch-gemma","stream":false,"think":"high",
  "options":{"temperature":1.0,"top_p":0.95,"top_k":64,"num_ctx":65536},
  "format":{ ... },
  "messages":[{"role":"user","content":"..."}]
}' | jq -r '.message.content'

# Agentic
ollama launch claude --model gemma4:31b
```

| Setting | Value |
|---|---|
| temperature | 1.0 |
| top_p | 0.95 |
| top_k | 64 |
| num_ctx | 32768 (code) / 65536 (architecture) |
| think | low → max, by task |
| Model | `31b` architecture & agents · `26b` speed · `12b` laptop · `e4b` autocomplete |

---

## Appendix: the phase-② prompt, ready to paste

```
<|think|>
Assume nothing you have not been told. Where you must assume, say so.

Produce an architecture plan for the requirement below. Target: .NET 8,
Clean Architecture, CQRS, EF Core, SignalR, Redis, Azure. Frontend: Angular,
standalone components, signals.

Sections, in order, with these exact headings:

## Assumptions
Numbered. Mark load-bearing assumptions with [LOAD-BEARING].

## Solution Structure
A table: Project | Layer | Purpose | Depends On.
Dependencies must point inward. If the requirement forces an outward
dependency, say so and explain how you inverted it.

## Domain Model
Per aggregate: root, entities, value objects, invariants, domain events.
State each invariant as a predicate that must hold after every operation.

## Application Layer
A table: Use Case | Command/Query | Handler | Validator | Failure Modes.

## Persistence
Concurrency strategy. Index plan. Migration approach. Where you chose
eventual consistency and why.

## Real-Time
Hubs, groups, backplane, reconnection and replay semantics.
What happens to a message published while a client is disconnected?

## Cross-Cutting
Authentication, authorization, idempotency, logging, caching, observability.

## Frontend
Feature structure. Where state lives. The API contract as a typed interface.

## Decision Log
Per decision: Decision / Rationale / Consequences / Rejected Alternatives.
At least one rejected alternative per decision. If you cannot name one,
the decision was not a decision.

## Risks
Ordered by likelihood × cost. Each with a mitigation and a trigger that
tells you the risk is materializing.

Write no code. Name concrete types and paths.

REQUIREMENT:
<paste>
```
