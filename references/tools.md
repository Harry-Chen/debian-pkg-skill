# Debian Packaging Tools

Primary sources:

- git-buildpackage overview/manpages: https://manpages.debian.org/unstable/git-buildpackage/git-buildpackage.1.en.html
- `gbp buildpackage`: https://manpages.debian.org/unstable/git-buildpackage/gbp-buildpackage.1.en.html
- `gbp.conf`: https://manpages.debian.org/unstable/git-buildpackage/gbp.conf.5.en.html
- `git-pbuilder`: https://manpages.debian.org/unstable/git-buildpackage/git-pbuilder.1.en.html
- `gbp dch`: https://manpages.debian.org/unstable/git-buildpackage/gbp-dch.1.en.html
- pristine-tar: https://manpages.debian.org/unstable/pristine-tar/pristine-tar.1.en.html
- pbuilder: https://manpages.debian.org/unstable/pbuilder/pbuilder.8.en.html
- sbuild: https://manpages.debian.org/unstable/sbuild/sbuild.1.en.html
- uscan/devscripts: https://manpages.debian.org/unstable/devscripts/uscan.1.en.html
- dch/devscripts: https://manpages.debian.org/unstable/devscripts/dch.1.en.html
- dput: https://manpages.debian.org/unstable/dput/dput.1.en.html
- quilt: https://manpages.debian.org/unstable/quilt/quilt.1.en.html

## gbp

Use `gbp` as the default repository-aware entrypoint. Always inspect existing config before assuming branch names, tags, builder, pristine-tar, or tarball location:

```bash
gbp config dump
gbp buildpackage --help
```

Common build commands:

```bash
gbp buildpackage
gbp buildpackage --git-pbuilder
gbp buildpackage --git-builder=sbuild
gbp buildpackage --git-tag-only
gbp buildpackage --git-ignore-new
gbp buildpackage --git-ignore-new --git-branch=<branch>
```

Use `--git-ignore-new` deliberately for pre-upload test builds when `debian/changelog` has been finalized but the release commit is intentionally delayed until after upload. Do not use it to hide uncommitted source, patch, or packaging logic changes.

## gbp With pbuilder

`gbp` can drive `pbuilder` so a single `gbp buildpackage` invocation produces a clean-chroot build. Inspect existing config (`~/.gbp.conf`, project `gbp.conf`, `DIST`/`ARCH` in the environment) before assuming chroot names, builder backend, or default suite.

Typical commands:

```bash
gbp buildpackage --git-pbuilder
gbp buildpackage --git-pbuilder --git-arch=<arch> --git-dist=<suite>
DIST=<suite> ARCH=<arch> git-pbuilder create
DIST=<suite> ARCH=<arch> git-pbuilder update
```

Treat `sid` and `unstable` as equivalent in Debian suite naming, but preserve the spelling actually used in a command when documenting or rerunning it.

Builder choice is a per-maintainer setting: some drive pbuilder through `gbp`, others use sbuild, plain `pdebuild`, or a personal wrapper script with its own result and log directories. Do not assume `gbp buildpackage --git-pbuilder` is the local build path — check `LOCAL.md` and the repository's gbp config, and ask if neither answers it.

If a pbuilder chroot is missing or stale, ask before creating/updating it because that may require root, network, and writes outside the workspace.

## sbuild

Use sbuild when the repository or maintainer workflow expects it, when matching Debian buildd behavior matters, or when autopkgtest/sbuild chroots are already configured.

Typical commands:

```bash
sbuild -A -s -v
sbuild --no-clean-source
gbp buildpackage --git-builder=sbuild
```

Do not assume schroot/unshare backend names. Inspect local config (`~/.sbuildrc`, available chroots) when needed.

## pbuilder

pbuilder builds packages in a clean chroot. It is useful for local reproducibility and dependency sanity. Prefer gbp's pbuilder integration (see above) over direct `pdebuild` when the repository's gbp config already drives pbuilder.

Common direct commands:

```bash
pbuilder create --distribution unstable
pbuilder update --distribution unstable
pbuilder build ../package_version.dsc
pdebuild
```

