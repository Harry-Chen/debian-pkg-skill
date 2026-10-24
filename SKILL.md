---
name: debian-pkg-skill
description: "Maintain Debian source packages: prepare releases and revisions, fix bugs and FTBFS, backport or security/stable updates, and validate upload candidates. Use for requests involving debian/ metadata, Debian suites, or packaging tools such as gbp, dpkg-buildpackage, sbuild, pbuilder, lintian, autopkgtest, uscan, or dput."
---

# Debian Package Skill

## Operating Model

Treat Debian packaging as policy-driven maintenance work. Before changing files, identify the package format, branch model, current Debian state, open bugs, and intended target suite. Prefer small, reviewable packaging changes with explicit validation.

This is a harness-neutral instruction set. Use the host's available filesystem,
shell, and web tools; if a check cannot run in the current environment, say so
and give the exact command or evidence still needed. Follow the host harness's
sandbox and approval controls rather than trying to work around them.

Default assumptions:

- Use Debian Policy as the normative reference for package requirements.
- Prefer archive consistency over upstream convenience when the two conflict.
- Start from the Debian maintainer perspective: inspect `debian/` and packaging metadata first, not the whole upstream source tree.
- Use git-buildpackage (`gbp`) for repository-aware changelog generation, upstream imports, patch queue management, and release tagging. Do **not** assume `gbp buildpackage` is the local build path; builder choice is a per-maintainer setting.
- Respect the repository's existing `gbp.conf`. Builder command, default architecture/suite, result and log locations, and upload targets are environment-specific: read `LOCAL.md` in this skill directory when it exists, otherwise inspect the repository configuration and ask before guessing.
- Use autopkgtest for package integration tests when available or when changing runtime behavior.
- Use debusine as an external automation and QA aid when relevant.
- Honor applicable repository and workspace instructions before making changes. When entering an upstream tree, avoid unrelated agent-instruction files unless they govern the packaging task or the host requires them.
- Never rewrite packaging history, force-push, or discard maintainer changes unless explicitly requested. Do not commit unless the task calls for it, do not amend or squash existing commits, and do not bypass hooks with `--no-verify` or `--no-gpg-sign`.
- Do not infer authorization for external mutations. Pushes, Salsa merge requests/comments, BTS mail, and uploads require an explicit request for that exact action; authorization for one does not authorize the others.

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

Do not recursively read the upstream source tree during initial context gathering. Enter upstream code only for a concrete reason: writing or refreshing a quilt patch, diagnosing a build or test failure, reviewing copyright/license changes, checking installed files, generated artifacts, or build-system behavior, or understanding an upstream API/ABI change that affects Debian metadata. Then inspect the smallest relevant paths, preferring targeted search, build logs, patch context, and exact failing paths over broad recursive reads.

## Workflow

1. Establish context: read `debian/changelog`, `debian/control`, `debian/rules`, source format, patches, CI files, and gbp configuration. Avoid upstream source except for narrowly scoped packaging needs.
2. Check external state when network is available: package tracker, BTS, Salsa merge requests/issues, upstream releases, build logs, debusine work requests. When the user mentions a Debian bug number, open the BTS report before trusting the summary or adding `Closes:`.
3. Make the smallest packaging change that satisfies the task. Preserve existing style and helper stack.
4. Update Debian metadata only when needed: `debian/changelog`, dependencies, symbols, install files, patches, tests, copyright, watch, or maintscript snippets.
5. Validate locally with the lightest useful checks first, then full build/tests when the change warrants it.
6. Summarize changed packaging intent, commands run, remaining risks, any external blockers, and links to the external services that informed the work.

## Reference Map

Read only the reference needed for the current task:

- `LOCAL.md` (top level, optional): the local maintainer's environment — builder command, default architecture/suite, result and log locations, commit timing, upload targets. Present only when someone has created it from `LOCAL.md.example`. Load it whenever the task needs a concrete build, test, or upload command.
- `references/policy.md`: normative Debian policy, archive rules, changelog/control/copyright basics.
- `references/tools.md`: gbp, pbuilder, sbuild, devscripts, uscan, quilt, pristine-tar, release helpers.
- `references/qa.md`: lintian, autopkgtest, piuparts, reproducibility, build logs, dependency/installability checks.
- `references/services.md`: tracker.debian.org, bugs.debian.org, salsa.debian.org, debusine, buildd, mentors.
- `references/workflows.md`: day-to-day packaging workflows — bug fixes, new upstream releases, packaging revisions, NMUs, backports, transitions, FTBFS, autopkgtest regressions, and the pre-upload checklist.
- `references/released-suite-updates.md`: landing fixes in a released suite — choosing between a security upload and a stable update, and the full procedure for each.
- `references/extensions.md`: additional Debian resources to consult or add to this skill over time.

## Local Validation Ladder

Start with the cheapest checks before reaching for full builds or chroot-backed tests:

```bash
dpkg-parsechangelog
dpkg-checkbuilddeps
gbp config dump
debian/rules clean
```

For the full ladder — source/binary build commands, lintian, autopkgtest, piuparts, and reproducibility checks — see `references/qa.md`. For the concrete builder invocation this environment uses, see `LOCAL.md` when it exists.

Adapt commands to the repository's existing configuration. If a command would require network, privileged chroots, or writes outside the workspace, use the host's normal approval flow or report the blocker; never bypass the environment.

## Updating This Skill

When a Debian maintenance task reveals reusable knowledge, add it to the narrowest reference file rather than expanding `SKILL.md`. Prefer concise rules, command patterns, gotchas, and links to primary documentation. Environment-specific defaults belong in `LOCAL.md`; `SKILL.md` and the `references/` stay neutral so they can be shared between maintainers.
