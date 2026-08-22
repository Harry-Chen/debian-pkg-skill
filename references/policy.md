# Debian Policy And Core Packaging Rules

Primary sources:

- Debian Policy Manual: https://www.debian.org/doc/debian-policy/
- Developer's Reference: https://www.debian.org/doc/manuals/developers-reference/
- Guide for Debian Maintainers: https://www.debian.org/doc/manuals/debmake-doc/
- Debian New Maintainers' Guide, when relevant for introductory context: https://www.debian.org/doc/manuals/maint-guide/

## Priority

Use Debian Policy as the normative source. Developer's Reference describes maintainer practice and project procedures. Guides explain workflows, but do not override Policy.

When packaging behavior is uncertain:

1. Check the exact Policy section.
2. Check existing package practice in the archive for the same language/ecosystem.
3. Check Developer's Reference for maintainer process.
4. If still ambiguous, surface the ambiguity and propose the least surprising Debian-compatible change.

## Files To Inspect

- `debian/changelog`: source name, version, target distribution, urgency, maintainer trailer, bug closures.
- `debian/control`: source/binary packages, maintainer/uploaders, build deps, standards version, rules-requires-root, Vcs fields, homepage, testsuite hints.
- `debian/rules`: debhelper compat style, build system override hooks, hardening, generated artifacts.
- `debian/source/format`: usually `3.0 (quilt)` for non-native packages.
- `debian/copyright`: copyright format, Files stanzas, license text, upstream metadata.
- `debian/patches/`: quilt patch stack, header style (preserved upstream commits vs. DEP-3), forwarding state.
- `debian/tests/`: autopkgtest control and test scripts.
- `debian/watch`: upstream release monitoring through uscan.

## Changelog Rules

Use `dch`, `debchange`, or `gbp dch` when possible so formatting, urgency, suite, and maintainer identity remain conventional.

Common patterns:

```bash
dch --local '~test' 'Local test build.'
dch --distribution unstable 'Release to unstable.'
dch --newversion <version>-1 'New upstream release.'
dch --nmu 'Non-maintainer upload.'
gbp dch
gbp dch --release
```

Bug closure syntax in changelog entries should use `Closes: #NNNNNN`. Keep changelog entries user-visible and Debian-facing; do not describe internal implementation details unless they matter to maintainers.

Recommended maintainer flow:

1. Generate a draft with `gbp dch` when git history is meaningful.
2. Rewrite commit-subject noise into release notes grouped by packaging area.
3. Add Debian bug closures manually with `Closes: #NNNNNN`.
4. Check target distribution, urgency, version, and trailer identity.
5. Avoid declaring `UNRELEASED` entries as ready for upload.

Style guidance:

- Use concise bullets that describe user-visible or maintainer-relevant changes.
- Use `d/<file-or-dir>` shorthand only when the repository already uses it.
- Group related patch, control, symbols, install, and test changes rather than listing every commit.
- For team-maintained packages, preserve contributor blocks like `[ Name ]` when multiple people contributed.
- Prefer `New upstream version X.` for upstream releases, then list Debian-specific patch/metadata follow-up.
- Use `Closes:` capitalization consistently within the package; Debian accepts case-insensitive variants, but consistency improves review.
- Do not add bug closure lines unless the change actually fixes the BTS issue in the upload target.

## Git Commit Messages For gbp dch

Write packaging commit subjects so `gbp dch` can turn them into usable changelog items with minimal editing. The subject should already read like an imperative-free Debian changelog bullet after `gbp dch` strips metadata.

Good subject patterns:

```text
d/control: add missing build dependency on foo
d/patches: refresh existing patches
d/patches: backport upstream fix for bar
d/tests: add smoke test for foo-helper
d/rules: stop installing generated cache files
binary-package-name: install new systemd service
New upstream version 1.2.3
Update upstream source from tag 'upstream/1.2.3'
Gbp-Dch: update and upload 1.2.3-1 to unstable
```

Rules:

- Keep the first line changelog-ready: concise, specific, and useful without reading the diff.
- Prefix Debian packaging file changes with `d/<path>:` when the package already uses that style.
- Prefix binary-package-specific changes with the binary package name when that is clearer than a file path.
- Use lowercase action verbs after the prefix unless repository style differs: `add`, `remove`, `refresh`, `update`, `fix`, `bump`, `switch`.
- Include `Closes: #NNNNNN` in the subject only when the commit directly corresponds to a BTS closure and the package's style accepts it; otherwise add closures while editing `debian/changelog`.
- Keep generated upstream-import commits as gbp creates them; do not rewrite them into hand-written packaging bullets.
- Use release/update commits such as `Gbp-Dch: update and upload <version> to <suite>` for finalized changelog updates when that matches repository practice.
- Avoid vague subjects like `fix build`, `update packaging`, `misc changes`, or `cleanup`; they produce poor changelog entries.
- Avoid packing unrelated changes into one commit because `gbp dch` cannot split them into meaningful bullets.

## Control File Guidance

Keep dependency changes minimal and justified. Prefer substvars where the helper stack expects them, for example `${misc:Depends}`, `${shlibs:Depends}`, `${python3:Depends}`, or `${perl:Depends}`.