Use direct pbuilder only if gbp integration is not available or the user asks for it.

## uscan And Upstream Imports

Use `uscan` to check upstream releases from `debian/watch`:

```bash
uscan --report
uscan --download-current-version
gbp import-orig --uscan
gbp import-orig ../upstream.tar.*
```

For new upstream releases, preserve the repository's branch/tag model and pristine-tar setting. Check `debian/watch`, upstream signing, repacks, Files-Excluded, and copyright changes before declaring the import ready.

## pristine-tar

When `gbp.conf` has `pristine-tar = True`, treat pristine-tar as part of the source package state. The `pristine-tar` branch stores the data needed to reconstruct the exact upstream tarball from the upstream branch plus delta metadata.

Common checks:

```bash
git branch --list '*pristine-tar*'
git config --get gbp.pristine-tar
gbp config dump
```

Common import/build patterns:

```bash
gbp import-orig --uscan --pristine-tar
gbp import-orig ../package_version.orig.tar.* --pristine-tar
gbp buildpackage --git-pristine-tar
```

Do not delete or regenerate pristine-tar history casually. If an upstream tarball was repacked, check `debian/copyright` `Files-Excluded`, `debian/watch`, and the orig tarball name before importing. If the tarball cannot be reproduced, report that as a release-blocking packaging issue rather than forcing an unrelated tarball.

## gbp dch

Use `gbp dch` to generate a changelog draft from git history, then edit it as maintainer-facing release notes.

Useful patterns:

```bash
gbp dch
gbp dch --since=<commit-or-tag>
gbp dch --debian-branch=<branch>
gbp dch --release
```

After running `gbp dch`, always review and edit `debian/changelog`; do not treat generated commit subjects as final changelog text. Add `Closes: #NNNNNN` manually where the release actually closes Debian bugs.

## quilt And Patch Queues

For `3.0 (quilt)` packages, maintain patches cleanly:

```bash
quilt push -a
quilt pop -a
quilt new fix-name.patch
quilt add path/to/file
editor path/to/file
quilt diff
quilt refresh
quilt header -e
quilt pop -a
```

With gbp patch queues:

```bash
gbp pq import
gbp pq export
gbp pq drop
```

Use whichever model the repository already uses. When adding patches, include DEP-3 metadata unless the repository's practice is clearly different.

Quilt rules that avoid corrupt patch stacks:

- Push all patches before editing if the change depends on current patched source.
- Use `quilt add` before modifying each file so the patch records the correct delta.
- Use `quilt refresh` only for the intended top patch; check `quilt top` first.
- Use `quilt diff` before refresh and `git diff debian/patches` after refresh.
- Keep `.pc/` untracked and do not manually edit it.
- Keep `debian/patches/series` ordered and deterministic.
- Prefer `quilt header -e` for DEP-3 metadata so refreshes preserve the header.
- After patch work, run `quilt pop -a` and confirm `dpkg-source --before-build .` can reapply patches.

Patch filename convention: use short lowercase names with hyphens and `.patch`, for example `fix-ftbfs-gcc-16.patch`. Avoid names that encode temporary bug theories.

When editing an existing patch:

```bash
quilt push -a
quilt top
quilt files
quilt add <additional-file-if-needed>
editor <file>
quilt diff
quilt refresh
quilt header -e
```

## devscripts Helpers

Useful commands:

```bash
dch
debchange
debuild
debsign
debrelease
debdiff
debc
debi
wrap-and-sort
mk-build-deps
```

Use helper tools to reduce formatting mistakes, but review the diff because they can reorder or rewrite packaging files extensively.

## Upload Helpers

Treat upload helpers as potentially dangerous. They may infer a profile and a recent `.changes` file.

Rules:

- Never run bare `dput` or bare `dupload`.
- Never run `dput --version` or similar probes if the local implementation might treat missing changes files as an upload prompt.
- Always specify the target profile and the exact `.changes` file, for example `dput debusine.debian.net zfs-linux_2.4.2-2_source.changes`.
- Use debusine for intermediate or unsigned QA uploads when configured; do not equate it with Debian archive upload.
