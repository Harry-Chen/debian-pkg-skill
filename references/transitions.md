# Local Library Transitions

A prepared, unpublished library is enough to plan a local transition: use an
applicable existing tracker, or run Ben against **existing archive metadata**.
Uploading to experimental is not a prerequisite. Archive publication and release
coordination remain separate, explicitly authorized actions; see
[Library Transition](workflows.md#library-transition).

## Inputs And Discovery

Record the candidate `.deb` source identity/version, architecture, expected new
runtime/development binary set and ABI `Provides`; explicitly identify the old
runtime **and development** roots from archive metadata. Do not derive every old
root by decrementing a SONAME. Map candidate and old-root source names explicitly
when renamed or ambiguous. Record suite, components, build/host architecture,
build profiles, mirror and metadata date/digests. Inspect artifacts with
[tool recipes](tools.md#build-artifact-inspection); validate identities with
[QA evidence](qa.md#build-evidence).

Choose one discovery path:

- **Retrieve a tracker.** Find the relevant tracker at
  <https://release.debian.org/transitions/> and its raw `.ben` under
  `https://release.debian.org/transitions/config/{phase}/` (the phase is not
  always `planned`). Save the raw HTML and `.ben` with URLs, retrieval time and
  checksums. As of 2026-10, the HTML encodes adjacency as `td.srcname` cells
  with a source-name `id`, a `title` such as `Dependencies: source1,source2`, and
  `roundN` classes; check the current markup before parsing, since displayed
  names/colors alone are not adjacency. Verify that the predicates, suite and
  architectures match the task. Stale or differently scoped trackers are leads,
  not an authoritative plan; recompute locally when in doubt.
- **Compute locally.** Use the old archive's Sources and Packages with Ben; the
  candidate need not appear in those indexes. Derive predicates from the explicit
  old roots and candidate binary metadata. Include header/static consumers by
  matching development packages in the build-dependency fields. `.build-depends`
  matches that field alone, so query `.build-depends-arch` and (when in scope)
  `.build-depends-indep` explicitly. Broaden roots deliberately where the
  transition has multiple development packages or virtual providers.

For example, replace these fictional names with the actual roots:

```ocaml
title = "Local libfoo transition";
is_affected = .depends ~ /\b(libfoo1|libfoo2)\b/
              | .build-depends ~ /\blibfoo-dev\b/
              | .build-depends-arch ~ /\blibfoo-dev\b/
              | .build-depends-indep ~ /\blibfoo-dev\b/;
is_good = .depends ~ /\blibfoo2\b/;
is_bad = .depends ~ /\blibfoo1\b/;
```

Ben uses regex literals `/.../`, `|` for OR and `&` for AND. Anchor patterns with
`\b` so `libfoo1` does not match `libfoo10`. Runtime predicates do not prove that
header-only consumers used the new headers; keep their build evidence separately.
Tracker good/bad/unknown are dependency-transition statuses, not build outcomes:
old-ABI baseline binaries are expected for an unpublished candidate, not evidence
of FTBFS.

See [Ben commands](tools.md#ben-transition-monitor) for download/monitor recipes.
Snapshot the `.ben`, raw outputs and index inputs. JSON reports statuses, **not
dependency edges**; use `levels` output, where a header announces a round and a
line `source: prerequisite1 prerequisite2` carries adjacency (an empty suffix
means no prerequisites). Verify the format against the installed Ben version.

## Graph Versus Execution Plan

Keep the selected cohort, prerequisite edges, original levels and strongly
connected components (SCCs), external/unresolved prerequisites and provenance as
the immutable discovery result. Ben builds source edges from build-dependency
names and binary/`Provides`-to-source mappings within the cohort; absence of an
edge outside the cohort does not establish installability.

Ben 1.22's graph is an approximation: it takes the first dependency alternative,
ignores version and architecture restrictions, and evaluates profile restrictions
with no active profiles. Levels/SCCs are ordering hints, not a solver; APT in the
actual build environment remains the authority for dependency selection.

A validated prebuilt candidate source is a **seed**. Keep its original edges and
SCC membership, then derive a separate seed-adjusted execution plan: a published
seed satisfies downstream prerequisites without rebuilding itself, which can
break an execution cycle without pretending the graph was acyclic. Other cycles
need a justified bootstrap seed or metadata correction, not a blanket FTBFS
label. Descendants of a cycle are blocked, but not necessarily cyclic.

## Standalone Rebuild Campaigns

Drive the seed-adjusted plan with a **single workspace-local script** around the
configured builder, reusing existing build and QA tools. Smoke-test it on one
prerequisite and one consumer before the full cohort. Do not run one agent per
package to launch builds or wait on them; agents plan the campaign and
investigate failures that need judgment, deterministic code owns execution.

Scheduling:

- Release a source once all its prerequisites are **published**: the complete
  validated runtime/development set, including ABI `Provides`, visible to the
  builder after a package-view refresh. Build success alone does not release
  dependents, and a runtime library alone is not a usable development seed.
- On failure, classify it ([QA failure classification](qa.md#failure-classification)),
  continue independent ready work, and finish with an aggregate report listing
  each unattempted source's blocked edges and why (prerequisite failed,
  unattempted, or missing from the cohort).
- Select/pin the new runtime, development packages and ABI providers; do not pin
  unrelated `A | B` alternatives. Confirm actual APT selection and headers/linkage
  with [dependency and ABI evidence](qa.md#dependency-and-abi-evidence).
- Default to serial source scheduling with the builder's normal parallelism. If
  sources run concurrently, bound aggregate CPU/RAM/disk, isolate workspaces and
  results, and serialize repository publication.

Durable state lives in the workspace, not the conversation:

- Save the discovery inputs, execution plan, and exact build/validation/
  publication commands with their input identities.
- Give each attempt its own result/log directory; never overwrite old attempts.
  Checkpoint atomically around build, validation and publication. Distinguish
  pending, running, validated, published, failed and blocked. Allow only one
  runner per campaign.
- Resume only compatible verified attempts: same source/patches, candidate
  digests, dependency inputs, suite/architecture/profiles and validation config.
  Reconcile interrupted builds and publications against logs and artifacts before
  retrying; an interrupted attempt is not a package failure. Changed prerequisite
  outputs require affected consumers to be revalidated. A changed resource knob
  alone does not invalidate compatible results.
- Maintain a compact machine-readable summary (completed/total, current sources,
  elapsed times, failures, blocked prerequisites, evidence paths) that monitors
  read without replaying logs or touching the queue. A running build with no new
  output is not by itself stalled.

Run the script as background work with its process identity and log paths
recorded; use a persistent job mechanism if it must outlive the session. Prefer
completion notifications over polling; if polling, read the summary first and
open log excerpts only when state changes. Host-specific polling intervals belong
in local configuration.

Public reports need the conditions tested and a meaningful failure excerpt, not a
manifest dump. A blocked graph node is not an FTBFS.

## References

- [Ben reference manual](https://salsa.debian.org/debian/ben/-/blob/master/doc/refman.txt): DSL and configuration.
- [Ben command options](https://manpages.debian.org/unstable/ben/ben.1.en.html).
- [Monitor implementation](https://salsa.debian.org/debian/ben/-/blob/master/frontends/ben_monitor.ml): output formats and levels adjacency.
- [Dependency extraction](https://salsa.debian.org/debian/ben/-/blob/master/lib/package.ml) and [source graph](https://salsa.debian.org/debian/ben/-/blob/master/lib/dependencies.ml): approximation boundaries.
