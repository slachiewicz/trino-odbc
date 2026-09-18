# Trino ODBC driver for Windows: specification

A requirements specification and decision record for an ODBC driver that connects
Windows BI tools (Excel, Power BI Desktop and Gateway, Tableau, SSIS, Access) to
[Trino](https://trino.io). There is no driver code in this repository.

## Contents

- [`SPEC.md`](SPEC.md)
  - §1 survey of existing drivers: [trinodb/trino-odbc](https://github.com/trinodb/trino-odbc),
    [stackabletech/stackable-odbc-trino](https://github.com/stackabletech/stackable-odbc-trino),
    Starburst V3, Simba, CData
  - §3–§4 requirements with stable IDs: protocol (`F-`), authentication (`A-`), TLS
    (`T-`), SQL semantics (`S-`), ODBC surface and type mapping (§3.4, §3.6),
    connection parameters (§3.7), Windows integration (`W-`), Power BI connector (`P-`),
    non-functional (`N-`)
  - §7 the version 1 cut
  - §9 a source-level gap analysis of stackable-odbc-trino v0.1.2 against the spec
- [`CLAUDE.md`](CLAUDE.md): working rules for maintaining the spec

## Status

Draft 2, 2026-09-18. The recorded default is to adopt the Stackable driver and contribute
the gaps it has (Kerberos/SSPI, OAuth2 client credentials, `DESCRIBE OUTPUT`,
`DefaultVarcharLength`, ANSI code-page conversion, 32-bit build, signed MSI). That becomes
final after the driver has been run under Excel, Microsoft Query, an MSDASQL linked server
and a Power BI Gateway; see `SPEC.md` §5 and §6.

## License

[Apache License 2.0](LICENSE).
