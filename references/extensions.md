# Extension Points And Additional Debian Knowledge

Add to this file when a task reveals a recurring area that deserves its own reference.

## High-Value Resources

- DEP-14 branch layout: https://dep-team.pages.debian.net/deps/dep14/
- DEP-3 patch headers: https://dep-team.pages.debian.net/deps/dep3/
- Machine-readable copyright format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/
- Debian source package format: https://www.debian.org/doc/debian-policy/ch-source.html
- Debian archive suite policy and freeze updates: Developer's Reference and release team docs.
- Debian reproducible builds: https://reproducible-builds.org/docs/
- Debian Python policy: https://www.debian.org/doc/packaging-manuals/python-policy/
- Debian Perl policy: https://perl-team.pages.debian.net/policy.html
- Debian Java policy: https://www.debian.org/doc/packaging-manuals/java-policy/
- Debian Go team packaging: https://go-team.pages.debian.net/packaging.html
- Debian Rust team packaging: https://rust-team.pages.debian.net/book/
- Debian Med, Science, Python, Perl, GNOME, KDE, and other team policies where package ownership implies team practice.

## Possible Future Reference Files

Split these out when enough concrete tasks accumulate:

- `references/python.md`: pybuild, dh-sequence-python3, autopkgtest-pkg-pybuild, pytest, PEP 517.
- `references/rust.md`: debcargo, crate updates, vendoring, feature flags.
- `references/go.md`: dh-golang, Built-Using, vendoring, module paths.
- `references/c-library.md`: symbols files, shlibs, Multi-Arch, transitions.
- `references/kernel.md`: DKMS, module signing, linux-support packages.
- `references/debusine.md`: concrete CLI/API workflows once local credentials and usage patterns are known.
