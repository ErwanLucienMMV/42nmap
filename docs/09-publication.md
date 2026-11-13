# Documentation publication

The user replaced the original **13 November 2026, 09:00 Europe/Paris** schedule
with an instruction to commit and push immediately. The scheduled task is
cancelled as part of immediate publication. Use the actual commit timestamp;
do not fabricate a commit date.

## Scope and local task files

The publishing helper lives under `.local/publication/`, which is ignored by
Git. Its manifest names only `README.md`, `.gitignore`, and the nine `docs/*.md`
files from this update, with their SHA-256 hashes. It excludes the source PDF,
rendered pages, scripts, logs, and any future scanner implementation.

Original task name: `42nmap-20261113-0900-documentation`.

The helper is designed to:

1. Honor the publication time in its local manifest, updated for immediate execution.
2. Confirm the repository, expected branch (`main`), origin URL, starting HEAD,
   and every listed file's hash; stop if they changed.
3. Refuse unrelated staged changes rather than commit them accidentally.
4. Verify remote access without interactive authentication prompts.
5. Stage and commit only the allowlisted documentation paths.
6. Push `main` normally to the existing `origin`, without force or rebasing.
7. Save the commit ID and log so a failed push can be retried without duplicating
   the commit. A success marker makes subsequent runs a no-op.

Git has an email configured but no author name at preparation time. The helper
uses the repository owner's GitHub login `ErwanLucienMMV` as a command-local
fallback author name; it does not change global Git configuration. An author
name configured before execution takes precedence.

## Publication inspection

Immediate publication needs working GitHub authentication and network access.
The local success marker is written only after the pushed commit is verified
against the remote branch. Inspect that record and Git history for the result.

From a normal PowerShell session in the repository:

```powershell
Get-Content -LiteralPath '.local/publication/publish.log'
Get-Content -LiteralPath '.local/publication/published.json'
git log -1 --format=fuller
git ls-remote origin refs/heads/main
```

To retry a failed push, inspect the log and fix the specific problem,
then invoke the same helper from this repository:

```powershell
& '.local/publication/publish.ps1'
```

If a documentation hash or starting HEAD changed, the helper stops for review.
Do not bypass that check blindly; review the changes and commit the intended
documentation manually. Never force-push to repair a remote divergence.
