# Debian QA, Tests, And Build Validation

Primary sources:

- autopkgtest: https://manpages.debian.org/unstable/autopkgtest/autopkgtest.1.en.html
- autopkgtest package tests spec: https://salsa.debian.org/ci-team/autopkgtest/raw/master/doc/README.package-tests.rst
- lintian: https://lintian.debian.org/
- lintian manpage: https://manpages.debian.org/unstable/lintian/lintian.1.en.html
- piuparts: https://piuparts.debian.org/
- reproducible builds: https://reproducible-builds.org/docs/
- Debian CI: https://ci.debian.net/
- buildd: https://buildd.debian.org/

## Validation Strategy

Use a ladder: static metadata checks, clean source/build checks, lintian, autopkgtest, install/upgrade checks, then archive/service results. Do not run the heaviest tool first unless the task specifically requires it.

## Static And Source Checks

Useful commands:

```bash
dpkg-parsechangelog
dpkg-checkbuilddeps
dpkg-source --before-build .
dpkg-source --build .
debian/rules clean
```

Look for source diffs after clean/build:

```bash
git status --short
```

Generated files that remain modified after `clean` are a packaging bug unless deliberately tracked.

## Build Validation

Prefer the maintainer's configured clean-build path — `LOCAL.md` records it when the environment has one:

```bash
gbp buildpackage --git-pbuilder
gbp buildpackage --git-builder=sbuild
dpkg-buildpackage -us -uc
```

For source-only upload checks:

```bash
gbp buildpackage -S
dpkg-buildpackage -S -us -uc
```

When full local chroot builds are unavailable, state that explicitly and run cheaper checks.

## lintian

Run lintian against `.changes`, `.dsc`, or built packages:

```bash
lintian ../*.changes
lintian -i -I --pedantic ../*.changes
```

Treat lintian output as signal, not law. Fix policy-relevant issues. For intentional overrides, add narrow `debian/*.lintian-overrides` entries with comments when helpful.

## autopkgtest

Inspect:

- `debian/tests/control`
- `debian/tests/*`
- package runtime dependencies
- test isolation assumptions

Common local commands:

```bash
autopkgtest . -- null
autopkgtest ../*.dsc -- null
autopkgtest ../*.dsc -- schroot <suite>-amd64-sbuild
autopkgtest ../*.dsc -- qemu <image>
```

Use `null` only for quick smoke tests because it is not isolated like schroot/qemu. If tests need network, root, isolation-machine, or installed built binaries, check `Restrictions` and the chosen backend.

Some maintainers drive autopkgtest through their builder wrapper instead of calling `autopkgtest` directly, so the testbed matches the build chroot. When `LOCAL.md` records such a command, prefer it over the raw invocations above.

When adding tests, keep them deterministic and archive-friendly. Avoid external network access unless explicitly allowed by the test restrictions and accepted by Debian CI practice.

## piuparts

Use piuparts for install, upgrade, and purge validation, especially when maintainer scripts, conffiles, diversions, alternatives, triggers, or system users are involved.

Typical command:

```bash
piuparts ../*.changes
```

piuparts often needs root/chroot setup. Ask before creating or updating chroots.

## Reproducibility And Build Logs

Check reproducibility-sensitive areas:

- timestamps and locale-dependent output
- embedded build paths
- generated docs
- file ordering in archives
- network access during build
- CPU/thread nondeterminism

Use build logs from local builds, buildd, debusine, or reproducible-builds infrastructure to identify environment-specific failures.

## Installability And Transitions

For dependency or library changes, consider:

- reverse dependencies
- ABI/API changes
- symbols/shlibs updates
- binNMU compatibility
- transitions coordinated with release team

Useful tools/sites:

- `dose-builddebcheck`
- `ben`
- `dak rm`/tracker information when available
- package tracker and release team transition pages
