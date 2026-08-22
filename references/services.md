# Debian Services And External State

Primary sources:

- Package Tracker: https://tracker.debian.org/
- Bug Tracking System: https://bugs.debian.org/
- Salsa GitLab: https://salsa.debian.org/
- Debusine introduction: https://freexian-team.pages.debian.net/debusine/explanation/introduction.html
- Debian CI: https://ci.debian.net/
- buildd: https://buildd.debian.org/
- mentors: https://mentors.debian.net/
- Debian Package Search: https://packages.debian.org/

## Package Tracker

Use tracker.debian.org for package overview:

- versions by suite
- migration status
- testing excuses
- RC bugs and open bugs
- autopkgtest status
- buildd status
- watch/upstream information
- VCS links
- maintainer/team subscription info

Pattern:

```text
https://tracker.debian.org/pkg/<source-package>
```

Check the tracker before version bumps, uploads, RC bug work, migration debugging, and QA triage.

## Debian BTS

Use bugs.debian.org for bug state and control commands. The web pattern is:

```text
https://bugs.debian.org/<bug-number>
https://bugs.debian.org/src:<source-package>
```

For changelog closures, use `Closes: #NNNNNN`.

When the user mentions a bug number, verify it in BTS before changing code or changelog:

```text
https://bugs.debian.org/<bug-number>
```

Check:

- package/source package matches the repository
- bug is not already closed in the target version
- severity/tags/merged bugs/forwarded upstream state
- found and fixed versions
- attached patches, PO files, logs, or maintainer comments
- whether `Closes: #NNNNNN` is appropriate for this upload

If the user's summary conflicts with BTS, report the mismatch and avoid adding a false closure.

Common BTS concepts:

- severity: wishlist, minor, normal, important, serious, grave, critical
- tags: patch, pending, upstream, fixed-upstream, ftbfs, help, moreinfo
- affects, blocks, blocked-by
- found/fixed versions
- usertags for teams

Triage state changes go by mail to `control@bugs.debian.org`, or as `Control:`
pseudo-headers at the top of a reply to `NNNNNN@bugs.debian.org`. Common
commands:

```text
severity NNNNNN important
tags NNNNNN + moreinfo upstream
reassign NNNNNN <package> <version>
forwarded NNNNNN <upstream-bug-url>
found NNNNNN <version>
fixed NNNNNN <version>
merge NNNNNN MMMMMM
```

Mailing the BTS leaves the workspace — confirm first (see `SKILL.md`). When
not asked to send, state the intended control commands in the summary rather
than pretending they happened.

## Salsa

Use salsa.debian.org for Debian GitLab packaging repositories, merge requests, issues, CI, and team maintenance.

Common checks:

- repository default branch
- branch naming and DEP-14 layout
- merge requests related to the bug/task
- CI pipeline failures
- tags matching Debian uploads
- pristine-tar/upstream branches

Do not push, create merge requests, or modify remote state unless the user explicitly asks and credentials/approval are available.

## Debusine

Debusine is a Debian-oriented automation system for package workflows, QA, and artifact tracking. Use it as an external build/test/analysis service when local reproduction is expensive or when the user points to a debusine workspace/work request.

When working with debusine:

- Identify workspace, work request, artifact IDs, collection, and suite.
- Read logs/artifacts first; do not rerun jobs blindly.
- Map debusine failures back to Debian packaging actions: build-deps, tests, lintian, piuparts, reproducibility, or archive policy.
- Preserve links and IDs in summaries so the maintainer can audit the result.

If debusine CLI/API credentials are not configured, use the web links and ask before attempting authenticated actions.

Debusine upload safety:

- Debusine may accept unsigned or intermediate work-in-progress source uploads for CI/QA assistance; treat this as different from uploading to the Debian archive.
- The `dput` rules in `tools.md` (*Upload Helpers*) apply: explicit profile, explicit `.changes` file, never a bare or probing invocation.
- Before uploading, print or inspect the `.changes` file enough to verify source, version, distribution, changed-by, and file list.
- Preserve the created artifact/work-request URLs in the final summary.

Archive upload safety:

- Do not upload to ftp-master or any archive profile unless the user explicitly asks for that exact target.
- Do not sign, upload, or retry archive uploads after a GPG/profile error without explicit confirmation.
- Prefer asking when `dput` output indicates it selected a target profile different from the one requested.

## buildd And Debian CI

Use buildd for architecture-specific build failures and logs:

```text
https://buildd.debian.org/status/package.php?p=<source-package>
```

Use ci.debian.net for autopkgtest regressions and migration blockers. Compare failures across architectures and triggered versions before changing packaging; many failures are caused by dependencies or testbed changes.

## mentors

Use mentors.debian.net for sponsored uploads and review context, especially for new maintainers or packages not directly uploaded by the user.

Check whether a `.dsc` or source package there differs from the git repository before advising changes.
