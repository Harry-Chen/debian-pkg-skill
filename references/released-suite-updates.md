# Updates To Released Suites

Landing a security fix in a released Debian suite through the Security Team.
The upstream-source boundary rules in `SKILL.md` apply here too — this
workflow routinely enters the upstream repository to locate fix commits.

## Security Update

Port a published fix into one or more released suites. Goal: the smallest
verifiable diff that resolves the issue — no refactors, no unrelated cleanups,
no version bumps. This workflow ends at "patches written, series and changelog
updated"; build, lintian, autopkgtest, and upload follow the standard
`Pre-Upload Checklist` in `workflows.md` with the local builder.

### 1. Identify the issue

When the task names a *source package* rather than a single CVE, start from the
package overview page:

```
https://security-tracker.debian.org/tracker/source-package/<src>
```

It tabulates every known issue against each suite. The work-list is the rows
marked **vulnerable** for your target suite(s). `fixed` and `<not-affected>`
rows need nothing, and `<ignored>` or `<end-of-life>` rows are a decision not
to fix that suite. Include the rows flagged `no-dsa` or `<unimportant>`:
neither justifies an upload on its own, but once one is being prepared each
costs one more patch and leaves nothing known open. Treat the per-CVE detail
pages as the authoritative description; the overview table truncates.

Capture both names each issue may go by:

- **CVE-YYYY-NNNNN** if assigned.
- **TEMP-NNNNNNN-XXXXXX** if the security tracker only has a temporary name
  (the pre-CVE placeholder on security-tracker.debian.org).

Open `https://security-tracker.debian.org/tracker/<id>`. The page lists:

- A short description.
- Per-suite status (vulnerable / fixed) — source of truth for which releases
  need work and which one may already carry a reusable patch.
- A **fixed version** column. This names the *first Debian/upstream version
  that carries the fix* and is the key to finding the commit (see §3): it bounds
  the upstream git range to search (vulnerable version → fixed version).
- A **Notes** block with upstream advisory URLs, upstream commit links, and
  the Debian BTS report. The Notes are best-effort: frequently they contain
  only a GHSA / advisory URL and no commit hash — and the linked advisory page
  often omits the commit too. Do not assume the commit is findable from the
  tracker alone; §3 covers deriving it from the upstream repo.

Also capture the **Debian bug number** for `Closes:` in the changelog (a CVE
may have no Debian bug — that is fine, just omit `Closes:`).

### 2. Confirm scope with the user

Always ask which suites to target. Default: **stable + oldstable** security
suites (e.g. trixie-security and bookworm-security). Other suites are opt-in:

- unstable / experimental — often already fixed; confirm before duplicating.
- LTS / ELTS — different team and workflow; confirm before touching
  `dla-needed.txt` or producing `+deb11uN` uploads.

A multi-suite confirmation up front avoids redoing the port for the wrong
tree.

### 3. Source the fix

Cheapest source first:

1. **A reusable backport patch already shipped in another Debian release.**
   The tracker's "fixed" column is a hint, not a guarantee — a newer release
   can be marked fixed for several reasons:

   - It carries an explicit backport patch in `debian/patches/` (reusable).
   - It includes a newer upstream release that contains the fix in the
     `orig.tar.xz` (no targeted patch to copy).
   - The vulnerable code was rewritten or removed upstream, so the release
     is unaffected by construction (nothing to copy and nothing applies).

   Treat (a) as the only reusable case. Verify before fetching: open
   `debian/patches/series` and `debian/changelog` of that release and look
   for an entry matching the CVE / temp id / bug number. If both name it,
   the patch is reusable — pull its `.debian.tar.xz` (or the salsa
   `debian/<suite>` branch) and copy the patch file as it ships there,
   header included. If only
   `debian/changelog` mentions it as a "New upstream release" with no
   matching patch, skip to source (2): the fix lives in upstream code that
   may or may not exist in your older version, and you need to compare to
   the upstream commit anyway.

2. **Upstream commit(s) from the Notes block.** Most git forges serve a raw
   mbox patch by appending `.patch` to a commit URL — gitlab.gnome.org,
   salsa.debian.org, GitHub, cgit all support this:

   ```bash
   curl -sL <commit-url>.patch -o .tmp/<short-hash>.patch
   ```

   When the Notes (and the GHSA / vendor advisory they link) name *no* commit,
   derive it from a local clone of the upstream repo. The tracker's
   **fixed version** column bounds the search: a CVE vulnerable in 3.7.1 and
   fixed in 3.11.0 lives somewhere in `git log 3.7.1..3.11.0`. Grep subjects
   and bodies for the vulnerability's topic, then confirm against the upstream
   release notes for the fixed version:

   ```bash
   git log --oneline <vuln-tag>..<fixed-tag> | grep -iE '<topic keywords>'
   git log -1 --format='%H%n%s%n%n%b' <candidate>     # read the body to confirm
   git show <fixed-tag>:NEWS | grep -iA3 CVE-YYYY-NNNNN   # or news.rst/ChangeLog
   ```

   The release-notes entry is the tiebreaker: it states what the project
   considers *the* fix and frequently cites the merging PR, which disambiguates
   when several nearby commits touch the same file.

3. **Vendor advisory** (Red Hat, SUSE, distro-patches list) if no upstream
   fix exists yet.

If multiple commits make up the fix, fetch each separately — they often want
to land as separate patches.

A CVE the release notes describe as "fixed in X.Y.Z" is not necessarily one
revertable commit: it may be a multi-commit hardening *series* whose later
commits depend on earlier refactors (changed signatures, new helpers). Confirm
applicability with `patch --dry-run` early — if the head commit's hunks fail
because the surrounding code was already rewritten upstream, you are in the
hand-recreated-backport case (§4 structural divergence, §5 DEP-3 format), not a
clean cherry-pick.

