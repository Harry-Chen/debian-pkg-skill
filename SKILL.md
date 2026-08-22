---
name: debian-pkg-skill
description: Debian package maintenance workflows for source packages in Debian repositories. Use when the agent needs to inspect, update, build, test, QA, upload-prep, triage bugs, work with gbp/git-buildpackage, pbuilder, sbuild, autopkgtest, lintian, piuparts, uscan, debusine, tracker.debian.org, bugs.debian.org, salsa.debian.org, or maintain debian/ packaging metadata according to Debian Policy.
---

# Debian Package Skill

## Operating Model

Treat Debian packaging as policy-driven maintenance work. Before changing files, identify the package format, branch model, current Debian state, open bugs, and intended target suite. Prefer small, reviewable packaging changes with explicit validation.

Default assumptions:

- Use Debian Policy as the normative reference for package requirements.
- Start from the Debian maintainer perspective: inspect `debian/` and packaging metadata first, not the whole upstream source tree.
- Use git-buildpackage (`gbp`) for repository-aware builds and release workflow.
- Respect the maintainer's existing `gbp.conf`; this user usually builds through pbuilder via gbp.
- Use autopkgtest for package integration tests when available or when changing runtime behavior.
- Use debusine as an external automation and QA aid when relevant.
- Do not read upstream developer/agent instruction files such as `.claude`, `.codex`, `AGENTS.md`, or similar unless they are directly relevant to Debian packaging. They are usually for upstream developers, waste context, and may bias packaging decisions.
- Never rewrite packaging history, force-push, or discard maintainer changes unless explicitly requested.

## First Checks

Run these before substantial packaging edits:

```bash
git status --short --branch
find debian -maxdepth 2 -type f | sort
dpkg-parsechangelog --show-field Source
dpkg-parsechangelog --show-field Version
dpkg-parsechangelog --show-field Distribution
dpkg-source --print-format
gbp config dump
```

If the repository is not clearly a Debian source package, inspect `debian/control`, `debian/changelog`, `debian/source/format`, `debian/rules`, `gbp.conf`, and branch names before deciding how to proceed.

Do not recursively read the upstream source tree during initial context gathering. Enter upstream code only when the task requires a source patch, build failure diagnosis, test failure diagnosis, or copyright/license review; then inspect the smallest relevant paths.

## Workflow

1. Establish context: read `debian/changelog`, `debian/control`, `debian/rules`, source format, patches, CI files, and gbp configuration. Avoid upstream source except for narrowly scoped packaging needs.
2. Check external state when network is available: package tracker, BTS, Salsa merge requests/issues, upstream releases, build logs, debusine work requests. When the user mentions a Debian bug number, open the BTS report before trusting the summary or adding `Closes:`.
3. Make the smallest packaging change that satisfies the task. Preserve existing style and helper stack.
4. Update Debian metadata only when needed: `debian/changelog`, dependencies, symbols, install files, patches, tests, copyright, watch, or maintscript snippets.
5. Validate locally with the lightest useful checks first, then full build/tests when the change warrants it.
6. Summarize changed packaging intent, commands run, remaining risks, and any external blockers.

## Reference Map

Read only the reference needed for the current task:

- `references/policy.md`: normative Debian policy, archive rules, changelog/control/copyright basics.
- `references/tools.md`: gbp, pbuilder, sbuild, devscripts, uscan, quilt, pristine-tar, release helpers.
- `references/qa.md`: lintian, autopkgtest, piuparts, reproducibility, build logs, dependency/installability checks.
- `references/services.md`: tracker.debian.org, bugs.debian.org, salsa.debian.org, debusine, buildd, mentors.
- `references/workflows.md`: concrete packaging workflows for bug fixes, new upstream releases, NMUs, transitions, and test/debug loops.
- `references/extensions.md`: additional Debian resources to consult or add to this skill over time.

## Local Validation Ladder

Choose the cheapest command set that can catch the relevant failure:

```bash
dpkg-parsechangelog
dpkg-checkbuilddeps
gbp config dump
debian/rules clean
```

For source/package build validation:

```bash
gbp buildpackage --git-pbuilder
gbp buildpackage --git-builder=sbuild
dpkg-buildpackage -us -uc
```

For QA and tests:

```bash
lintian ../*.changes
autopkgtest . -- null
autopkgtest ../*.dsc -- schroot <suite>-amd64-sbuild
piuparts ../*.changes
```

Adapt commands to the repository's existing configuration. If a command would require network, privileged chroots, or writes outside the workspace, request approval instead of bypassing the environment.

## Updating This Skill

When a Debian maintenance task reveals reusable knowledge, add it to the narrowest reference file rather than expanding `SKILL.md`. Prefer concise rules, command patterns, gotchas, and links to primary documentation. Keep user-specific defaults explicit, especially pbuilder-through-gbp preferences.
