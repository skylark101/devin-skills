---
name: check-flyway-dup
description: Check new or modified Flyway migrations for version conflicts against the DB, git remote, and local files
argument-hint: "[env] [service]"
---

Check **new or modified** Flyway migrations in the current service for version conflicts. This skill focuses on the code that was just generated or changed, not on old migrations whose DB script names may differ for historical reasons. It always reports first and only renames files after you explicitly approve.

1. Identify the target service:
   - If a service name is provided as an argument (e.g. `/check-flyway-dup dev policy-management-command-service`), use it directly.
   - Otherwise, find the nearest parent directory that contains `pom.xml` and matches one of: `quote-management-service`, `rating-management-service`, `policy-management-command-service`, `policy-management-query-service`, `sme-document-management-service`.
   - If the current directory is not inside any of these service roots and no service name was given, list the detected service directories and ask the user which one to check.

2. Determine arguments:
   - The first argument, if it matches a known Spring profile/env (`dev`, `sit`, `uat`, `perf`, etc.), is the environment. Default env is `dev`.
   - The remaining argument, if any, is treated as the service name.
   - Examples:
     - `/check-flyway-dup` → env `dev`, auto-detect service.
     - `/check-flyway-dup sit` → env `sit`, auto-detect service.
     - `/check-flyway-dup policy-management-command-service` → env `dev`, service `policy-management-command-service`.
     - `/check-flyway-dup sit policy-management-command-service` → env `sit`, service `policy-management-command-service`.

3. Locate migrations:
   - Check `src/main/resources/db/migration/` under the service root.
   - If the directory does not exist or contains no `V*.sql` files:
     - Report "No Flyway migrations found in this service" and stop.
     - (For `policy-management-query-service`, this is expected because Flyway is disabled there.)

4. Identify candidate files (the code that is actually being worked on):
   - Run `git status --short src/main/resources/db/migration/`.
   - Candidate files are those with statuses `??` (untracked), `A` (added), `M` (modified), or `R`/`C` (renamed/copied) in the migration directory.
   - If git is not available or `git status` shows no migrations, stop and ask the user: "No new or modified migrations detected. Run a full scan of all migrations instead? (yes/no)".

5. Extract all local versions for context:
   - For every `V<version>__<description>.sql` file in `src/main/resources/db/migration/`, parse the version string (the part after `V` and before `__`).
   - Build a map of `version → [all local filenames]`.
   - This is needed to detect local duplicates involving the candidate files.

6. Read DB config from `src/main/resources/application-{env}.properties`:
   - `spring.datasource.url`
   - `spring.datasource.username`
   - `spring.datasource.password` (read but never print)
   - `spring.flyway.table` (default `flyway_schema_history`)
   - `spring.flyway.schemas` (default `MEDICAL_SME`)
   - Do not echo the password or full connection string in the final report.

7. Query the Flyway history table (primary check):
   ```sql
   SELECT version, script, installed_on, success
   FROM <schema>.<flyway_table>
   WHERE success = 1
   ORDER BY installed_rank;
   ```
   - Quote the table name with square brackets if needed, e.g. `[MEDICAL_SME].[flyway_schema_history_policy_cmd]`.
   - Try running the query with whatever tool is available: `sqlcmd`, `python` + `pyodbc`/`pymssql`, or `mvn` with a temporary query bean.
   - If the DB is unreachable, clearly report the connection failure and continue to step 8.

8. Git fallback check (run if DB was NOT reachable, or optionally alongside DB for extra safety):
   - Make sure you are inside a git repository. If not, skip this step.
   - Try `git fetch --all` or `git fetch origin` first to get latest remote refs. If network is unavailable, continue with already-fetched refs.
   - Search all local and remote refs for migration files that match the versions of the candidate files:
     ```bash
     git ls-tree -r HEAD -- src/main/resources/db/migration/
     git ls-tree -r origin/HEAD -- src/main/resources/db/migration/
     git ls-tree -r --remotes -- src/main/resources/db/migration/
     ```
     Or use `git log --all --name-only --pretty=format: -- src/main/resources/db/migration/` and grep for the versions.
   - For each candidate version, check if any branch/commit/ref has a migration file with the same version but a different filename.
   - Report any matches as "version exists in git history/remote".

9. Detect conflicts only for candidate files:
   - **Local duplicate**: A candidate file's version appears in more than one local file (i.e. the version map has >1 filename). This includes when the candidate itself collides with another existing local migration.
   - **DB conflict**: A candidate file's version exists in the DB history table with a different `script` name than the candidate filename. Ignore DB rows whose script name differs from the candidate; do not flag every historical rename.
   - **Git remote conflict**: A candidate file's version exists in any git ref (local or remote) with a different filename.
   - A candidate version exists in the DB with the same script name → OK, not a conflict.

10. Report:
    - Target service and environment.
    - Candidate files detected (the new/changed migrations).
    - Total local migrations.
    - Local duplicate versions (if any).
    - DB reachable? (yes/no). If no, show the connection failure reason.
    - Git fallback result: reachable refs checked, any version found in git history/remote.
    - Table of conflicts: version | candidate filename | local duplicate? | DB script | git source | issue.
    - If no candidate files and user did not request a full scan, report "No new or modified migrations to check" and stop.
    - If no conflicts, report "No conflicts found in new/modified migrations" and stop.

11. Propose fixes (do NOT apply yet):
    - Compute `nextVersion = max(all DB versions, all local versions, all git-detected versions) + 1` as a number.
    - For each conflicting candidate file, propose a rename from `V<old>__desc.sql` to `V<nextVersion>__desc.sql` (bump `nextVersion` by 1 for each additional conflict).
    - Show the proposed renames and ask: "Approve these renames? (yes/no)"

12. Only if the user explicitly replies with "yes", "approve", "fix", or "rename":
    - Rename each conflicting candidate file.
    - Verify the new filenames exist.
    - Report the final filenames.

Rules:
- Never modify the Flyway history table or run any write SQL against the DB.
- Never rename a file unless the user has explicitly approved the proposed rename.
- Focus only on new or modified migrations; do not flag old historical DB renames as conflicts.
- If the DB cannot be reached, still report local duplicate-version conflicts and git-detected conflicts for candidate files.
- Do not print secrets (passwords, connection strings) in the output.
- Be clear about what was checked and what could not be checked.
