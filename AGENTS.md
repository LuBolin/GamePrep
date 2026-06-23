# AGENTS.md

## Purpose

This repository is a long-running research, synthesis, and curriculum-building project for game development and Unreal Engine interview preparation.

`.codex/research.md` is the master scope.

The agent’s job is to turn that scope into a useful, structured learning package.

The agent may reorganise files, folders, sections, and deliverables when doing so improves clarity, maintainability, or learning value.

Do not silently narrow the scope.

---

## Required Reading Before Each Run

Before doing work, read:

1. `.codex/research.md`
2. `AGENTS.md`
3. `.codex/STATUS.md`
4. existing source index or source notes if present


If source tracking exists in multiple formats, consolidate it into the simplest useful format.

---

## Main Principle

Prioritise useful learning content over metadata maintenance.

A good run should usually produce or improve real study material, such as:

* topic explanations
* interview answer patterns
* debugging workflows
* performance workflows
* system design frameworks
* hands-on tasks
* flashcards
* knowledge graph nodes
* study plans

Metadata exists to support the curriculum. Metadata is not the product.

---

## Scope Preservation

The final project must eventually cover the full scope of `.codex/research.md`, including:

* Unreal Engine core systems
* standard C++
* UE C++ idioms
* game programming patterns
* game architecture
* game system design
* algorithms and data structures
* game maths
* rendering
* networking
* AI and pathfinding
* ECS and MassEntity
* Lua
* C#
* tools
* build systems
* asset pipelines
* debugging
* profiling
* optimisation
* interview practice

The agent may merge, split, rename, or rearrange topics, but must preserve all meaningful scope.

If restructuring changes the mapping from `.codex/research.md` to output files, update `.codex/STATUS.md` with the new mapping.

---

## Structure Autonomy

The agent may choose the repository structure.

The agent is encouraged to organise the project into a clean, navigable structure rather than creating many tiny top-level files.

The agent may:

* create folders
* merge overlapping files
* split oversized files
* rename files for clarity
* replace exact-file deliverables with equivalent grouped deliverables
* convert heavy JSON metadata into lighter Markdown if that improves workflow
* keep JSON only where machine-readability is genuinely useful

The agent must not:

* delete useful content
* hide unfinished scope
* collapse many topics into vague umbrella sections
* create empty scaffold files unless they are immediately useful
* keep producing structure changes instead of learning content

When restructuring, prefer one bounded reorganisation pass, then return to content production.

---

## Working Method

Use a rolling research-and-synthesis workflow.

For each bounded topic cluster:

1. Define the cluster scope.
2. Identify prerequisites and dependent topics.
3. Collect enough high-quality sources to write accurately.
4. Write useful synthesis before moving to unrelated lower-priority clusters.
5. Add interview questions, answer patterns, flashcards, and hands-on tasks.
6. Update source notes only enough to support traceability.
7. Update `.codex/STATUS.md`.
8. Stop at a clean checkpoint.

Do not collect sources for the entire project before writing.

Do not allow source collection, indexing, or graph maintenance to become a substitute for synthesis.

---

## Source Policy

Use sources, but keep source tracking lightweight.

Prefer sources in this order:

1. Official documentation
2. Official API documentation
3. Engine/source references where available
4. Official talks or conference presentations
5. Studio engineering blogs
6. Reputable technical articles
7. Books and established references
8. Interview anecdotes and forum posts only as supplementary evidence

Every major factual claim should be traceable to a source.

For each source, record at least:

* source ID
* title
* URL
* source type
* relevant topics
* version notes
* where it is used

A Markdown source index is acceptable.

A JSON source manifest is optional. Use JSON only if it clearly improves the workflow.

Do not spend an entire run only maintaining source metadata unless the user explicitly asks for that.

---

## Writing Standards

For each topic, aim to include:

* what it is
* why it exists
* how it works
* what a 3-year engineer should know
* what is specialist depth
* common bugs
* common misconceptions
* debugging workflow
* profiling workflow, if relevant
* system design implications
* strong interview answer pattern
* hands-on verification task
* source IDs

Keep writing practical and interview-oriented.

Avoid shallow filler.

When uncertain, mark uncertainty instead of guessing.

Label information as needed:

* UE4-era
* UE5-specific
* UE5.3+
* UE5.4+
* UE5.5+
* UE5.6+
* deprecated
* experimental
* plugin-dependent
* anecdotal
* version-sensitive

---

## Knowledge Graph

Maintain a knowledge graph if useful.

The graph should show:

* prerequisites
* related topics
* priority
* expected depth
* role relevance
* common bugs
* debugging workflows
* hands-on tasks
* interview questions

The graph does not need to be updated on every run unless the current work changes topic structure meaningfully.

Do not let graph maintenance block useful writing.

---

## STATUS.md

Use `.codex/STATUS.md` as a lightweight project memory.

It should track:

* current active cluster
* last run summary
* current goal
* next recommended step
* completed areas
* partial areas
* blocked areas
* broad coverage map
* important restructuring decisions

The coverage tracker should be living, not fixed.

The agent may split, merge, rename, or rearrange coverage rows if it preserves scope and improves clarity.

Do not maintain a huge detailed matrix if it slows down content production.

---

## Run Behaviour

For each run:

1. Read the required files.
2. Identify the next useful bounded unit of work.
3. Prefer producing real content.
4. Collect only the sources needed for that unit.
5. Update relevant files.
6. Update `.codex/STATUS.md`.
7. Stop at a clean checkpoint.

A run may be a reorganisation run, but only when structure is actively blocking progress.

After a reorganisation run, the next run should produce content.

---

## Usage / Context Limit Safety

If approaching context, output, tool-call, execution, or Codex usage limits:

1. Stop early.
2. Save partial work.
3. Update `.codex/STATUS.md`.
4. State the next recommended step.

Prefer a clean checkpoint over an unfinished large section.

Do not start a large new section if there may not be enough budget to finish and checkpoint it.

---

## Definition of Done

Do not claim the project is complete unless:

* all major scope areas in `.codex/research.md` are covered
* the output structure is coherent and navigable
* topic files contain real content, not just scaffolds
* major claims are source-traceable
* interview questions exist
* flashcards exist
* hands-on projects exist
* study schedules exist
* gap analysis exists
* version-sensitive, deprecated, experimental, and plugin-dependent items are labelled
* `.codex/STATUS.md` confirms full cov