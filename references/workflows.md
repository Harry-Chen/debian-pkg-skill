# Debian Packaging Workflows

The upstream-source boundary — when to read upstream code rather than just `debian/`, and which upstream agent-instruction files to skip — is part of the operating model in `SKILL.md`. Re-read it before any workflow that might pull you into the upstream tree.

## Bug Fix In Debian Packaging

1. Read the BTS report, package tracker, and Salsa MRs/issues.
2. Reproduce locally if possible with the package's normal build/test path.
3. Decide whether the fix belongs in Debian packaging, upstream source, dependencies, or tests.
4. For upstream source in `3.0 (quilt)`, add or update a quilt patch. If the fix already exists upstream, import that commit and keep its `git format-patch` header; otherwise write a DEP-3 header (see `policy.md`).
5. Update `debian/changelog` with a concise entry and `Closes: #NNNNNN` if appropriate.
6. Run focused validation, then full build/autopkgtest if the fix affects runtime behavior.

For quilt patches, use this default sequence:

```bash
quilt push -a
quilt new fix-specific-issue.patch
quilt add path/to/file
editor path/to/file
quilt diff
quilt refresh
quilt header -e
quilt pop -a
dpkg-source --before-build .
```

That sequence is for a patch written in Debian. Its header must explain provenance and forwarding state: use `Forwarded: not-needed` only for genuinely Debian-specific changes, otherwise `Forwarded: no` until the patch is sent upstream. When the fix instead exists as an upstream commit, `quilt import` it and leave the upstream header alone.

## New Upstream Release

1. Check tracker, watch file, upstream changelog, current Debian delta, and open bugs.
2. Run `uscan --report`; use `gbp import-orig --uscan` if the repository supports it.
3. If `pristine-tar = True`, import with pristine-tar enabled and verify the orig tarball is reproducible.
4. Refresh/drop Debian patches; forward remaining patches upstream where possible.
5. Review copyright and license changes.
6. Review build system changes and dependencies.
7. Generate a changelog draft with `gbp dch`, then edit it into release notes and add `Closes:` entries.
8. Run clean build, lintian, and autopkgtest.
9. Update changelog as `<upstream>-1` unless Debian versioning requires otherwise.

Common command sequence:

```bash
uscan --report
gbp import-orig --uscan --pristine-tar
quilt push -a
quilt refresh
quilt pop -a
gbp dch
editor debian/changelog
dpkg-buildpackage -S -us -uc
# then a clean-chroot build of the resulting .dsc with the local builder
# (gbp --git-pbuilder, sbuild, pdebuild, or a wrapper — see LOCAL.md)
```

Drop `--pristine-tar` only when the repository does not use pristine-tar.

## Packaging-Only Revision

Use for metadata, dependency, test, or packaging fixes with no upstream version change.

1. Increment Debian revision with `dch`.
2. Keep changes under `debian/` unless a quilt patch is required.
3. Commit each logical change with a changelog-ready subject, for example `d/control: add missing build dependency on foo`.
4. Use `gbp dch` when the packaging branch has meaningful commits to summarize.
5. Edit `debian/changelog` to remove commit noise and add accurate `Closes:` entries.
6. Validate with source build and relevant QA.
7. Avoid unrelated standards/version churn.

## Commit Before Changelog Generation

Before running `gbp dch`, review recent commit subjects:

```bash
git log --oneline <last-upload-tag>..HEAD
```

The subjects should be good raw material for changelog entries. If they are vague, mixed-purpose, or too implementation-heavy, prefer fixing/squashing local commits before generating the changelog when history is still private. Do not rewrite shared history without explicit maintainer approval.

Suggested commit granularity:

- one commit for a dependency/control change
- one commit for a quilt patch add/remove/refresh set
- one commit for autopkgtest changes
- one commit for symbols/install/rules metadata updates
- one release commit for final `gbp dch`/upload changelog state when repository practice uses it

