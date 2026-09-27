---
name: create-doc
description: Create concise, evidence-based documentation and meaningful flow diagrams for one API, function, or feature in the current project
argument-hint: "<API path, function, method, or feature>"
triggers:
  - user
---

Create focused documentation for exactly one user-supplied API, function, method, or feature. The required input is `$ARGUMENTS`.

## Input requirement

1. Accept inputs such as:
   - `/api/v1/policies`
   - `createPolicy`
   - `MemberValidationService.validateMember`
   - `census upload validation`
2. If `$ARGUMENTS` is empty, stop and ask the user for an API path, function/method name, or feature description. Do not scan or document the whole codebase.
3. Treat the input as the strict documentation scope. Follow dependencies only as far as needed to explain that feature's real behavior.
4. If the input matches multiple unrelated entry points, show the likely matches with file paths and ask the user to select one. Never guess silently.
5. If no implementation can be found, report what was searched and ask for a more precise identifier. Do not generate speculative documentation.

## Locate the project and feature

1. Use the nearest Git repository root as the project root. If Git is unavailable, use the nearest parent containing the project's primary build manifest, such as `pom.xml`, `build.gradle`, `package.json`, `pyproject.toml`, `go.mod`, or `Cargo.toml`.
2. Resolve the supplied input using the appropriate evidence:
   - API path: route/controller annotations, router registrations, OpenAPI declarations, gateway mappings, and tests.
   - Function or method: declarations, callers, implementations, interfaces, overrides, and tests.
   - Feature description: domain terms, entry points, services, handlers, jobs/listeners, persistence, integrations, and tests.
3. Identify the concrete entry point before writing documentation.
4. Trace the actual execution path from the entry point through relevant orchestration, validation, domain logic, persistence, events/messages, and external integrations.
5. Inspect tests and configuration when they reveal behavior, edge cases, feature flags, transactions, retries, or environment-dependent paths.
6. Distinguish code-confirmed behavior from inference. Do not present naming-based assumptions as facts.

## Analysis boundaries

- Document one feature, not the repository architecture.
- Include only components that participate in or materially affect the selected feature.
- Stop tracing utility/framework internals once their role is clear.
- Preserve exact API paths, class/function names, status codes, event names, table/entity names, and external system names found in the code.
- Never invent runtime ordering, asynchronous behavior, retries, rollback behavior, database changes, or error responses.
- If an important behavior cannot be confirmed, state it briefly under `Open questions` instead of filling the gap.

## Output location

1. Create or update this structure under the target project's root:

   ```text
   documentation/
   └── <feature-slug>/
       ├── README.md
       └── diagrams/
           ├── 01-overview.html
           ├── 02-validation.html
           └── 03-error-handling.html
   ```

   Every feature MUST produce at least one standalone diagram file under `diagrams/` in addition to any Mermaid content in `README.md`. Prefer Archify-generated interactive HTML (dark/light theme, pan/zoom, export built in). Fall back to SVG only when Archify is unavailable.

2. Derive `<feature-slug>` from the resolved feature, not blindly from the raw input:
   - use lowercase kebab-case;
   - remove URL parameter syntax and punctuation;
   - keep it short and recognizable;
   - prefer the business capability name over a generic method name when clearly supported by the code.
3. If `documentation/<feature-slug>/README.md` already exists, read it first and update it in place. Preserve still-correct useful content and replace stale or unsupported statements.
4. Do not create documentation outside this feature folder. Do not create a whole-codebase report.

## Documentation style

Write for developers and non-developers together:

- Start with plain language, then introduce technical details.
- Be precise, concise, and concrete.
- Prefer short sections, tables, and bullets over long paragraphs.
- Explain business meaning rather than merely listing classes.
- Avoid generic filler such as "this service handles requests".
- Avoid a fixed empty template. Include only sections supported by the selected feature.
- Keep source references repository-relative and attach them to the claims they support.
- Do not paste large code blocks from the application.
- Do not write a long prose-only document; every feature explanation must be paired with a diagram.

## README content

Use the following order, omitting sections that genuinely do not apply:

1. **Title** — human-readable feature name.
2. **At a glance** — 2–4 sentences describing what triggers the feature, why it exists, and its observable result.
3. **Who/what uses it** — actor, upstream caller, scheduled job, listener, or internal caller.
4. **Contract** — for APIs include method/path, important headers, request, response, and meaningful status codes; for functions/features include inputs, outputs, and side effects.
5. **How it works** — numbered end-to-end flow using real component and operation names.
6. **Flow diagram(s)** — Mermaid diagrams following the rules below. A README without at least one meaningful Mermaid diagram is incomplete.
7. **Business rules and validation** — only meaningful rules, with resulting behavior.
8. **Data and integrations** — reads/writes, transactions, messages/events, and external calls.
9. **Failures and edge cases** — confirmed rejection, exception, fallback, rollback, retry, and partial-success behavior.
10. **Code map** — compact table: component/file, responsibility, and why it matters to this feature.
11. **Open questions** — only unresolved facts that materially affect understanding.

## Diagram rules

1. Every feature MUST produce at least one standalone diagram file under `documentation/<feature-slug>/diagrams/` in addition to any Mermaid diagram embedded in `README.md`. Text-only documentation is not acceptable.
2. Prefer Archify-generated interactive HTML files (`NN-name.html`). Fall back to SVG only when Archify cannot be found or fails after focused repairs.
3. Embed a corresponding Mermaid diagram directly in `README.md` so it renders alongside the prose and is version-control friendly.
4. Choose diagram types based on the actual feature:
   - `flowchart` for decisions, validation, branching, jobs, and business workflow;
   - `sequenceDiagram` for API/service/database/external call ordering;
   - `stateDiagram-v2` only when the code contains meaningful state transitions.
