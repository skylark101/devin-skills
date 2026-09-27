---
name: push-all
description: Commit a tested fix and push environment-specific branches for selected services across dev, sit, and uat
argument-hint: "[service ...]"
triggers:
  - user
---

Propagate already-tested local fixes for selected services to fresh feature branches based on `dev`, `sit`, and `uat`, then return Azure DevOps PR creation links. Do not change application logic while propagating the fix.

## Supported services

Use only this hardcoded list; do not spend time discovering services:

- `quote-management-service`
- `rating-management-service`
- `policy-management-command-service`
- `policy-management-query-service`
- `sme-document-management-service`

## Service selection

1. Parse `$ARGUMENTS` as one or more service names, accepting spaces or commas as separators.
2. Validate every supplied name against the supported list. Report unknown names and ask the user to correct the selection; never guess.
3. If no valid service names were supplied, use **exactly one** ask_user_question call with **exactly one** multi-select question. Put all five supported services in a single options list. Allow the user to select one or more services; do not require a second section or a second question. If the tool does not support an empty optional section, do not add any extra sections.
4. Process only the selected services. Never automatically include an unselected service.
5. Locate each selected service as either the current repository root, a child of the current workspace directory, or a sibling of the current repository. If a selected directory cannot be found, report it and continue only after asking whether to skip it.

## Safety rules

- The user has already tested the fix. Do not edit, refactor, regenerate, or otherwise change application logic.
- Never force-push, rebase, reset, delete branches/files, bypass hooks, or overwrite an existing local or remote branch.
- Never include secrets or ignored files. Before committing, inspect `git status --short`, staged changes, untracked files, and `git diff --check`.
- Do not mix changes between services. Run all Git commands from the selected service's repository root.
- Preserve the original branch name for each service and return to it after processing that service, including after a failure when it is safe to do so.
- If there is an in-progress merge, rebase, revert, or cherry-pick, stop that service and report it. Do not abort another operation automatically.
- Stop the affected service on conflicts, rejected pushes, failed hooks, unresolved Flyway conflicts, ambiguous parent environment, or an unexpected dirty tree. Do not silently continue.
- A `/push-all` invocation is authorization to create ordinary commits and push the newly prepared branches for the selected services. It is not authorization for destructive Git operations.
- PRs are not submitted automatically. Return Azure DevOps PR creation links, as requested by the user.

## Workflow for each selected service

### 1. Preflight

1. Confirm the directory is a Git worktree and has an `origin` remote.
2. Record:
   - service name;
   - original branch;
   - original `HEAD`;
   - complete `git status --short`;
   - staged and unstaged diffs;
   - untracked files.
3. Refuse to run from detached HEAD or directly from `dev`, `sit`, `uat`, or `main`; this workflow requires an existing feature/fix branch containing the local work.
4. Run `git fetch origin --prune`. If it fails, stop this service because fresh environment bases cannot be guaranteed.
5. Confirm `origin/dev`, `origin/sit`, and `origin/uat` all exist.
6. Show exactly which local files will be committed. Ask for confirmation if any file appears unrelated to the fix, generated, secret-bearing, or unexpectedly large.
7. Obtain a concise commit message from the user once per service. If the current branch already has unpushed commits in addition to local changes, show them and ask whether all such fix commits belong to this propagation. Never guess the intended commit range.

### 2. Determine the source environment

Determine which environment branch the current feature branch was created from:

1. For each of `origin/dev`, `origin/sit`, and `origin/uat`, test ancestry and calculate the number of commits from its merge base to `HEAD`.
2. Prefer an environment whose current remote tip is an ancestor of `HEAD`; if more than one qualifies, choose the one with the smallest distance to `HEAD` only when it is uniquely nearest.
3. Use branch tracking/reflog information only as supporting evidence.
4. If no environment is a clear unique parent, show the evidence and ask the user to choose `dev`, `sit`, or `uat`. Do not infer from the branch-name suffix alone.

### 3. Derive fresh branch names

1. Derive a valid feature slug from the original branch:
   - remove an initial `feature/`, `fix/`, `bugfix/`, or `hotfix/` prefix;
   - remove one trailing `-dev`, `-sit`, or `-uat` suffix;
   - lowercase it;
   - replace runs of invalid Git branch-name characters or whitespace with `-`;
   - trim leading/trailing separators;
   - validate it with `git check-ref-format --branch`.
2. Use exactly `feature/{slug}-{env}` for each environment.
3. The existing original branch may serve as the source-environment branch even if its name differs from this pattern; do not rename it.
4. For each newly required environment branch, verify that neither the local branch nor `origin/<branch>` exists. If one exists, ask the user for a different valid feature slug. Never reuse, overwrite, or force-push it.

### 4. Prepare and push the source-environment fix

1. Invoke `/check-flyway-dup <source-env> <service>` before committing.
2. If the Flyway skill reports conflicts and proposes renames, wait for the user's explicit approval as required by that skill. Include approved environment-specific renames in the source commit.
3. If the check cannot complete, clearly state why and ask whether to stop. Never proceed with a known or unresolved duplicate.
4. Reinspect the diff to ensure only the intended fix and approved Flyway rename are present.
5. Stage the intended files explicitly, commit with the approved message, and record the resulting source commit SHA. Do not use `git add .` blindly when unrelated files exist.
6. Push the current branch with upstream tracking when needed. Do not force-push.
7. Record an Azure DevOps PR creation link from this branch to the source environment.

### 5. Prepare each other environment

For each of the other two environments, one at a time:

1. Create the fresh `feature/{slug}-{env}` branch directly from the latest `origin/<env>`.
2. Apply the source fix without committing by using `git cherry-pick -n <source-commit>`.
   - If the source work intentionally comprises multiple approved commits, apply the exact approved commit list in chronological order with `-n` so the environment check runs before the environment commit.
   - On a conflict, stop and report the files. Do not auto-resolve and do not abort without user confirmation.
3. Invoke `/check-flyway-dup <env> <service>` while the applied changes are still uncommitted, so that skill can detect changed migration files.
4. If the Flyway skill proposes an environment-specific migration rename, wait for explicit approval and include the approved rename only on this environment branch.
5. Verify that the resulting staged/unstaged patch still represents the same application fix. Flyway version filename differences are allowed; unrelated logic differences are not.
6. Run `git diff --check`. Stage only the intended files and commit using the service's approved message. Preserve authorship information from the source commit where practical without bypassing hooks.
7. Push with `git push -u origin <branch>`. Never force-push.
8. Record an Azure DevOps PR creation link from this branch to this environment.

### 6. Azure DevOps PR creation links

1. Parse the Azure DevOps repository base URL from `git remote get-url origin`, removing embedded username information if present.
2. Prefer a PR-creation URL printed by Azure Repos during `git push` when it targets the correct source and environment.
3. Otherwise construct the standard Azure Repos creation URL from the repository base URL using the `pullrequestcreate` route, with URL-encoded `sourceRef=<feature branch>` and `targetRef=<env>` query parameters.
4. Do not claim a PR was created. Label every URL as **Create PR** because the user selected link-only mode.

### 7. Finish and report

1. Return to the service's original branch when safe.
2. Verify `git status --short` and report any remaining state; never hide leftovers.
3. Continue to the next selected service only if the previous service is in a safe state. If not, ask the user whether to stop the entire run.
4. Final output must be grouped by service and include a table with:
   - environment;
   - target branch;
   - pushed feature branch;
   - commit SHA;
   - Flyway check result;
   - push result;
   - **Create PR** link.
5. Separately list skipped/failed services and the exact manual action needed. Never report success for a branch that was not pushed.
