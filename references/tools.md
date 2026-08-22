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

## Tool Environment

Several packaging tools take defaults from the environment, and maintainers commonly export them from a shell profile or `~/.quiltrc`. Check what is already set before prefixing a command with your own values:

```bash
printenv QUILT_PATCHES QUILT_REFRESH_ARGS DEBEMAIL DEBFULLNAME DEB_BUILD_OPTIONS
```

- `QUILT_PATCHES` — must be `debian/patches` for Debian source packages. It can also come from `~/.quiltrc` or, once a series has been pushed, `.pc/.quilt_patches`. If any of those is already correct, run `quilt` bare instead of prefixing every invocation; setting it wrongly creates a second patch stack in the wrong directory.
- `QUILT_REFRESH_ARGS` — commonly `-p ab --no-timestamps --no-index` so refreshes do not add timestamp and index noise to every patch.
- `DEBEMAIL` / `DEBFULLNAME` — identity used by `dch`, `gbp dch`, and the changelog trailer. Do not override them unless the task explicitly calls for a different identity.
- `DEB_BUILD_OPTIONS` — e.g. `parallel=N`, `nocheck`, `noopt`. Builder wrappers often set this already; adding your own value can silently disable the build-time test suite.

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

## Chroot Mirrors

Chroot creation, chroot updates, and test builds should use the mirror the system is already configured with, not a hardcoded URL. Read it from the existing configuration first:

```bash
grep -rhE '^(deb |URIs:)' /etc/apt/sources.list /etc/apt/sources.list.d/
grep -hE 'MIRRORSITE|OTHERMIRROR|APTCACHE' ~/.pbuilderrc /etc/pbuilderrc
grep -hiE 'mirror|distribution' ~/.sbuildrc
```

A non-default mirror is usually configured for a reason: bandwidth, a caching proxy such as apt-cacher-ng, or an internal archive carrying packages the public mirrors do not have. Overriding it silently wastes the cache at best and produces an unreproducible chroot at worst.

Fall back to `https://deb.debian.org/debian` only when the configured mirror is unreachable, stale, or missing the suite you need — and say which mirror was used and why.

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

Use whichever model the repository already uses.

For a fix that exists upstream, import the commit instead of retyping it as a new patch — `quilt import` copies the file into `debian/patches/` and appends it to `series` without touching its contents, which is what keeps a cherry-pick comparable to upstream:

```bash
curl -sL <commit-url>.patch -o .tmp/<short-hash>.patch
quilt import -P CVE-YYYY-NNNNN.patch .tmp/<short-hash>.patch
quilt push
```

`gbp pq export` produces the same shape from the patch-queue branch. See `policy.md` (*Source Format And Patches*) for when to preserve the upstream header and when to write a DEP-3 one instead.

Quilt rules that avoid corrupt patch stacks:

- Confirm `QUILT_PATCHES` resolves to `debian/patches` (see *Tool Environment*) before creating a patch; do not reflexively prefix every command with it.
- Push all patches before editing if the change depends on current patched source.
- Use `quilt add` before modifying each file so the patch records the correct delta.
- Use `quilt refresh` only for the intended top patch; check `quilt top` first.
- Use `quilt diff` before refresh and `git diff debian/patches` after refresh.
- Keep `.pc/` untracked and do not manually edit it.
- Keep `debian/patches/series` ordered and deterministic.
- Prefer `quilt header -e` for DEP-3 metadata so refreshes preserve the header.
- Avoid `quilt refresh` on a preserved upstream cherry-pick. Refresh rewrites the diff body while keeping everything above it, so the diffstat `git format-patch` emits between `---` and the diff is left describing the old diff. Adjust paths and hunk offsets by editing the file, or drop the diffstat when importing.
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

Each entry below is a one-line summary plus a single representative command. Consult the manpage before relying on flags not shown.

- `dch` / `debchange` — edit `debian/changelog` with correct formatting, urgency, and trailer identity. `debchange` is the long-name alias of `dch`.

  ```bash
  dch --newversion 1.2.3-1 'New upstream release.'
  ```

- `debuild` — wrapper around `dpkg-buildpackage` that also runs `lintian` and signs the result. Useful for quick local builds outside pbuilder/sbuild.

  ```bash
  debuild -us -uc
  ```