5. Every node, participant, and arrow must correspond to code-confirmed behavior. Use actual component names or clear business labels backed by code.
6. Show the meaningful happy path plus the most important validation/failure branches. Do not clutter diagrams with trivial getters, mappers, logging, or framework plumbing.
7. Label arrows with meaningful actions or data, especially for async messages, persistence, and external calls.
8. Keep diagrams simple enough for a fresher to follow. If a diagram needs more than about 8–10 nodes/participants to tell its story, split it.
9. For a simple feature, one overview diagram file is enough.
10. For a complicated feature, create multiple simplified diagram files under `diagrams/` rather than one crowded diagram. Use numeric prefixes so they appear in reading order:
    - `01-overview.html` — the primary happy path and actors;
    - `02-validation.html` — input checks and decision rules;
    - `03-persistence.html` — database writes/reads and side effects;
    - `04-error-handling.html` — key failure branches and responses;
    - `05-external-integration.html` — external calls and callbacks (if relevant).
    Only create files that are genuinely needed for the feature; do not force every template file.
11. Each standalone diagram should explain one thing clearly. Link related diagrams with short text such as "If validation fails, see `02-validation.html`".
12. Introduce every diagram with a short sentence that links it to the corresponding numbered step in `How it works`. Do not dump a diagram without context.
13. Reuse consistent component names across diagrams so the reader can follow from overview to detail.
14. Do not use decorative diagrams or a generic `Client → Controller → Service → Database` flow unless that is genuinely the complete meaningful behavior.
15. Validate Mermaid syntax manually before finishing: balanced blocks, valid identifiers, quoted labels where needed, and matching `alt`/`else`/`end` or `subgraph`/`end` structures.
16. If a feature naturally separates into distinct phases (for example input validation, core processing, side effects), model each phase explicitly; do not squeeze unrelated concerns into one diagram.

## Generating diagram files

### Preferred engine: vendored Archify

The `devin-skills` repository vendors Archify at `third-party/archify/`. It renders polished interactive HTML diagrams (dark/light theme, pan/zoom, guided story, built-in export) that are much easier for freshers to read than plain SVG.

1. Locate `archify/bin/archify.mjs` by checking, in order:
   - `devin-skills/third-party/archify/` in the parent/sibling directories of the project root (the normal checkout layout puts `devin-skills` next to service repos);
   - `.agents/skills/archify/` or `.devin/skills/archify/` inside the project;
   - `~/.agents/skills/archify/` or `%APPDATA%/devin/skills/archify/`.
2. If found, run `node <path>/bin/archify.mjs doctor` once. If it reports ready, use Archify for all diagram files.
3. For each diagram:
   - Choose the Archify type matching the flow: `sequence` for API call chains, `workflow` for decisions/pipelines, `architecture` for component maps, `dataflow` for data movement, `lifecycle` for state transitions.
   - Read the matching `schemas/` file, `schemas/common.schema.json`, and one `examples/` file from the vendored Archify before authoring.
   - Author a spec JSON under `diagrams/` (e.g. `01-overview.archify.json`) using the traced code facts — real component names, real calls, at most ~10 primary nodes.
   - Validate and repair until it passes:
     ```bash
     node <archify>/bin/archify.mjs validate <type> diagrams/01-overview.archify.json --quality showcase --json
     ```
   - Deliver to HTML:
     ```bash
     node <archify>/bin/archify.mjs deliver <type> diagrams/01-overview.archify.json diagrams/01-overview.html --quality showcase --json
     ```
   - Do not edit a spec after it passes validation; create the next diagram instead.
4. Follow Archify's own authoring rules (one clear main path, sparse labels, semantic relationship labels). Keep the fresher-simplicity limits from this skill — they override Archify's larger layouts.
5. If Archify is not found or `doctor` fails, say so once in the report, note the install command `npx degit tt-a1i/archify/archify devin-skills/third-party/archify`, and continue with the SVG fallback below. Never fail the whole documentation task because Archify is missing.

### Fallback: simple SVG

1. When Archify is unavailable, write a clean, simple SVG by hand. Keep it readable: rectangles/rounded boxes for components, arrows with clear labels, and a short title. Do not produce overly complex hand-written SVG.
2. Alternatively use Mermaid CLI (`mmdc -i diagram.mmd -o diagrams/01-overview.svg`) if installed.
3. Every SVG file must have a matching Mermaid block in `README.md` or a companion `.mmd` file in `diagrams/` so the diagram source remains editable.

## Evidence and accuracy review

Before writing, maintain a working list of claims and their source files. Before finishing:

1. Re-read the entry point and each file used in the documented primary flow.
2. Confirm that every diagram edge and numbered flow step has source evidence.
3. Check that API contracts match code annotations/models rather than assumptions.
4. Check that failure behavior comes from code, exception handling, or tests.
5. Remove unrelated architecture details and repetitive prose.
6. Ensure a non-developer can understand `At a glance` and `How it works` without reading the code.
7. Ensure a developer can use `Code map` and repository-relative references to find the implementation quickly.
8. Verify that the README contains at least one rendered Mermaid diagram and that each diagram is introduced with context.
9. Verify that the `diagrams/` folder contains at least one standalone diagram file (Archify HTML preferred, SVG fallback) for the feature. For complex features, verify there is an overview diagram plus at least one focused subflow diagram.
10. Confirm that standalone diagrams are simple enough for a fresher to understand without reading code.
11. Report the created or updated README path, the resolved entry point, Mermaid diagrams embedded, standalone diagram files created, and any material open questions.
