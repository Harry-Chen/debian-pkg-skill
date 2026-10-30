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
- Debian control fields, including source/binary version relationships: https://www.debian.org/doc/debian-policy/ch-controlfields.html#source

## Validation Strategy

Select checks relevant to the requested change or assessment, starting with the cheapest useful evidence: static metadata checks, clean source/build checks, lintian, autopkgtest, install/upgrade checks, then archive/service results. This is a menu, not a requirement to complete every rung. Run required checks and those justified by the affected behavior; expand research or validation only when a concrete finding needs it. Report unavailable relevant checks as limits on the assessment, without treating every unrun optional check as a blocker. Keep judgments and their explanations in the conversation, following the output-artifact rule in `SKILL.md`.

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

## Build Evidence

Keep compilation/linkage, build-time tests, installed-package tests, and runtime smoke tests as separate claims. A zero builder exit status does not establish that every kind of test ran, or that the intended dependency versions were selected.

### Artifact Identity And Integrity

Preserve the exact source input, command, suite, build/host architecture, profiles, builder configuration, dependency manifest, complete log, `.changes`, and `.buildinfo` for each attempt. Attribute outputs through the attempt's `.changes`, not a broad glob over a shared results directory. Copy logs before a later attempt can overwrite them and retain the original archive source version separately from local rebuild suffixes.

For artifact validation:

- Verify every referenced file's checksum and size. `dscverify --nosigcheck` can check unsigned local `.dsc`, `.changes`, and `.buildinfo` files; this checks integrity, not signer authenticity. See the [tool recipes](tools.md#build-artifact-inspection).
- Check source identity/version and each binary's declared architecture against the intended build. An architecture-independent binary may legitimately accompany architecture-dependent outputs.
- Do not require all binary `Version` fields to equal the source version. A binary's `Source: name (version)` explicitly identifies its source version, which need not equal its binary version. Without the parenthesized version, its source version is the binary version; without `Source`, its source name is `Package`. Account for binNMUs and deliberate local version changes when matching the source build.
- Preserve source and binary versions as separate values. A legitimate differently versioned transitional binary is not an unrelated-source artifact, and the original source version is usually the relevant version for an FTBFS report.

### Dependency And ABI Evidence

Repository availability and APT candidate listings are not proof of installation or use. Compare `.buildinfo`'s `Installed-Build-Depends` with an installed-package query inside the build environment. For a particular dependency test, also inspect configuration output and compiler include/library paths: an installed development package can be bypassed by bundled headers.

When the test concerns a library, inspect generated package `Depends`/`Provides`, direct ELF `DT_NEEDED` entries, and applicable symbol versions or namespaces. Use static inspection such as `readelf`, not `ldd` on untrusted artifacts. Check the actual target architecture rather than assuming the host running the inspection is the build architecture.

Do not reject a result solely because an older runtime ABI is co-installed. Build tools or unrelated dependencies may still need it; establish which headers and ABI the rebuilt payload uses. Conversely, direct linkage inspection does not prove an entirely transitioned transitive dependency closure, every inlined/static/header-only dependency, or runtime correctness. Define the expected library packages and ABI in the [transition context](transitions.md), not by banning every older library in the chroot.

### Test Execution

Inspect the log and packaging test targets before saying tests passed:

- Record `DEB_BUILD_OPTIONS=nocheck`, the `nocheck` build profile, selected testbed restrictions, and skipped tests.
- An empty `override_dh_auto_test`, a disabled CMake test target, or a package with no test invocation can suppress execution even when `nocheck` is absent.
- Building test executables is not running them. Report compilation success and tests not run separately; use the configured isolated test path for additional runtime assurance.

### Failure Classification

Use the builder exit status, failure stage, and the first substantive compiler/test/resolver diagnostic together. Base any log-parsing automation on real logs from the builder version in use.

| Outcome | Evidence and handling |
| --- | --- |
| Build/test failure | Compilation or packaging tests actually failed. For example, sbuild can report `Status: attempted`, `Fail-Stage: build`, and a nonzero exit after `dpkg-buildpackage` fails; it need not say `Status: failed`. Inspect the log to distinguish compilation from tests. |
| Dependency/installability failure | A resolver conflict or unsatisfied build dependency prevented compilation. `install-deps` is a typical stage. A later `explain-bd-uninstallable` failure may be a secondary diagnostic-tool error; retain the preceding resolver cause. |
| Infrastructure/setup failure | Chroot creation/access, mirror/network access, missing builder identity, or publication failed. This is not a package compiler failure. |
| Evidence/validator failure | Missing, stale, inconsistent, or wrongly interpreted logs/artifacts prevent a conclusion. Revalidate existing evidence before spending another full rebuild, especially when the builder itself succeeded. |
| Blocked | The source was not attempted because required outputs were unavailable. Do not report it as FTBFS. |

A failed build with a candidate dependency is not automatically a regression caused by that dependency. Where attribution is uncertain, compare the identical source and build context against a baseline, including ABI-compatible companion libraries, and record environmental differences. A diagnostic tied to an independently verified API removal is stronger evidence than unrelated test failures in the same campaign.

Keep internal evidence detailed, but public bug text concise: affected source/version, relevant reproduction conditions, the substantive log excerpt with enough context to identify the failing code/test, and any necessary causal caveat. Check existing bugs before submission and never infer permission to send a report from permission to draft it.

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

For library-specific cohort discovery, candidate dependency stacks, and rebuild ordering, see the [transition playbook](transitions.md). ABI/symbols and binNMU coordination remain part of the [Library Transition workflow](workflows.md#library-transition).
