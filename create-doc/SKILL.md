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
       └── README.md
   ```

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

## README content

Use the following order, omitting sections that genuinely do not apply:

1. **Title** — human-readable feature name.
2. **At a glance** — 2–4 sentences describing what triggers the feature, why it exists, and its observable result.
3. **Who/what uses it** — actor, upstream caller, scheduled job, listener, or internal caller.
4. **Contract** — for APIs include method/path, important headers, request, response, and meaningful status codes; for functions/features include inputs, outputs, and side effects.
5. **How it works** — numbered end-to-end flow using real component and operation names.
6. **Flow diagram(s)** — Mermaid diagrams following the rules below.
7. **Business rules and validation** — only meaningful rules, with resulting behavior.
8. **Data and integrations** — reads/writes, transactions, messages/events, and external calls.
9. **Failures and edge cases** — confirmed rejection, exception, fallback, rollback, retry, and partial-success behavior.
10. **Code map** — compact table: component/file, responsibility, and why it matters to this feature.
11. **Open questions** — only unresolved facts that materially affect understanding.

## Diagram rules

1. Embed Mermaid directly in `README.md` so it renders with the documentation.
2. Choose diagrams based on the actual feature:
   - `flowchart` for decisions, validation, branching, jobs, and business workflow;
   - `sequenceDiagram` for API/service/database/external call ordering;
   - `stateDiagram-v2` only when the code contains meaningful state transitions.
3. Every node and arrow must correspond to code-confirmed behavior. Use actual component names or clear business labels backed by code.
4. Show the meaningful happy path plus important validation/failure branches. Do not clutter diagrams with trivial getters, mappers, logging, or framework plumbing.
5. Label arrows with meaningful actions or data, especially for async messages, persistence, and external calls.
6. For a simple feature, use one diagram.
7. For a complicated feature, create multiple diagrams in the same README rather than one unreadable diagram:
   - begin with an overview diagram;
   - follow with focused diagrams for subflows such as validation, persistence, external integration, async processing, or failure handling;
   - introduce each focused diagram with a sentence linking it to the relevant numbered step in `How it works`;
   - reuse consistent component names across diagrams.
8. Keep each diagram readable: target roughly 5–12 meaningful nodes/participants. Split it when branches or participants obscure the primary path.
9. Do not use decorative diagrams or a generic `Client → Controller → Service → Database` flow unless that is genuinely the complete meaningful behavior.
10. Validate Mermaid syntax manually before finishing: balanced blocks, valid identifiers, quoted labels where needed, and matching `alt`/`else`/`end` or `subgraph`/`end` structures.

## Evidence and accuracy review

Before writing, maintain a working list of claims and their source files. Before finishing:

1. Re-read the entry point and each file used in the documented primary flow.
2. Confirm that every diagram edge and numbered flow step has source evidence.
3. Check that API contracts match code annotations/models rather than assumptions.
4. Check that failure behavior comes from code, exception handling, or tests.
5. Remove unrelated architecture details and repetitive prose.
6. Ensure a non-developer can understand `At a glance` and `How it works` without reading the code.
7. Ensure a developer can use `Code map` and repository-relative references to find the implementation quickly.
8. Report the created or updated README path, the resolved entry point, diagrams included, and any material open questions.