### 4. Adapt the patch to the target version

Upstream `main` is usually newer than the suite version. Check both paths and
context before trusting the patch:

- **File paths.** Upstream may have moved or renamed files. Rewrite the
  `--- a/…` / `+++ b/…` headers to the path that exists in the target
  version; verify with `find` or `git log --follow` on the upstream repo.
- **Hunk context.** Even when surrounding lines match, line numbers drift.
  `patch --dry-run` reports `Hunk #N succeeded at <line> (offset M lines)` —
  tolerable but worth fixing so the apply lands clean. Edit the
  `@@ -X,Y +X,Y @@` headers to match the target source.
- **Structural divergence.** If upstream refactored, renamed symbols, or
  introduced helpers that don't exist in the older release, preserve the
  *security intent*, not the textual diff. Recreate the equivalent check and
  document the divergence in a DEP-3 `Note:` paragraph.

Validate by dry-running:

```bash
patch -p1 --dry-run < debian/patches/<new>.patch
```

### 5. Write the patch files

- **One file per upstream commit. Do not combine.** Keep the upstream commit
  boundary intact even when the commits fix the same issue, are small, and
  always travel together. The cost of splitting is zero; the cost of
  unmerging later — partial revert, bisecting which half broke a test,
  selectively backporting one half to another suite — is real. Combining
  cherry-picks also makes the patch stop matching what `git show <hash>`
  produces upstream, which is what reviewers will compare against.

- **Order patches by upstream commit order.** When multiple commits make
  up the fix (or one upload addresses several CVEs), apply them in the
  order upstream did — author date, or the topological order from
  `git log --oneline --reverse <fix-range>` on the upstream repo. This
  avoids "applies but doesn't compile" intermediate states and matches
  what a reviewer would see cherry-picking them by hand. Reflect the same
  order in `debian/patches/series`.

- **Keep upstream test hunks.** Cherry-picks usually include changes to
  upstream tests and CI manifests; keep them. They are the cheapest way to
  document the fix and to catch a regression on rebuild, and dropping them
  invites the package's behaviour to drift from upstream. Drop test hunks
  only when retaining them would force pulling in an even larger earlier
  commit (e.g. a new test harness introduced a few commits before the
  fix) — and note the omission in the patch.

- **Format.** The cherry-pick-vs-DEP-3 rule is in `policy.md` (*Source
  Format And Patches*). Two points specific to security work:

  - Do not append `Origin:`, `Bug:`, `Bug-Debian:`, or `Forwarded:` to a
    preserved cherry-pick just because it is a security fix. The `From <hash>`
    line identifies the commit and the changelog carries the CVE and bug
    citation, so the extra fields only make the file diverge from upstream and
    make a future re-import noisy to diff.

  - A hand-recreated backport must state in `Note:` exactly how it differs
    from upstream. The security team reviews the patch against the upstream
    fix, and an undocumented divergence is what makes that review expensive.

- **Filenames.** `CVE-YYYY-NNNNN.patch` when a single commit fixes a single
  CVE. With multiple commits per CVE, suffix each with the upstream short
  hash (`CVE-YYYY-NNNNN-d220aa2f.patch`) or an ordinal that matches
  upstream order (`CVE-YYYY-NNNNN-1.patch`, `-2.patch`). Without a CVE,
  use a descriptive slug (`sandbox-escape-1-no-ghelp-proc.patch`) or
  `bts-<bugnum>-*.patch`. Avoid temp tracker IDs in filenames — they are
  flagged "Not for external reference" and get superseded by a real CVE.

### 6. Wire into series and validate

Append the new patches at the end of `debian/patches/series` unless they need
to precede an existing Debian-only patch. Then exercise the full quilt cycle
in both directions:

```bash
quilt push -a   # all patches must apply
quilt pop -a    # and reverse cleanly
```

A `Hunk … succeeded at N (offset M lines)` warning is acceptable but signals
the line numbers should be tightened (see §4).

### 7. Update `debian/changelog`

Use **`dch --security`** (devscripts) — do not write the header by hand.
It opens a new stanza with the right shape derived from the current top
entry: the `+deb<N>u<M>` version suffix is incremented for the matching
suite, the distribution is set to `<codename>-security`, and urgency is
set to `high`. Hand-writing the header is error-prone — the suite number,
the `u<M>` counter, and the distribution name all have to agree, and
`dch --security` is the only thing that gets all three right at once.

```bash
cd <source-tree>
dch --security        # opens $EDITOR on a freshly-templated stanza
```

Then edit the body. Security uploads are per-suite source packages, so run
`dch --security` once per source tree (trixie tree, bookworm tree, …) —
not a single stanza spanning multiple suites.

Required content of the body:

- **NMU note** when the Security Team or another non-maintainer is
  uploading: start with `* Non-maintainer upload by the Security Team.`.
- **One bullet per CVE, id leading**: `* CVE-YYYY-NNNNN: <description>`, so
  DSA tooling and search indexers pick the id up. Name the patch file added
  in the bullet body. If the issue still has only a `TEMP-…` tracker id,
  cite it the same way; replace it with the real CVE on the next upload.
- **Debian bug closure** with `(Closes: #NNNNNN)` trailing the bullet for
  the issue it reports — this is what triggers the BTS to close the report
  when the upload reaches the archive.

Example shape (with a CVE assigned):

```
<source> (<version>+deb13u1) trixie-security; urgency=high

  * Non-maintainer upload by the Security Team.
  * CVE-YYYY-NNNNN: <one-line description of the vulnerability>.
    Add debian/patches/CVE-YYYY-NNNNN.patch, <what it changes>.
    (Closes: #NNNNNN)

 -- <Uploader Name> <email>  <RFC2822 date>
```