Commit timing for the release changelog entry varies by maintainer: some commit it before the upload, others delay it until the upload succeeds so the version is never claimed twice. When it is delayed, a pre-upload build legitimately has exactly one uncommitted file, `debian/changelog`, with the distribution already set — build with `--git-ignore-new` (or the builder's equivalent) rather than committing the release entry prematurely. `LOCAL.md` records the local rule.

## Workspace Hygiene

Debian builds legitimately write outside the source tree: `dpkg-buildpackage` leaves `.dsc`, `.changes`, `.buildinfo`, and binaries in the parent directory, and chroot builders use their own result and log directories. That is expected output — do not clean the parent directory, and do not redirect build output elsewhere unless asked.

Everything you generate yourself is different. Scratch notes, upstream patches downloaded before they become `debian/patches/*`, diff dumps, and log excerpts belong in a git-ignored scratch directory inside the workspace, not in the parent directory next to the build artifacts, not under `debian/`, and not in `/tmp` where the user cannot see them. Delete them when the task ends, and never let one reach a commit.

Before committing, check `git status --short` for scratch files, editor backups, `.pc/`, and build leftovers. If the repository lacks a `.gitignore` entry that would have caught a recurring artifact, propose adding one rather than deleting the file every time.

Rewriting history is limited to what is still private: local commits not yet pushed and not yet part of a release tag. Squashing or reordering those before `gbp dch` is fine; touching shared history is not, and neither is amending a commit that already exists — add a new one instead.

## NMU

Non-maintainer uploads require extra care.

1. Confirm the bug severity and NMU appropriateness in Developer's Reference.
2. Keep the diff minimal.
3. Use NMU versioning, usually `+nmu1` for non-native packages.
4. Include changelog entries that clearly identify the fixed bug.
5. Respect DELAYED queue conventions unless an immediate upload is justified.

## FTBFS

1. Collect failing build log, architecture, suite, build profile, and dependency versions.
2. Try reproducing in clean pbuilder/sbuild matching the failure.
3. Distinguish missing Build-Depends from upstream/compiler/toolchain breakage.
4. Patch with the smallest change and include upstream forwarding status.
5. Validate at least a source build and clean binary build.

Compiler/toolchain fixes often belong upstream. If the patch is backported from upstream, set `Origin: upstream` or `Origin: backport`; if created in Debian, include `Bug-Debian` and forward it upstream when practical.

## Autopkgtest Regression

1. Read `debian/tests/control`, failing logs, triggered packages, and testbed architecture.
2. Reproduce with the same backend if possible; use `null` only for quick local checks.
3. Determine whether the regression is in the package, a dependency, test assumptions, or infrastructure.
4. Prefer fixing the package/test over adding broad restrictions.
5. If adding `Restrictions`, document why the test genuinely requires them.

## Library Transition

1. Identify ABI/API change, reverse dependencies, and release team coordination needs.
2. Update symbols/shlibs carefully; compare build logs and generated files.
3. Avoid breaking coinstallability or multiarch metadata.
4. Consider binNMUs and transition tracker state before upload.

## Pre-Upload Checklist

Run or explicitly defer:

```bash
git status --short
dpkg-parsechangelog
gbp config dump
dpkg-buildpackage -S -us -uc
<clean-chroot build of ../<source>_<version>.dsc>    # local builder; see LOCAL.md
lintian <result-dir>/<source>_<version>_<arch>.changes
autopkgtest ../<source>_<version>.dsc -- <backend>
```

Builders differ in where they leave `.changes`, `.deb`, and `.buildinfo` files: `dpkg-buildpackage` and gbp use `..`, while most chroot wrappers use their own result directory. Resolve that path before running `lintian`, `debdiff`, `debc`, or `dput`.

Check whether the builder already writes a timestamped log of its own before piping through `tee`. When you do add a `tee`, use `set -o pipefail` so build failures are not hidden by the pipeline.

Check:

- changelog distribution is correct, not `UNRELEASED`
- version is correct for target suite
- bug closures are intended
- source format and patch stack are clean
- generated files are cleaned
- Vcs fields point to the active repository
- test failures and lintian findings are understood