Check these fields before upload-prep:

- `Standards-Version`: update only after reviewing relevant Policy changes; do not bump mechanically.
- `Rules-Requires-Root`: use `no` when the package builds rootlessly.
- `Vcs-Git` and `Vcs-Browser`: should point to Salsa or the active packaging repository.
- `Testsuite`: may be generated from `debian/tests/control`; do not hand-edit without need.
- `Build-Depends`: include test/build helper dependencies needed by `debian/rules` and autopkgtest when appropriate.

## Source Format And Patches

For `3.0 (quilt)`, keep upstream source changes as quilt patches under `debian/patches/`, not as direct edits to upstream files.

Every patch needs a header that explains what it does and where it came from. Which *kind* of header depends on whether the change exists upstream as a commit.

### Prefer a preserved upstream cherry-pick

When the change already exists as an upstream commit, ship that commit as `git format-patch` produced it. Do not reformat it into a hand-written DEP-3 header. Keep the `From <hash> Mon Sep 17 00:00:00 2001` line, `From:`, `Date:`, `Subject:`, the commit message body, and the `---` separator before the diff.

This is not a deviation from DEP-3. The spec defines `Subject` as an alias for `Description` and `From` as an alias for `Author` precisely so that `git format-patch` output is already a valid patch header.

Why the upstream form is worth preserving:

- The file stays comparable with `git show <hash>` in the upstream repository, so provenance is verifiable in one command instead of trusted from a hand-copied `Origin:` URL.
- Re-importing the same commit later — new upstream release, a second suite, a different maintainer — reproduces the same file, so the diff is empty rather than noisy.
- Nothing has to be maintained by hand. A hand-written `Description:` drifts from the commit message it paraphrases, and `Last-Update:` goes stale on every refresh.

Two edits to the upstream output are expected and fine:

- Correcting `--- a/…` / `+++ b/…` paths when upstream moved the file after the release being patched.
- Tightening `@@` hunk line numbers so the patch applies without offsets.

`gbp pq export` already emits this shape from the patch-queue branch; prefer it over hand-assembling patch files when the repository uses `gbp pq`.

If the package's practice is to record Debian-side context on every patch, append the extra fields to the header block instead of rewriting it, and only for facts the commit does not already carry — `Bug-Debian:` is worth adding, `Origin:`/`Forwarded:` are redundant for something demonstrably already upstream.

### Write a DEP-3 header when there is no upstream commit

Use a full DEP-3 header for patches that do not correspond to an upstream commit:

- Debian-specific changes with no upstream counterpart (`Forwarded: not-needed`).
- Patches created in Debian and not yet sent upstream (`Forwarded: no`).
- Hand-recreated backports, where the upstream diff does not apply because the surrounding code was refactored. Describe the divergence in a `Note:` paragraph — that is information the upstream commit cannot carry.

Fields:

- `Description` or `Subject`: required short purpose; add a longer rationale when needed.
- `Origin`: required unless `Author` is present; use `upstream`, `backport`, `vendor`, or `other` prefixes when useful.
- `Bug` and `Bug-Debian`: upstream and Debian bug URLs.
- `Forwarded`: URL when forwarded, `no` when not yet forwarded, `not-needed` for Debian-specific patches.
- `Author` or `From`: patch author, especially for long-lived downstream patches.
- `Reviewed-by` or `Acked-by`: reviewer identity when available.
- `Last-Update`: ISO date, `YYYY-MM-DD`.
- `Applied-Upstream`: version, URL, or commit where upstream accepted it.

Minimal DEP-3 header for a Debian-created patch:

```text
Description: Fix <specific behavior>
 Explain why this patch is needed in Debian and what user-visible
 behavior it changes.
Author: Name <email@example.org>
Bug-Debian: https://bugs.debian.org/NNNNNN
Forwarded: no
Last-Update: YYYY-MM-DD
```

For a backport that had to be rewritten rather than cherry-picked, keep the upstream attribution and state what differs:

```text
From: <upstream author of the equivalent commit>
Subject: <descriptive backport title>

<what the fix does>

Origin: backport, <upstream commit URL>
Bug: <upstream tracker URL>
Bug-Debian: https://bugs.debian.org/NNNNNN
Forwarded: not-needed
Note: <how this differs from upstream — file renamed, helper inlined, …>
```

Use `quilt`, `gbp pq`, or the repository's existing patch workflow. Do not mix workflows casually.

## Copyright

Prefer machine-readable `debian/copyright` format. Verify new upstream files, vendored code, generated files, and test fixtures when updating to a new upstream release.

Useful commands:

```bash
licensecheck --recursive .
grep -R "Copyright" -n .
```

## Versioning

Common Debian version patterns:

- New Debian revision of existing upstream: `<upstream>-<debian_revision>`.
- New upstream release: `<new_upstream>-1`.
- Native package: `<version>` with no Debian revision.
- NMU: usually `<upstream>-<debian_revision>+nmu1`.
- Security/stable updates: suite-specific suffixes, coordinate with the relevant team.
- Backports: usually `~bpo<debian_release>+1`.

Do not invent suffixes for archive uploads without checking the target suite's practice.
