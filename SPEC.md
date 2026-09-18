# Trino ODBC driver for Windows: requirements specification

Status: draft 2, 2026-09-18. Market survey and feature collection; reviewed once by a
fresh-context reviewer (corrections folded in, see §8). No implementation decision taken.

## 1. What already exists

Before writing a new driver, know that four open-source drivers and three commercial ones
already target Trino. Two of the open-source ones are usable on Windows today.

| Driver | Language | Windows | Status (2026-09-18) | Auth | Notes |
|---|---|---|---|---|---|
| [trinodb/trino-odbc](https://github.com/trinodb/trino-odbc) | C++ (libcurl, nlohmann-json) | x64, x86 MSI | v0.0.7 (2026-05-03), 22 stars, 9 open issues, roadmap issue [#30](https://github.com/trinodb/trino-odbc/issues/30) is empty | OAuth2 external (browser), OIDC client credentials, device code; basic auth is an open PR ([#33](https://github.com/trinodb/trino-odbc/pull/33)) | Official trinodb org (Corteva donation). Self-described "partially implemented": read-only, ANSI only (no wide-char), no transactions, single-row fetch only, everything reported nullable, no ARM64. 38 of 57 ODBC entry points are stubs. Targets Excel and Power BI Desktop only. |
| [stackabletech/stackable-odbc-trino](https://github.com/stackabletech/stackable-odbc-trino) on [stackable-odbc-core](https://github.com/stackabletech/stackable-odbc-core) | Rust | x64 zip + `install.bat` | v0.1.2 (2026-09-01), repos created 2026-07; 0 stars each; single-vendor; Apache-2.0 | Basic, JWT, mTLS, OAuth2 browser flow | ODBC 3.80 Unicode driver. Source read 2026-09-18: W-only exports (37), `SQLSetPos`, `SQL_ATTR_ROW_ARRAY_SIZE`, `SQL_OV_ODBC2` handling, `01S02` substitution, `SQL_GD_BLOCK`, `SQLCancel`; metadata from `system.jdbc.*`; intervals as `SQL_WVARCHAR`; unbounded varchar reports `SQL_NO_TOTAL` (no `DefaultVarcharLength`). README: transactions, cancel, spooled protocol, JDBC-parity parameters, Power BI `.mez` with DirectQuery folding, fuzzing. Tested by its authors with Power BI Desktop, pyodbc, isql against Trino 483. Not yet run by us under Excel, MS Query or MSDASQL. |
| [skilld-labs/trino-odbc](https://github.com/skilld-labs/trino-odbc) | C++ | dll | No release; 15 stars; issues ask "support non-Amazon Trino" | Azure AD, Okta | Fork of the Amazon Timestream ODBC driver retargeted at "Amazon Trino"; not general-purpose. |
| [facebookarchive/presto-odbc](https://github.com/facebookarchive/presto-odbc) | D | — | Archived | — | Historical only. |
| Simba/insightsoftware Trino ODBC | C++ (SimbaEngine) | 32/64 | Commercial | Kerberos, OAuth2, LDAP, JWT | ODBC 3.8, Unicode, full metadata, SQL passthrough; licensed per seat or OEM. |
| [Starburst ODBC V3](https://docs.starburst.io/clients/odbc/odbc-v3/index.html) | Starburst's own (V2 was Simba/Magnitude) | Windows only; macOS and Linux "on request" | Commercial (free download, Starburst-branded) | None, Password, Windows Integrated (Kerberos/NTLM via SSPI), Windows Password (user/password → Kerberos TGT), JWT, OAuth2 | Reference for the option set enterprise users expect. Certified against Power BI, Tableau (Desktop, Server, Cloud, Prep), SQL Server linked servers, Excel (import only). V3 dropped mTLS, keytabs, cert-revocation checks and `Min_TLS` that V2 had. Works against open-source Trino. |
| [CData Trino ODBC](https://www.cdata.com/drivers/trino/) | C++ | 32/64 | Commercial | Various | Adds SQL-92 rewriting, caching, write support. |

Other hits, not viable: `MinervaDB/Trino-ODBC-Driver-for-.NET` (a .NET client, not an ODBC
driver despite the name), `iscsrwm/trino-odbc` (C, one commit), `VargaFoundation/argus`
(generic lakehouse ODBC, early), `HaiTrieu0902/trino-odbc` (fork of trinodb's).

Implication: the gap the trinodb driver was written for (Excel and Power BI on Windows,
OAuth2) is now covered twice. Only three things are genuinely missing from the open-source
options, and all three are deliverable as contributions to Stackable as well as by a new
driver:

- Windows-native auth: Kerberos/SSPI "Windows Integrated" and "Windows Password", which
  is what Power BI Gateway SSO and every AD-joined desktop needs.
- Excel/Access/MSDASQL compatibility knobs, first of all `DefaultVarcharLength` (§3.7).
- Governance: a driver that trinodb ships and lists on trino.io, rather than a
  vendor-owned repo.

ARM64 and 32-bit are not differentiators: for a Rust driver they are CI-matrix targets
(`aarch64-pc-windows-msvc`, `i686-pc-windows-msvc`); trinodb closed its ARM64 PR
([#34](https://github.com/trinodb/trino-odbc/pull/34)) without merging. "Reuse an existing
client library" is an implementation preference, not a user-visible benefit.

Sections 2 to 7 collect the requirements regardless of which path is chosen: build new,
contribute to trinodb/trino-odbc, or adopt and contribute to Stackable.

## 2. Goals and non-goals

Goals:

- Let Windows BI and analytics tools (Excel, Power BI Desktop, Power BI Gateway, Tableau,
  Qlik, SSIS, Access linked tables, pyodbc, R/odbc, Alteryx) read from Trino through the
  Windows ODBC Driver Manager.
- Support the authentication mechanisms an enterprise Trino deployment uses: password
  (LDAP/file), JWT, OAuth2 browser flow, OAuth2 client credentials, Kerberos/SSPI,
  mutual TLS.
- Behave as a well-formed ODBC 3.8 Unicode driver so that generic tools work without a
  custom connector.

Non-goals (version 1):

- macOS and Linux builds (design for them, do not ship them).
- SQL dialect translation. The driver passes SQL through; Trino SQL is the contract.
  The one exception is ODBC escape sequences (S-3), which the ODBC contract requires the
  driver to translate.
- Bulk load, array-bound parameter insert, updatable cursors.
- Stored procedures, primary/foreign key metadata (Trino has none).

The v1 cut is in §7; §3 to §5 are the full inventory.

## 3. Functional requirements

### 3.1 Connectivity and protocol

| ID | Requirement | Source |
|---|---|---|
| F-1 | Implement the Trino client REST protocol: `POST /v1/statement`, `GET nextUri` polling loop, `DELETE` for cancel. Treat `status` as informational; completion is the absence of `nextUri`. | trino.io client-protocol |
| F-2 | Retry on 502/503/504 with backoff; honour `Retry-After` on 429; any other non-200 is a query failure. Retry budget configurable (`MaxAttempts`). | protocol doc; trino-go-client experience |
| F-3 | Send `X-Trino-User`, `X-Trino-Source`, `X-Trino-Catalog`, `X-Trino-Schema`, `X-Trino-Time-Zone`, `X-Trino-Language`, `X-Trino-Client-Info`, `X-Trino-Client-Tags`, `X-Trino-Trace-Token`, `X-Trino-Session`, `X-Trino-Role`, `X-Trino-Extra-Credential`, `X-Trino-Resource-Estimate`, `X-Trino-Transaction-Id`, `X-Trino-Prepared-Statement`, `X-Trino-Path`, `X-Trino-Client-Capabilities`. | protocol doc |
| F-4 | Apply response headers to connection state: `Set-Catalog`, `Set-Schema`, `Set-Path`, `Set-Session`, `Clear-Session`, `Set-Role`, `Set-Authorization-User`, `Reset-Authorization-User`, `Added-Prepare`, `Deallocated-Prepare`, `Started-Transaction-Id`, `Clear-Transaction-Id`. Multi-valued headers must be merged, not overwritten (trino-go-client #189). | protocol doc |
| F-5 | Advertise `X-Trino-Client-Capabilities: PATH, PARAMETRIC_DATETIME, SESSION_AUTHORIZATION`; make the list extensible (`ClientCapabilities`). | protocol doc |
| F-6 | Spooled protocol: opt-in via `Encoding=json`, `json+zstd`, `json+lz4`; handle inline and spooled segments, fetch from the segment URI with the segment headers, acknowledge segments. Off by default; fall back transparently when the coordinator does not spool. | Stackable README; Trino 466+ |
| F-7 | HTTP compression on by default (`DisableCompression` to turn off). | trinodb #3 |
| F-8 | System proxy by default (WinHTTP/IE settings); explicit HTTP proxy with optional basic credentials and a bypass list; SOCKS optional. | Starburst, JDBC |
| F-9 | Follow redirects only when `AllowHttpRedirect=true`; never forward auth headers to a different host. | Starburst; trino-go-client #207 |
| F-10 | Three distinct timeouts, not aliased: `LoginTimeout` → `SQL_ATTR_LOGIN_TIMEOUT` (connect only), `ConnectionTimeout` → `SQL_ATTR_CONNECTION_TIMEOUT` (per HTTP request), `QueryTimeout` → `SQL_ATTR_QUERY_TIMEOUT` (wall clock from execute to last row, then cancel). | ODBC spec; Stackable |
| F-11 | Metadata queries (`SQLTables`, `SQLColumns`, `SQLTablePrivileges`, `SQLGetTypeInfo`) run against `system.jdbc.tables`, `system.jdbc.columns`, `system.jdbc.catalogs`, `system.jdbc.schemas`, `system.jdbc.table_types`, `system.jdbc.types`, as the JDBC driver does; the server then handles `%`/`_` patterns, broken catalogs and nullability. | JDBC driver; Stackable source |
| F-12 | Prepare-time metadata: `SQLPrepare` followed by `SQLNumResultCols`/`SQLDescribeCol` before `SQLExecute` (SSIS, MSDASQL, Power BI do this) is answered with `DESCRIBE OUTPUT`; `SQLDescribeParam` with `DESCRIBE INPUT`. | Trino SQL |

### 3.2 Authentication

| ID | Requirement | Notes |
|---|---|---|
| A-1 | None (plain HTTP, user only) for dev clusters. | |
| A-2 | Password (HTTP Basic over TLS): LDAP, file, or Salesforce authenticators. Refuse to send a password over `http` unless an explicit `AllowInsecurePassword` flag is set. | trino-go-client #205 |
| A-3 | JWT bearer token (`AccessToken`). | |
| A-4 | OAuth2 external authentication (browser flow): parse `WWW-Authenticate: Bearer x_redirect_server=..., x_token_server=...`; open the system browser; poll the token server; cache the token per process so ten Power BI connections open one browser tab; token refresh for long queries. Handle multiple `WWW-Authenticate` headers (trinodb #39). In this flow the **coordinator** is the OAuth client and the IdP redirects to `https://<coordinator>/oauth2/callback`; the driver never sees the IdP. Desktop only: no browser exists on a Gateway host. | trinodb, Stackable |
| A-5 | OAuth2 client credentials (`ClientId`, `ClientSecret`, `TokenEndpoint`, `Scope`) for unattended use: Power BI Gateway, Report Server, scheduled refresh. | trinodb #21, #11, PR #45 |
| A-6 | OAuth2 device code flow for headless hosts. | trinodb PR #27 |
| A-7 | Kerberos via Windows SSPI ("Windows Integrated"): `AcquireCredentialsHandle(NULL, "Kerberos")` on the **calling thread** so it picks up the thread's impersonation token (Power BI Desktop alternate credentials and Gateway SSO via constrained delegation both impersonate); use the `Kerberos` SSP, not `Negotiate`, so an NTLM downgrade cannot happen (Trino rejects NTLM). `KrbServiceName` (default `HTTP`), principal pattern, canonical hostname toggle. Optional MIT GSSAPI path with keytab and credential cache. | Starburst, JDBC `Kerberos*`, review |
| A-7b | "Windows Password": `KerberosUsername`/`KerberosPassword` acquire a TGT through SSPI (`AcquireCredentialsHandle` with explicit `SEC_WINNT_AUTH_IDENTITY`) so a service account or gateway can use Kerberos without an interactive logon. | Starburst V3 `WindowsPassword` |
| A-8 | Mutual TLS: client certificate from a PEM/PKCS#12 file or the Windows certificate store. | JDBC `SSLKeyStore*` |
| A-9 | `SessionUser` impersonation and `X-Trino-Original-User`. | JDBC |
| A-10 | `ExtraCredentials` for connector-level credentials. | JDBC |
| A-11 | Credential storage: DSN passwords and client secrets encrypted with DPAPI; machine scope for System DSNs so services can read them (trinodb #24); user scope needs a loaded profile, which `NT SERVICE\PBIEgwService` does not have. | trinodb, review |

### 3.3 TLS

| ID | Requirement |
|---|---|
| T-1 | `TlsVerify` = `full` (default), `ca` (skip hostname match), `none`. |
| T-2 | Trust anchors: Windows certificate store by default; `Certificate` PEM path overrides. |
| T-3 | `HostnameInCertificate` override for load balancers. |
| T-4 | TLS 1.2 minimum; TLS 1.3 where the TLS library allows. Revocation checking follows the TLS stack default. |

### 3.4 ODBC API surface

Target ODBC 3.80 as a Unicode driver. The Driver Manager detects a Unicode driver by the
presence of the `SQLConnectW` export; there is no registration flag. Export the W set plus
the unsuffixed functions (`SQLAllocHandle`, `SQLBindCol`, `SQLFetch`, `SQLGetData`, ...).
Do not export A-suffixed functions: the Driver Manager converts ANSI *call arguments* (SQL
text, names) to W calls itself. It does **not** convert *bound data*: `SQL_C_CHAR` buffers
are the driver's job, in the active ANSI code page (`CP_ACP`, which is 65001 when the
process runs under the UTF-8 code page), signalled by `SQL_ATTR_ANSI_APP`. Provide a
`CharEncoding=utf8` override.

ODBC 2.x applications: `SQL_ATTR_ODBC_VERSION = SQL_OV_ODBC2` on the environment
switches the date/time type codes in `SQLDescribeCol`, `SQLColAttribute` and
`SQLGetTypeInfo` to 9/10/11 (from 91/92/93); the Driver Manager does not remap them.
MS Query (Excel "From Microsoft Query") and Access are 2.x applications. Legacy
`SQLSetConnectOption`/`SQLColAttributes` calls arrive already mapped, so the attribute
handlers must accept the legacy option values.

Core level (all required):

- Handles: `SQLAllocHandle`, `SQLFreeHandle`, `SQLSetEnvAttr`, `SQLGetEnvAttr`. One
  driver environment per process under connection pooling; `SQL_ATTR_RESET_CONNECTION`
  (3.8) on return to the pool; `CPTimeout` in ODBCINST.INI; no ODBC or thread work in
  `DllMain`; the Driver Manager may `FreeLibrary` the driver and load it again.
- Connect: `SQLConnectW`, `SQLDriverConnectW` (with `SQL_DRIVER_PROMPT` and
  `SQL_DRIVER_COMPLETE` opening the DSN dialog), `SQLDisconnect`. `SQLBrowseConnect`
  optional.
- Attributes: `SQLSetConnectAttrW`, `SQLGetConnectAttrW`, `SQLSetStmtAttrW`,
  `SQLGetStmtAttrW`. Must support `SQL_ATTR_AUTOCOMMIT`, `SQL_ATTR_CURRENT_CATALOG`,
  `SQL_ATTR_LOGIN_TIMEOUT`, `SQL_ATTR_CONNECTION_TIMEOUT`, `SQL_ATTR_QUERY_TIMEOUT`,
  `SQL_ATTR_MAX_ROWS`, `SQL_ATTR_ROW_ARRAY_SIZE`, `SQL_ATTR_ROW_BIND_TYPE`,
  `SQL_ATTR_ROWS_FETCHED_PTR`, `SQL_ATTR_ROW_STATUS_PTR`, `SQL_ATTR_METADATA_ID`,
  `SQL_ATTR_ANSI_APP`. Option substitution, not error: `SQL_ATTR_CURSOR_TYPE` (anything
  but forward-only), `SQL_ATTR_CONCURRENCY`, `SQL_ATTR_TXN_ISOLATION` return
  `SQL_SUCCESS_WITH_INFO` with SQLSTATE `01S02` and the substituted value; Excel and MS
  Query set `SQL_CURSOR_STATIC` and expect this. `SQL_ATTR_ASYNC_ENABLE`: report
  `SQL_AM_NONE` in `SQLGetInfo(SQL_ASYNC_MODE)` and `HYC00` if it arrives anyway. Unknown
  attributes (`SQL_ATTR_TRACE`, `SQL_ATTR_ENLIST_IN_DTC`, vendor ones) return `HY092` or
  `HYC00`, never crash.
- Execute: `SQLExecDirectW`, `SQLPrepareW`, `SQLExecute`, `SQLNativeSqlW`,
  `SQLNumParams`, `SQLBindParameter`, `SQLDescribeParam` (F-12), `SQLCancel`,
  `SQLCancelHandle`, `SQLCloseCursor`, `SQLFreeStmt`, `SQLMoreResults`, `SQLRowCount`,
  `SQLEndTran`.
- Results: `SQLNumResultCols`, `SQLDescribeColW`, `SQLColAttributeW`, `SQLBindCol`,
  `SQLFetch`, `SQLFetchScroll` (`SQL_FETCH_NEXT` only), block fetching with
  `SQL_ATTR_ROW_ARRAY_SIZE` > 1 in column-wise and row-wise binding, and
  `SQLSetPos(SQL_POSITION)` so `SQLGetData` can address a row of a block cursor
  (advertised as `SQL_GD_BLOCK`; the update/delete/add operations of `SQLSetPos` return
  `HYC00`). `SQLGetData`: `SQL_GD_ANY_COLUMN | SQL_GD_ANY_ORDER | SQL_GD_BLOCK | SQL_GD_BOUND`,
  chunked retrieval of long varchar/varbinary with `01004` on truncation and re-call
  semantics, `SQL_NO_TOTAL` only when the remaining length is genuinely unknown.
  `SQL_C_NUMERIC` output honours `SQL_DESC_PRECISION`/`SQL_DESC_SCALE` set on the ARD
  (default scale 0 truncates).
- Descriptors: `SQLGetDescFieldW`, `SQLSetDescFieldW`, `SQLGetDescRecW`, `SQLSetDescRec`,
  `SQLCopyDesc` (minimum viable; Excel and Power BI call `SQLGetDescField`).
- Diagnostics: `SQLGetDiagRecW`, `SQLGetDiagFieldW`, with correct SQLSTATEs, Trino
  error code as native error, Trino message text and query ID, and `SQL_DIAG_ROW_NUMBER`
  where known.
- Info: `SQLGetInfoW` (see 3.6 for the minimum list), `SQLGetFunctions`,
  `SQLGetTypeInfoW` (also with an ODBC 2 type code as argument).
- Catalog: `SQLTablesW` (including the `SQL_ALL_CATALOGS` / `SQL_ALL_SCHEMAS` /
  `SQL_ALL_TABLE_TYPES` enumerations), `SQLColumnsW`, `SQLPrimaryKeysW` (empty),
  `SQLForeignKeysW` (empty), `SQLStatisticsW` (empty), `SQLSpecialColumnsW` (empty),
  `SQLTablePrivilegesW`, `SQLColumnPrivilegesW`, `SQLProceduresW` (empty),
  `SQLProcedureColumnsW` (empty). Pattern arguments honour `_` and `%` and the
  `SQL_ATTR_METADATA_ID` identifier mode (F-11); provide
  `AssumeLiteralUnderscoreInMetadataCalls` for tools that pass raw names.
- Setup DLL: `ConfigDSNW`, `ConfigDriverW`, so the ODBC Data Source Administrator can
  add, edit and remove DSNs with a native dialog.

Threading: `SQLCancel` is called from a **different thread** while `SQLExecDirect` or
`SQLFetch` is blocked on the same statement handle; this is how Power BI cancels a
refresh. Every other call on one handle is serialised by the application; different
handles run concurrently. This rule constrains the runtime choice more than any other
(§6).

Explicitly unsupported, reported with `HYC00` (or `HY092` for a bad option), never
silently ignored: scrollable cursors, positioned updates, `SQLBulkOperations`, async
execution, multiple isolation levels.

### 3.5 SQL execution semantics

| ID | Requirement |
|---|---|
| S-1 | Pass SQL through unchanged apart from S-3. Do not strip comments or rewrite quoting. |
| S-2 | Parameters: implement `?` markers via Trino `PREPARE`/`EXECUTE ... USING` (explicit prepare) or `EXECUTE IMMEDIATE` (Trino 431+). Serialise each C type to a Trino literal with the correct type (typed literals for decimal, date, timestamp, varbinary, interval; escape strings; reject NaN/Inf for numeric unless as `nan()`/`infinity()`). Numeric-to-string conversions always use `.` as the decimal separator regardless of locale. This is where trino-go-client had wrong-results bugs (#202, #203, #206); test every type at both ends of the supported Trino version range. |
| S-3 | ODBC escape sequences: `{d '...'}`, `{t '...'}`, `{ts '...'}` and `{fn ...}` for the function set advertised in `SQLGetInfo` are required (Excel, Access and Power BI folding emit them); `{oj ...}`, `{escape '...'}`, `{call ...}` (reject) in a later release. `SQLNativeSql` returns the translated text. |
| S-4 | `SQLRowCount` from `updateCount` for DML; `-1` for SELECT. |
| S-5 | Transactions: `SQL_ATTR_AUTOCOMMIT=OFF` starts a Trino transaction on the next statement via `START TRANSACTION`; `SQLEndTran` issues `COMMIT`/`ROLLBACK`; carry `X-Trino-Transaction-Id`. If a statement fails inside a transaction, roll back and report, do not report commit success. Only one isolation level. Until implemented, report `SQL_TC_NONE` and treat `SQL_ATTR_AUTOCOMMIT=OFF` as `01S02`. |
| S-6 | Cancel: `SQLCancel` sends `DELETE nextUri` (or `partialCancelUri`) so the cluster stops the query; closing a statement with unfetched rows also cancels. |
| S-7 | Session management: `USE catalog.schema`, `SET SESSION`, `RESET SESSION`, `SET ROLE`, `SET PATH`, `SET TIME ZONE`, `SET SESSION AUTHORIZATION` all work through header propagation (F-4). |
| S-8 | Warnings from `QueryResults.warnings` surface as `SQL_SUCCESS_WITH_INFO` with SQLSTATE `01000`, unless `MuteServerWarnings`. |
| S-9 | `SQL_ATTR_MAX_ROWS`: implemented client-side (stop fetching and cancel), not by rewriting SQL. |

### 3.6 Type mapping

Every Trino type must map to an ODBC SQL type, an accurate `SQLDescribeCol` (size,
decimal digits, nullability, unsigned, searchable), and correct conversions to every
reasonable C type. This table is a compatibility contract from the first release:
Starburst changed the `with time zone` and `interval` mappings between V2 and V3 and had
to warn users that refreshing existing Excel workbooks changes values and layout.

Nullability is two different answers: result sets (`SQLDescribeCol`) report
`SQL_NULLABLE_UNKNOWN` because Trino gives no nullability for query output; `SQLColumns`
reports the real value from `system.jdbc.columns.NULLABLE`, which is what Power BI and
Access read.

| Trino | ODBC SQL type | Notes |
|---|---|---|
| boolean | SQL_BIT | |
| tinyint, smallint, integer, bigint | SQL_TINYINT (signed), SQL_SMALLINT, SQL_INTEGER, SQL_BIGINT | Trino tinyint is signed (trinodb #5). |
| real, double | SQL_REAL, SQL_DOUBLE | JSON `NaN`, `Infinity`, `-Infinity` arrive as strings; parse them. |
| decimal(p,s) | SQL_DECIMAL with p,s | Deliver as string and as SQL_NUMERIC_STRUCT; p up to 38. |
| varchar(n), char(n) | SQL_WVARCHAR(n), SQL_WCHAR(n) | UTF-8 to UTF-16; column size in characters, octet length in bytes. |
| varchar (unbounded), json, array, map, row, other text-rendered types | SQL_WVARCHAR(`DefaultVarcharLength`), default 2048 | **Not** SQL_WLONGVARCHAR: Access refuses long types in WHERE, Power BI folding disables comparisons on them, SSIS and MSDASQL allocate from the reported size. Values longer than the reported size are still delivered in full through `SQLGetData` chunking; `SQLBindCol` truncates with `01004`. Starburst V3 does the same; Stackable reports `SQL_NO_TOTAL`, which is correct by the spec and worse in practice. |
| varbinary | SQL_VARBINARY / SQL_LONGVARBINARY | Base64 in JSON. |
| date | SQL_TYPE_DATE | Years outside 1..9999 must not crash. |
| time(p), time(p) with time zone | SQL_TYPE_TIME (fractional seconds lost) | Deliver full precision as string too. Offset preserved as string for the tz variant. |
| timestamp(p) | SQL_TYPE_TIMESTAMP | p up to 12; SQL_TIMESTAMP_STRUCT holds nanoseconds; beyond 9 truncates. |
| timestamp(p) with time zone | SQL_TYPE_TIMESTAMP, converted to the session time zone; original text available via SQL_C_WCHAR | trinodb #7/#14; Starburst V3 mapping |
| interval year to month, interval day to second | SQL_WVARCHAR carrying Trino's text form | Starburst V3 moved intervals from BINARY to VARCHAR and Stackable maps them to WVARCHAR because BI tools cannot consume `SQL_INTERVAL_*`. Offer `SQL_C_INTERVAL_*` conversions on request. Sign handling for sub-second (trino-go-client #203). |
| uuid | SQL_GUID and SQL_WCHAR(36) | |
| ipaddress | SQL_WVARCHAR(45) | |
| HyperLogLog, P4HyperLogLog, QDigest, TDigest, KdbTree, Geometry, SphericalGeography, Color, CodePoints, Function, Bing tile, unknown | SQL_WVARCHAR(`DefaultVarcharLength`) or SQL_LONGVARBINARY as the JSON gives it | Never fail a `SELECT *` because of an unknown type. |

`SQLGetTypeInfo` must list the same set with `CREATE_PARAMS`, `LITERAL_PREFIX/SUFFIX`,
`SEARCHABLE` and `UNSIGNED_ATTRIBUTE` filled in, because Power BI and Tableau build
their SQL generation from it. `SQLColumns.TYPE_NAME` reports the bare name (`varchar`,
not `varchar(20)`) unless `TypeNameParameters=true`.

`SQLGetInfo` minimum list, because Power BI folding, Excel, MS Query and MSDASQL read
these before issuing any SQL: `SQL_DRIVER_NAME` (DLL file name), `SQL_DRIVER_VER`,
`SQL_DRIVER_ODBC_VER = "03.80"`, `SQL_DBMS_NAME = "Trino"`, `SQL_DBMS_VER`,
`SQL_SERVER_NAME`, `SQL_USER_NAME`, `SQL_DATA_SOURCE_NAME`,
`SQL_ODBC_INTERFACE_CONFORMANCE = SQL_OIC_CORE`, `SQL_SQL_CONFORMANCE`, `SQL_SQL92_*`,
`SQL_IDENTIFIER_QUOTE_CHAR = "\""`, `SQL_IDENTIFIER_CASE`, `SQL_QUOTED_IDENTIFIER_CASE`,
`SQL_SPECIAL_CHARACTERS`, `SQL_CATALOG_TERM = "catalog"`, `SQL_SCHEMA_TERM = "schema"`,
`SQL_CATALOG_NAME_SEPARATOR = "."`, `SQL_CATALOG_USAGE`, `SQL_SCHEMA_USAGE`,
`SQL_CATALOG_LOCATION`, `SQL_OJ_CAPABILITIES`, `SQL_GROUP_BY`,
`SQL_ORDER_BY_COLUMNS_IN_SELECT`, the `SQL_*_FUNCTIONS` bitmasks (numeric, string,
timedate, system, aggregate, `SQL_TIMEDATE_ADD_INTERVALS`), `SQL_CONVERT_*`,
`SQL_DATETIME_LITERALS`, `SQL_MAX_*` (identifier, column name, table name, statement
length, columns in select/group by/order by), `SQL_TXN_CAPABLE`,
`SQL_DEFAULT_TXN_ISOLATION`, `SQL_TXN_ISOLATION_OPTION`,
`SQL_CURSOR_COMMIT_BEHAVIOR`/`ROLLBACK_BEHAVIOR`, `SQL_GETDATA_EXTENSIONS`,
`SQL_FORWARD_ONLY_CURSOR_ATTRIBUTES1/2`, `SQL_SCROLL_OPTIONS`, `SQL_DESCRIBE_PARAMETER`,
`SQL_MULT_RESULT_SETS`, `SQL_NEED_LONG_DATA_LEN`, `SQL_ASYNC_MODE = SQL_AM_NONE`,
`SQL_CONCAT_NULL_BEHAVIOR`, `SQL_NULL_COLLATION`, `SQL_LIKE_ESCAPE_CLAUSE`.

### 3.7 Connection parameters

Use the JDBC names where one exists so users can copy from a JDBC URL, plus the ODBC
conventional aliases (`UID`, `PWD`, `Server`, `Database`).

Core: `Host`/`Server`, `Port`, `Protocol` (`https` default), `User`/`UID`,
`Password`/`PWD`, `Catalog`/`Database`, `Schema`, `Source`, `ClientTags`, `ClientInfo`,
`TraceToken`, `TimeZone`, `Locale`, `Path`, `SessionProperties`, `ResourceEstimates`,
`ExtraCredentials`, `ExtraHeaders`, `Roles`, `SessionUser`, `ClientCapabilities`.

Auth: `AuthType` (`none`, `password`, `jwt`, `oauth2`, `clientcredentials`,
`devicecode`, `kerberos`, `kerberospassword`, `mtls`), `AccessToken`,
`ExternalAuthentication`, `ExternalAuthenticationTimeout`,
`ExternalAuthenticationTokenCache` (`none`, `memory`, `dpapi`), `ClientId`,
`ClientSecret`, `TokenEndpoint`, `Scope`, `KrbServiceName`, `KerberosPrincipal`,
`KerberosServicePrincipalPattern`, `KerberosUseCanonicalHostname`, `KerberosDelegation`,
`KerberosUsername`, `KerberosPassword`, `ClientCertificate`, `ClientCertificatePassword`.

TLS and network: `TlsVerify`/`SSLVerification`, `Certificate`, `HostnameInCertificate`,
`UseSystemProxy` (default true), `Proxy`, `ProxyUser`, `ProxyPassword`, `ProxyBypass`,
`AllowHttpRedirect`, `DisableCompression`, `MaxAttempts`, `LoginTimeout`,
`ConnectionTimeout`, `QueryTimeout`.

Behaviour: `Encoding`, `AssumeLiteralUnderscoreInMetadataCalls`, `IgnoreBrokenCatalog`
(default true; Starburst V3 changed the default because one dead connector otherwise
hides every table), `AllowMetadataFromMultipleCatalogs` (Starburst V3, default true;
false restricts `SQLTables`/`SQLColumns` to the connection catalog so a 40-catalog
cluster does not take a minute to open in Power BI), `DefaultVarcharLength` (Starburst
V3, default 2048; see 3.6), `TypeNameParameters`, `CharEncoding`, `ExplicitPrepare`,
`ValidateConnection`, `DatabaseAsSchema` (report schemas of the default catalog as
databases for tools with a two-level model), `RowsPerFetch`, `MuteServerWarnings`
(Starburst V2; some tools turn every `SQL_SUCCESS_WITH_INFO` into a dialog).

Knobs Starburst V2 shipped and V3 dropped, listed so the decision is deliberate:
`ClientCert`/`ClientPrivateKey`/`TwoWaySSL` (mTLS: keep, Stackable has it),
`KerberosKeytab` (drop; SSPI covers Windows), `CheckCertRevocation` and `Min_TLS`
(fold into TLS defaults), `MaxCatalogNameLen`/`MaxColumnNameLen`/`MaxTableNameLen`/
`MaxSchemaNameLength`/`MaxComplexTypeColumnLength`/`MaxPreparedStatementLength`
(`SQLGetInfo` answers; do not make them tunable), `UseEqualInMetadataFilters`
(same as `AssumeLiteralUnderscoreInMetadataCalls`), `UseDSNSchemaForMetadata`,
`RemoveTypeNameParameters` (inverted into `TypeNameParameters`), `ConnectionTest`,
`AutoIPD`.

Defaults worth copying from Starburst V3: `ApplicationName`/`Source` defaults to the
calling process executable name, with the caveat that under Power BI, Excel Get Data and
the Gateway the calling process is the mashup container
(`Microsoft.Mashup.Container.NetFX45.exe`), so the Power BI connector must set `Source`
explicitly; `KrbServiceName` defaults to `HTTP`; log files default to `%USERPROFILE%`,
50 files × 20 MB; `CacheOAuthToken` defaults to true.

Logging: `LogLevel` (off, error, warn, info, debug, trace), `LogPath`, `LogMaxFiles`,
`LogMaxSizeMB`; driver-wide, also settable through the registry and an environment
variable so support can ask a user to turn it on without editing the DSN (trinodb #16).
Secrets redacted at every level.

Values containing `;` use `{...}` in connection strings and are stored bare in DSNs;
document this explicitly (Stackable's README shows the confusion it causes).

### 3.8 Windows integration

| ID | Requirement |
|---|---|
| W-1 | Deliverables: 64-bit driver DLL for everything; 32-bit driver DLL for 32-bit Excel, Access and MS Query only (Power BI Desktop and the Gateway are 64-bit only since 2021); ARM64 DLL as a CI target. Setup DLL combined or separate. |
| W-2 | Signed MSI per architecture, built with WiX using the native `ODBCDriver` element (MSI `ODBCDriver` table, which calls the ODBC installer API) rather than a custom action, so the existing drivers list is never clobbered (trinodb #36). 64-bit registration lands under `HKLM\SOFTWARE\ODBC\ODBCINST.INI`, 32-bit under `HKLM\SOFTWARE\WOW6432Node\ODBC\ODBCINST.INI`. Driver registration is per-machine only; a per-user MSI cannot install an ODBC driver, so admin rights are a documented prerequisite. Silent install (`/qn`) with properties for Intune/SCCM. `winget` manifest later. |
| W-3 | DSN dialog: all parameters from 3.7 grouped in tabs (Connection, Authentication, TLS, Advanced, Logging), a Test button that executes `SELECT 1`, and a "copy connection string" button. Must render correctly at 150% and 200% DPI. |
| W-4 | Runs under a service account with no interactive desktop: Power BI Gateway (`NT SERVICE\PBIEgwService`, no loaded user profile), Power BI Report Server, SSIS on SQL Agent, Tableau Server. No browser flow attempted when `AuthType` is not `oauth2`; System DSN secrets readable by services (trinodb #21, #11, #24). |
| W-5 | Static-link or private-copy all dependencies (TLS, JSON, compression); no dependency on a system OpenSSL or a redistributable the user has to install separately. Prefer SChannel for TLS so the Windows certificate store, revocation checking and corporate proxies work without configuration. |
| W-6 | Power BI custom connector; requirements in 3.9. Note that Excel does not load custom connectors at all: Excel's folding depends solely on the driver's `SQLGetInfo`/`SQLGetTypeInfo` answers. |
| W-7 | Tableau `.tdc` file to tune SQL generation; Starburst certifies Desktop, Server, Cloud and Prep Builder, so the SQL Tableau generates for each is a test fixture. |
| W-8 | Excel Get Data > From ODBC and From Microsoft Query both work with a System and a User DSN and with a connection string only, and refresh with saved credentials. |
| W-9 | File logging mandatory; ETW optional. |
| W-10 | SQL Server linked server (`sp_addlinkedserver` over MSDASQL) works for four-part-name queries and `OPENQUERY`; this exercises `SQLGetInfo` catalog-usage flags, `SQLColumns` and prepare-time metadata (F-12) more strictly than BI tools do. |
| W-11 | Code signing: Authenticode with timestamp on every DLL, the MSI and the `.pqx`; expect SmartScreen reputation warnings for a new publisher until it accrues. |
| W-12 | Microsoft Store (MSIX) Office: registry virtualisation may hide third-party ODBC drivers from Store-installed Excel. Uncertain; test and document. |

### 3.9 Power BI connector

Power BI is the tool every existing driver was written for, and Starburst's connector
history ([docs](https://docs.starburst.io/clients/powerbi.html), versions 3.0 to 5.4)
is a list of what users hit in practice. Starburst's connector is built into Power BI
Desktop 2.122+ and shows up as "Starburst connector" and "Starburst secured by Entra
ID"; a Trino connector starts as a `.mez` in the Custom Connectors folder (needs the
"allow any extension" security setting) and in production ships as a signed `.pqx`
(MakePQX) trusted via `TrustedCertificateThumbprints`.

| ID | Requirement | Source |
|---|---|---|
| P-1 | Own entry in Get Data with Import and DirectQuery modes; DirectQuery is the recommended mode. Import is bounded by the Power BI dataset limit (1 GB for Pro workspaces), not by the driver. | Starburst docs |
| P-2 | Query folding for filters, joins, group-by, top-N and timestamp comparisons, via `Odbc.DataSource` overrides (`SqlCapabilities`, `SQLGetInfo`, `SQLGetTypeInfo`, `SQLGetFunctions`). Use `LimitClauseKind.Limit`, not `LimitOffset`: Power BI emits `LIMIT n OFFSET m`, Trino's grammar requires `OFFSET m LIMIT n`. Timestamp folding was a Starburst bug fix (5.2.1), so it needs a test. | Starburst 5.2.1; review |
| P-3 | Credential kinds: Basic (password), Key (JWT), Windows (Kerberos/SSPI through thread impersonation, A-7), and OAuth2 in one of two distinct flows: (a) driver-side Trino external authentication (A-4), Desktop only; (b) connector-side OAuth implemented in M (`StartLogin`/`FinishLogin`) with the IdP redirect `https://oauth.powerbi.com/views/oauthredirect.html`, which is what works in the Power BI Service and is what the "secured by Entra ID" variant does. | Starburst 5.0, 5.4; review |
| P-4 | Lazy metadata: do not enumerate every catalog, schema and table when the navigator opens; fetch children on expand. A Catalog field in the dialog to scope the tree. | Starburst 5.2.1 "Lazy Evaluation of Metadata" |
| P-5 | Broken-catalog tolerance: a connector that fails `SHOW SCHEMAS` must not blank the whole navigator. | Starburst 4.0.0 |
| P-6 | Regex filter for catalogs and schemas shown in the navigator. | Starburst 5.0.0 |
| P-7 | Native query (custom SQL as source) with the Power BI "native database queries" approval flow. | Starburst 5.2.1 |
| P-8 | Connection-string passthrough field for extra keywords, rejecting confidential ones (`PWD`, `AccessToken`, `ClientSecret`). | Starburst 5.2.1 |
| P-9 | Role and session properties fields; a "use system proxy" toggle. | Starburst 5.3 |
| P-10 | Query cancellation when the user cancels a refresh or navigates away (cross-thread `SQLCancel`, 3.4). | Starburst 5.3 |
| P-11 | On-premises data gateway: same connector and driver installed on the gateway host, custom-connector folder configured in the gateway settings, connection created under Settings > Manage connections and gateways, credentials stored by the gateway. No browser is available, so the credential kinds are password, key, client credentials and Windows (Kerberos SSO via constrained delegation). | Starburst docs |
| P-12 | Certificate handling exposed in the dialog: system trust store (default), custom PEM, and an explicit "accept self-signed" that maps to `TlsVerify=none`. | Starburst docs |

## 4. Non-functional requirements

| ID | Requirement |
|---|---|
| N-1 | Memory safety: the driver runs inside Excel and Power BI; a crash takes the host down. Buffer-overflow bugs already occurred in the C++ driver (trinodb #22, #43). Either use a memory-safe language or fuzz every buffer-writing path (`SQLGetData`, `SQLBindCol`, diagnostics, `SQLGetInfo` string returns). |
| N-2 | Threading: `SQLCancel` from another thread against a blocked statement (3.4); connection pooling compatible; multiple statements per connection sequentially; multiple connections in parallel from one process (Power BI opens several). |
| N-3 | Throughput: stream rows from the JSON pages without materialising the full result; block fetch of 10 M rows of 10 columns should be bounded by network, not driver CPU. Publish a benchmark against the same query via the CLI. |
| N-4 | Startup: `SQLDriverConnect` to first row of `SELECT 1` under 500 ms on a warm cluster (excluding OAuth login). |
| N-5 | Supported Trino versions: latest LTS and latest release, tested against both, and the oldest version the users run (state it). |
| N-6 | Diagnostics: every failure has an SQLSTATE, the Trino error code, the Trino query ID, and the message; the log at `debug` records every request and header (secrets redacted). |
| N-7 | Supply chain: reproducible build in CI, Authenticode signing (W-11), dependency audit; SBOM and OpenSSF Scorecard later. |
| N-8 | License: Apache-2.0, matching Trino. |

## 5. Test strategy

- Unit tests for type conversion (every Trino type × every C type) and literal
  serialisation, including negative sub-second intervals, decimal(38,0), timestamp(12),
  timezones with historical offsets, empty strings, NULL in every type, `SQL_OV_ODBC2`
  type codes.
- Integration tests in CI against Trino in Docker (Windows runners can run Linux
  containers via WSL2 or a remote host); both ends of the version range.
- ODBC conformance: run the driver through `odbctest` from the Windows ODBC SDK and
  Microsoft's ODBC Test tool scripts; pyodbc test suite; `isql`.
- Application smoke tests, recorded manually per release: Excel 365 (32 and 64-bit,
  Get Data and Microsoft Query), Power BI Desktop import and DirectQuery, Power BI
  Gateway refresh under `PBIEgwService`, MSDASQL via a SQL Server linked server, Access
  linked table, Tableau Desktop, SSIS ODBC source.
- Fuzzing of the JSON response parser and the connection-string parser.
- Crash-free run under Application Verifier and PageHeap.

## 6. Open decisions

| Decision | Options | Assessment |
|---|---|---|
| Build, contribute, or adopt | (a) new driver; (b) contribute to trinodb/trino-odbc; (c) adopt Stackable, contribute SSPI, `DefaultVarcharLength`, the 32-bit/ARM64 CI matrix, and negotiate a trinodb home | (b) needs a rewrite of the fetch and Unicode layers, which is most of a driver. (c) is the shortest path to a complete driver; the honest description is "vendor-owned, single-vendor-maintained, two months old, Apache-2.0, forkable". Its source survives the checks in §1; it has not yet been run under Excel, MS Query, MSDASQL or a Gateway host. (a) only wins if (c) fails that run. |
| Language | C++ (trinodb, Simba), Rust (Stackable), Go via `-buildmode=c-shared` on top of trino-go-client | The Go option carries five independent risks that a spike under `isql` would not surface: a Go runtime inside a 32-bit Excel process (address-space reservation, GC threads); a DLL the Driver Manager may `FreeLibrary` (a Go DLL cannot be unloaded); a process-wide vectored exception handler inside a .NET host (the mashup container); cgo cross-toolchains for 386 and ARM64; and no CNG/Windows-store client-certificate support in `crypto/tls`. The spike must run under 32-bit Excel and the Gateway. Reusing trino-go-client is a development convenience, not a user benefit. |
| TLS stack | SChannel, OpenSSL, rustls | SChannel gives the Windows cert store, revocation checking and Kerberos SSPI for free. |
| Governance | trinodb org, own org, Stackable org | trinodb membership carries the "official" label users search for and a listing on trino.io/ecosystem. |
| Power BI connector | ship with driver or separate repo | Stackable ships it in the driver archive. |

Next step before any of these is decided: install Stackable v0.1.2 on a Windows host and
run the §5 application smoke tests (Excel 64 and 32-bit, MS Query, MSDASQL linked server,
Gateway refresh), recording which fail. That turns the adopt option's README claims into
evidence of the same quality this document has for trinodb.

## 7. Version 1 cut

Target: ships in about three months; Excel, Power BI Desktop, Power BI Gateway;
password, JWT, OAuth2 (driver-side), Kerberos/SSPI.

Stays: F-1..F-5, F-7, F-9, F-10, F-11, F-12; A-1..A-5, A-7, A-7b, A-9, A-10; T-1..T-4;
S-1..S-4, S-6..S-9 (S-5 as `SQL_TC_NONE`); §3.6 in full; W-1 (x64 and x86, ARM64 only
if the toolchain gives it for free), W-2 (MSI, silent install; no winget), W-3, W-4, W-5,
W-6, W-8, W-9 (file only), W-11; P-1..P-5, P-7, P-10, P-11, P-12; N-1, N-2, N-5, N-6,
N-8.

Goes to a later release: F-6 (spooled protocol), F-8 beyond system proxy, A-6, A-8,
A-11, S-5 transactions, the `{oj}`/`{escape}`/`{call}` part of S-3, W-7, W-10, W-12,
P-6, P-8, P-9, N-3 benchmark publication, N-4, N-7 SBOM and Scorecard, `SQLBrowseConnect`
and `ConfigDriverW`.

## 8. Sources and review

- https://github.com/trinodb/trino-odbc (README, issues 1–43, PRs 2–45; source read
  2026-09-18: 57 entry-point files, `curl`, `nlohmann-json`, `gtest` via vcpkg)
- https://github.com/stackabletech/stackable-odbc-trino and
  https://github.com/stackabletech/stackable-odbc-core (README v0.1.2; source grepped
  2026-09-18 for the entry points and mappings named in §1 and §3.6)
- https://github.com/skilld-labs/trino-odbc
- https://github.com/facebookarchive/presto-odbc
- https://docs.starburst.io/clients/odbc/odbc-v3/index.html
- https://docs.starburst.io/clients/odbc/odbc-v3/configuration.html
- https://docs.starburst.io/clients/odbc/odbc-v3/connection-string-keywords.html
- https://docs.starburst.io/clients/odbc/odbc-v3/mapping-v2-keywords.html
- https://docs.starburst.io/clients/powerbi.html
- https://insightsoftware.com/drivers/trino-odbc-jdbc/
- https://www.cdata.com/drivers/trino/
- https://trino.io/docs/current/develop/client-protocol.html
- https://trino.io/docs/current/client/jdbc.html
- https://trino.io/ecosystem/client-driver.html

Draft 1 was reviewed 2026-09-18 by a fresh-context reviewer; the corrections folded in
here are: Unicode driver detection and the `SQL_C_CHAR` conversion responsibility (3.4);
`01S02` option substitution (3.4); `SQLSetPos(SQL_POSITION)` with `SQL_GD_BLOCK` (3.4);
`SQL_OV_ODBC2` type codes (3.4); `DESCRIBE OUTPUT`/`INPUT` and `system.jdbc.*` (F-11,
F-12); cross-thread `SQLCancel` (3.4, N-2); the three timeouts (F-10); unbounded varchar
and interval mappings, and the nullability split (3.6); the mashup-container process name
(3.7); WiX `ODBCDriver`, `WOW6432Node`, per-machine only (W-2); 32-bit scope (W-1); the
two Power BI OAuth flows and SSPI thread impersonation (A-7, P-3); `LimitClauseKind`
(P-2); the 1 GB figure (P-1); Go runtime risks and the Stackable assessment (§6); the
v1 cut (§7). Claims the reviewer marked uncertain and which remain unverified: Driver
Manager gating of `SQL_ATTR_ASYNC_ENABLE` on `SQL_ASYNC_MODE`; MSIX Office registry
virtualisation (W-12).

## 9. Gap analysis: stackable-odbc-trino v0.1.2 against this spec

Source read 2026-09-18: `stackable-odbc-trino` (35 k lines Rust), `stackable-odbc-core`
v0.1.0 (95 k lines), and the protocol layer, which is Stackable's fork of
`trino-rust-client` (branch `stackable-main`, 7.4 k lines, `reqwest` 0.13 on rustls with
`rustls-platform-verifier`). 3 076 unit tests, fuzz targets, a Python integration suite
with Windows VM tests (`integration-tests/windows/`), SBOMs. Toolchain: Rust 1.97.1,
Windows release built with `x86_64-pc-windows-gnu` (MinGW), not MSVC.

### 9.1 Present and verified in source

F-1, F-3 (all headers except `X-Trino-Original-User`), F-4 (`Set-Catalog/Schema/Path/
Session/Role`, `Clear-Session`, `Started/Clear-Transaction-Id`, `Added/Deallocated-Prepare`
handled in the client fork; multi-valued headers read with `get_all`), F-5
(`PARAMETRIC_DATETIME`, `PATH`, plus `ClientCapabilities`), F-6 spooling (json, zstd,
lz4, segment fetch), F-7 (`DisableCompression` wired to `Accept-Encoding: identity`),
F-9 redirects, F-11 (`system.jdbc.*`), half of F-12 (`DESCRIBE INPUT` for
`SQLDescribeParam`); A-2, A-3, A-4 (process-shared token), A-8 (PEM only), A-9, A-10;
T-1 (`ca` mode needs an explicit PEM, a rustls limitation, documented), T-2 (platform
roots by default), T-4; the whole 3.4 core surface including `SQLSetPos(SQL_POSITION)`,
`SQL_GD_BLOCK`, `SQL_OV_ODBC2` type codes, `01S02` substitution, `SQLBrowseConnect`,
`SQLCancelHandle`, `SQLMoreResults`, `SQLNativeSql`, `SQLCopyDesc`, `SQL_C_NUMERIC` from
the ARD, `SQL_ATTR_RESET_CONNECTION`, `SQL_ATTR_METADATA_ID`, `SQL_ATTR_MAX_ROWS`,
row-wise binding, `ConfigDSN`; S-1, S-2 (single and batched parameters), S-3 (`{fn}`,
`{d}`, `{t}`, `{ts}`, `{oj}`, `{escape}`; `{call}` rejected), S-4, S-5 transactions,
S-6 cancel, S-7, S-8 warnings, S-9; 3.6 with intervals as `SQL_WVARCHAR`; N-1 (Rust,
fuzzing, panic guard), N-2, N-7 (SBOM, `cargo audit`, Scorecard), N-8; Power BI
connector with `SqlCapabilities`/`SQLGetInfo`/`SQLGetTypeInfo` overrides and a folding
contract test.

### 9.2 Missing or wrong, by impact

| Spec ID | Gap | Evidence | Effort |
|---|---|---|---|
| A-7, A-7b | No Kerberos at all: no SSPI, SPNEGO or GSSAPI code in driver, core or client fork; the only "kerberos" hits are comments. reqwest/rustls has no Negotiate support, so this is a new auth path in the client fork plus a Windows-only SSPI module. Blocks every AD-joined desktop and Gateway SSO. | grep of all three repos | Large |
| A-5 | No OAuth2 client-credentials flow (`ClientId`/`ClientSecret`/`TokenEndpoint`); external auth is the browser flow only. Blocks unattended Gateway and Report Server refresh with OAuth. | `connect_params.rs`, `auth/oauth2.rs` | Medium |
| A-6 | No device-code flow. | same | Small |
| F-12 | `SQLPrepare` runs nothing; `SQLNumResultCols`/`SQLDescribeCol` before `SQLExecute` cannot describe the result. No `DESCRIBE OUTPUT`. Breaks SSIS, MSDASQL linked servers, Access and any tool that describes before executing. | `backend/execute.rs:318` "preparing runs no query on the coordinator" | Medium |
| 3.6 | Unbounded `varchar`, `json`, `array`, `map`, `row` report `SQL_NO_TOTAL` column size; no `DefaultVarcharLength`. Correct by the ODBC spec, but Excel, Access, SSIS and MSDASQL allocate from the reported size. | `type_conversion.rs:460-492` | Small |
| 3.4 | `SQL_C_CHAR` is delivered as UTF-8 bytes regardless of the process code page; no `SQL_ATTR_ANSI_APP`, no `CP_ACP` conversion. Non-ASCII text in ANSI applications (MS Query, Access, VBA, 32-bit ODBC apps) is mojibake unless the machine runs the UTF-8 code page. | `column_value.rs:237` "Bytes: UTF-8 for SQL_C_CHAR" | Medium |
| P-2 | Connector sets `LimitClauseKind.LimitOffset` with the comment "Trino uses LIMIT x OFFSET y syntax". Trino's grammar is `OFFSET m LIMIT n` (`SqlBase.g4` `queryNoWith`), so any folded step with an offset is a parse error. Should be `LimitClauseKind.Limit` or an `AstVisitor` override. | `connector/StackableTrinoODBC.pq:65`; grammar | Small; upstream bug report |
| P-3, P-11 | Connector credential kinds are `UsernamePassword` and `Implicit` only: no `Key` (JWT), no `Windows`, no M-side OAuth (`StartLogin`/`FinishLogin`), so no OAuth in the Power BI Service or through a Gateway. | `StackableTrinoODBC.pq:465-467` | Medium (Key), Large (M OAuth) |
| P-4, P-6, P-7, P-8, P-9 | No lazy navigation beyond `Odbc.DataSource` defaults, no catalog regex filter, no native-query entry, no connection-string passthrough, no roles/session-properties fields in the dialog (they exist as record options only). | `StackableTrinoODBC.pq` | Small each |
| W-2, W-11 | No MSI; `install.bat` copies the DLL and writes the registry. No Authenticode signing. Not deployable through Intune/SCCM without wrapping. `.mez` only, no signed `.pqx`. | `packaging/windows/` | Medium |
| W-1 | x64 only, and the release is built with the GNU (MinGW) target rather than MSVC. No 32-bit (`i686`) build, so 32-bit Excel, Access and MS Query cannot load it; no ARM64. Core's handle registry already accounts for 32-bit pointers, so the port is a CI-matrix change plus an MSVC or `gnullvm` toolchain decision. | `.github/workflows/build.yaml`, `registry.rs:36` | Small to medium |
| W-3 | `ConfigDSN` launches `configure-dsn.ps1` (WinForms via PowerShell) rather than a native dialog. Works, but depends on PowerShell execution policy and a `%ProgramFiles%` script path, and is not what an admin expects from an ODBC driver. | `install.bat`, `configure-dsn.ps1` | Medium to replace |
| A-11 | DSN secrets are stored as plain registry values (Save box off by default, tooltip warns). No DPAPI. System DSNs for services therefore either hold plaintext in HKLM or need the application to supply the secret. | `configure-dsn.ps1:770-784` | Small |
| 3.7 logging | Logging is `ODBC_LOG_LEVEL`/`ODBC_LOG_FILE` environment variables only; no DSN or registry keys, no rotation, default stderr. A support engineer cannot ask a user to "set LogLevel in the DSN". | `core/src/logging.rs:63-71` | Small |
| F-2 | Retries cover 502/503/504 and connect/timeout errors; HTTP 429 with `Retry-After` is not handled. | `client/mod.rs:251-282` | Small |
| F-8 | Proxy: explicit HTTP/HTTPS with basic auth only. No system-proxy (WinHTTP/IE) detection, no bypass list, SOCKS rejected. Corporate desktops with PAC files need manual configuration. | `proxy.rs` | Small to medium |
| F-10 | `LoginTimeout` is an alias of `QueryTimeout`; no separate `SQL_ATTR_LOGIN_TIMEOUT`/`SQL_ATTR_CONNECTION_TIMEOUT` semantics. | README table | Small |
| F-3, F-4 | No `X-Trino-Original-User`, `Set-Authorization-User`, `Reset-Authorization-User`; `SESSION_AUTHORIZATION` capability not advertised (consistent, so `SET SESSION AUTHORIZATION` fails cleanly). | client fork header list | Small |
| 3.7 knobs | Absent: `IgnoreBrokenCatalog`, `AllowMetadataFromMultipleCatalogs`, `DatabaseAsSchema`, `MuteServerWarnings`, `AssumeLiteralUnderscoreInMetadataCalls`, `ValidateConnection`, `HostnameInCertificate`, `TypeNameParameters`, `CharEncoding`. Broken-catalog tolerance is whatever `system.jdbc.tables` does server-side. | grep | Small each |
| A-8 | mTLS from PEM only; no PKCS#12, no Windows certificate store (rustls cannot use CNG keys). | `ssl.rs:39` | Medium |
| T-1 | `TlsVerify=ca` replaces the trust store with the supplied PEM (rustls constraint), so it cannot be combined with platform roots. | `builder.rs:420-445` | Accept or switch TLS stack |
| W-7, W-10 | No Tableau `.tdc`; linked servers untested (and blocked by F-12). | — | Small after F-12 |
| Governance | Vendor org, two repos plus a forked client at a pinned tag; core not yet on crates.io. Bus factor one organisation. | `Cargo.toml` TODOs | — |

### 9.3 What that means for the decision

The core ODBC machinery (§3.4) is complete to a degree neither trinodb/trino-odbc nor a
three-month new build would reach, and it is the part that is hardest to get right. The
gaps cluster in four places, all of them additive: Windows-native auth (Kerberos, client
credentials), prepare-time metadata (`DESCRIBE OUTPUT`), the compatibility knobs BI and
Office tools need (`DefaultVarcharLength`, `CP_ACP`, 32-bit build), and Windows
packaging (MSI, signing, native DSN dialog). The connector has one real bug (`LIMIT`/
`OFFSET` order) and lacks the credential kinds the Service and Gateway need.

Estimated to bring v0.1.2 to the §7 v1 cut: Kerberos/SSPI and client credentials in the
client fork and driver (the largest item), `DESCRIBE OUTPUT`, `DefaultVarcharLength`,
`SQL_ATTR_ANSI_APP`/`CP_ACP`, 429 retry, timeout split, 32-bit CI target, MSI with
signing, connector fixes (limit kind, `Key` credential). Everything else in 9.2 is a
later release. Before committing, run the §5 application smoke tests against the
unmodified v0.1.2 so the failures listed here are observed, not inferred from source.