- `debsign` — sign an already-built `.changes` (and the referenced `.dsc`) with your GPG key. Use after an unsigned build when ready to upload.

  ```bash
  debsign ../package_1.2.3-1_amd64.changes
  ```

- `debrelease` — wrapper that uploads the latest `.changes` via `dput`/`dupload`. Same upload-safety rules as bare `dput` apply — prefer explicit `dput <profile> <changes>`.

- `debdiff` — compare two source or binary packages and surface added/removed files, control-field changes, and patch differences. Standard pre-upload sanity check, and the engine behind `nmudiff`.

  ```bash
  debdiff ../package_1.2.3-1.dsc ../package_1.2.3-2.dsc
  debdiff ../package_1.2.3-1_amd64.changes ../package_1.2.3-2_amd64.changes
  ```

- `debc` — list the contents of the binary packages produced by the latest build (reads the `.changes` in `..`). Quick check that the right files landed in the right binary package.

  ```bash
  debc
  ```

- `debi` — install the binary packages produced by the latest build into the local system via `sudo dpkg -i`. Useful for ad-hoc local smoke tests; not a substitute for piuparts.

  ```bash
  debi
  ```

- `wrap-and-sort` — reformat `debian/control`, `debian/copyright`, and similar files into a canonical wrapped+sorted layout. Run only if the package already uses that style; otherwise it produces large unwanted diffs.

  ```bash
  wrap-and-sort -ast
  ```

- `mk-build-deps` — generate and optionally install a `<source>-build-deps` meta-package satisfying the source's `Build-Depends`. Handy when reproducing a build outside a clean chroot.

  ```bash
  mk-build-deps --install --remove --tool='apt-get -y --no-install-recommends'
  ```

- `build-rdeps` — list source packages that build-depend on a given binary package. Install `dose-extra` for accurate transitive, architecture-, and profile-aware results; the default falls back to a naive `grep` over `Sources` files.

  ```bash
  build-rdeps --distribution unstable libfoo-dev
  ```

  Use before transitions, SONAME bumps, or removals to estimate breakage scope.

- `dget` — fetch a `.dsc` or `.changes` URL and all referenced components, verify checksums and signatures via `dscverify`, and unpack with `dpkg-source`. Works for both archive URLs and mentors/salsa links.

  ```bash
  dget https://deb.debian.org/debian/pool/main/h/hello/hello_2.10-3.dsc
  ```

  For plain archive lookups, `apt-get source <pkg>` is usually shorter; reach for `dget` when the source lives outside your apt config or you need signature verification.

- `nmudiff` — generate a `debdiff` between the previous archive version and the freshly built NMU in `..`, open it in `$EDITOR`/`mutt`, and mail the result to the BTS bug(s) the NMU closes. Run from the NMU source tree after the build.

  ```bash
  nmudiff --delay 5
  ```

  Use `--new` to file a fresh bug instead of mailing the closed ones; pair with `--delay` matching the DELAYED queue you uploaded to. Required step for any non-trivial NMU.

- `getbuildlog` — download buildd logs for a package across versions and architectures. Patterns are extended regexes; the literal token `last` fetches only the most recent version.

  ```bash
  getbuildlog zfs-linux last amd64
  getbuildlog hello '2\.10-.*' '.*'
  ```

  Use when a binNMU or porter build fails and you need the exact log without clicking through `buildd.debian.org`.

Use these helpers to reduce formatting mistakes and shortcut common lookups, but review their diffs and output — several (notably `wrap-and-sort`, `mk-build-deps`, `debrelease`) can rewrite files or take privileged actions.

## Upload Helpers

Treat upload helpers as potentially dangerous. They may infer a profile and a recent `.changes` file.

Rules:

- Never run bare `dput` or bare `dupload`.
- Never run `dput --version` or similar probes if the local implementation might treat missing changes files as an upload prompt.
- Always specify the target profile and the exact `.changes` file, for example `dput debusine.debian.net zfs-linux_2.4.2-2_source.changes`.
- Use debusine for intermediate or unsigned QA uploads when configured; do not equate it with Debian archive upload.
