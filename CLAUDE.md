# trino-odbc

Requirements and decision record for an ODBC driver for Trino on Windows. There is no code
here yet; the deliverable so far is `SPEC.md`.

## State

- `SPEC.md` is the single source of truth: market survey (§1), requirements with stable
  IDs `F-`, `A-`, `T-`, `S-`, `W-`, `P-`, `N-` (§3–§4), test strategy (§5), open
  decisions (§6), the v1 cut (§7), sources and review log (§8), and a source-level gap
  analysis of stackabletech/stackable-odbc-trino (§9).
- Decision pending: adopt Stackable and contribute the gaps, contribute to
  trinodb/trino-odbc, or build new. The spec's recorded default is "adopt Stackable";
  it becomes final only after the §5 application smoke run on a Windows host.
- Next step: install Stackable v0.1.2 and run it under Excel (64 and 32-bit), Microsoft
  Query, an MSDASQL linked server and a Power BI Gateway; record what fails.

## Working rules for this repo

- Refer to requirements by ID. When a requirement changes, edit its row; do not add a
  second row with the same intent.
- Claims about an existing driver come from its source, not its README. Name the file
  and line in the spec's Evidence column. A keyword grep is not evidence; read the
  function body.
- Trino behaviour is checked against Trino's own source (grammar in
  `core/trino-grammar/.../SqlBase.g4`, protocol in `client/trino-client`) or a running
  server, at both ends of the supported version range.
- ODBC behaviour is checked against the Microsoft ODBC reference, and marked
  "uncertain" in the spec when it was not.
- Type-mapping rows in §3.6 are a compatibility contract; changing one after a release
  needs a migration note.
- Commit messages follow the Trino convention: imperative subject ≤ 50 characters, no
  trailing period, body wrapped at 72, no AI attribution trailers.

## Related

- Upstream candidates: https://github.com/trinodb/trino-odbc,
  https://github.com/stackabletech/stackable-odbc-trino,
  https://github.com/stackabletech/stackable-odbc-core,
  https://github.com/stackabletech/trino-rust-client (branch `stackable-main`).
- Reference commercial driver: https://docs.starburst.io/clients/odbc/odbc-v3/index.html
- The author maintains trinodb/trino-go-client; protocol lessons from it are cited in
  the spec as `trino-go-client #NNN`.
