# Ghidra 12.2 change reference

What REA users and bridge-source maintainers need to know when moving from the
previously verified Ghidra 12.1.4 (JDK 21) to Ghidra 12.2 (JDK 25). Sources:
`docs/WhatsNew.md` and `docs/ChangeHistory.md` of the Ghidra 12.2 release.

REA admission accepts Ghidra 12.x (>=12.1). Ghidra 12.2 declares
`application.java.min=25`, so the full JDK must be 25 or newer; the bridge is
verified with Ghidra 12.2 and JDK 25. See
[installation](../getting-started/installation.md#ghidra).

## JDK 25 requirement

Ghidra 12.2 requires JDK 25 or later to run (GP-6012); 12.1.4 required JDK 21.
Ghidra 12.2 also runs Java with Compact Object Headers to reduce memory usage
(GP-7037). REA's admission reads the installed `application.java.min`, so a
12.2 installation rejects a JDK 21 host during `rea doctor` with the same
rule that accepted it for 12.1.4. No REA configuration change is needed; point
`JAVA_HOME` at a full JDK 25 build.

## `ghidraRun` project and file arguments

- Function: the `ghidraRun`/`ghidraRun.bat` launch argument can now open a
  project and one of its files with the default tool in a single start.
- Inputs: a combined path argument
  `/path/to/project.gpr:/project-folders/program.exe` (project path, colon,
  in-project content path).
- Outputs: the GUI opens the project and the named program in the default
  tool.
- Invocation: `./ghidraRun /path/to/project.gpr:/project-folders/program.exe`
  (GP-6543).
- Use case: skipping the project dialog when reopening a known target.
- Difference vs 12.1.4: the argument previously opened only a project.
- Compatibility impact: none for REA; REA drives `support/analyzeHeadless`
  and never launches the GUI. Listed here because headless users often also
  run the GUI on the same installation.

## `support/updateServerAllowList`

- Function: view and update the client-side Server Allow List that now gates
  `ghidra://` server connections and http/https links clicked in listing
  comment annotations. Explicit repository access via a shared project adds
  the server implicitly (GP-6796).
- Inputs: subcommands operating on the per-user allow-list store.
- Outputs: the current list, or a mutated list after add/remove.
- Invocation: `support/updateServerAllowList` (see `analyzeHeadlessREADME.md`
  in the installation for the headless workflow).
- Use case: headless processes that open `ghidra://` URLs must pre-approve the
  server, since there is no interactive prompt to accept an unknown server.
- Difference vs 12.1.4: the allow-list mechanism did not exist; any URL was
  followed.
- Compatibility impact: REA never connects to a Ghidra Server and only
  analyzes local files, so no REA workflow is affected. Custom headless
  scripts that use `ghidra://` URLs must run this utility once per server.

## `server/certTool` and enforced TLS server authentication

- Function: assists keystore request and certificate generation for Ghidra
  Server and BSim PostgreSQL deployments. Ghidra clients now enforce server
  authentication on every SSL/TLS connection, including loopback; deployments
  without an explicit `ghidra.keystore` bind to the loopback interface only
  (GP-6829).
- Inputs: keystore/certificate request parameters (see
  `server/svrREADME.md`).
- Outputs: keystore files or certificate requests suitable for a CA.
- Invocation: `server/certTool` (plus `server/certTool.bat` on Windows).
- Use case: any multi-user Ghidra Server or BSim PostgreSQL setup that needs
  remote TLS connections under 12.2.
- Difference vs 12.1.4: clients did not enforce server authentication;
  `certTool` did not exist.
- Compatibility impact: none for REA (local-only, no server connections).
  Local Ghidra Server workflows that previously relied on unauthenticated TLS
  must generate or import certificates or stay on loopback.

## `DebuggerStaticMappingContext`

- Function: static-to-debugger mapping logic factored out of the GUI static
  mapping service so mapper queries work in headless scripts (GP-6680).
- Inputs: the static program (and optionally a trace) to map.
- Outputs: the resolved mappings between static memory and trace memory.
- Invocation: construct in Java/PyGhidra headless scripts; replaces direct use
  of the GUI-only service.
- Use case: headless scripts that need "which trace address corresponds to
  this static address" without the Debugger UI.
- Difference vs 12.1.4: only the GUI plugin exposed this logic.
- Compatibility impact: additive API; REA does not use the Debugger and its
  bridge sources are unaffected.

## Trace byte search (`TraceMemoryOperations`)

- Function: search for a byte pattern in a trace across a given address range
  and lifespan (GP-7170).
- Inputs: byte pattern, address range, lifespan/snapshot range.
- Outputs: matching addresses (with the span they were found in).
- Invocation: new `TraceMemoryOperations` API method, usable from Debugger
  scripts.
- Use case: locating runtime data in recorded traces without dumping memory
  snapshots.
- Difference vs 12.1.4: no trace-scoped byte search API existed.
- Compatibility impact: additive. Note the companion change (GP-6630): old
  traces no longer open under 12.2 (DBTrace schema re-indexing, no upgrade
  provided), so re-record traces produced by earlier releases.

## Sleigh `.sla` lock files

- Function: lock files now protect Sleigh `.sla` files while they are being
  (re)compiled, preventing multiple — typically headless — processes from
  corrupting each other's rewrites (GP-4045).
- Inputs/outputs: none from the caller; automatic when a language is first
  compiled.
- Invocation: automatic.
- Use case: parallel headless runs sharing one installation, exactly the
  pattern REA's MCP server and CLI lanes use.
- Difference vs 12.1.4: concurrent first-use compilation of the same language
  could race.
- Compatibility impact: improvement only; no REA change required. Stale lock
  files after a crashed process are re-acquirable.

## Archive format support reduction

- Function: the sevenzipjbinding native library was removed; supported
  archive formats are now handled by Apache commons-compress (GP-7095).
- Difference vs 12.1.4: `zip`, `7z`, `bzip2` and `xz` still load.
  `cabarc` loads only when a `7z` command-line tool is on `PATH`.
  `hfsplus`, `iso9660`, `wim`, `arj`, `chm`, `deb`, `lha`, `rar`, `rpm`,
  `vhd` and `z` are no longer supported at all.
- Compatibility impact: this narrows REA's firmware/container loading
  boundary. Binaries wrapped in the dropped formats must be extracted with
  external tools before `open_binary`; Ghidra 12.2 reports them as unknown or
  unsupported containers rather than loading them. Standalone `zip`-style
  archives and Ghidra's own project formats are unaffected.

## Removal of APIs deprecated since 10.x

- Function: classes and methods deprecated since Ghidra 10.x and before were
  removed (GP-7104).
- Difference vs 12.1.4: code compiling against those symbols fails on 12.2.
- Compatibility impact: REA's bridge sources do not use the removed symbols
  and compile against 12.2 (verified by `verify:ghidra:bridge-sources`).
  Related 12.2 deprecations that still compile:
  - `PrototypeModel.getArgLocation`, `getReturnLocation`,
    `getStorageLocations(Program, Datatype[], boolean)` and
    `getNextArgLocation` are deprecated; use
    `getStorageLocations(Program, Datatype[], boolean, boolean)` instead
    (GP-6776). REA's calling-convention bridge sources still call the old
    signature, which remains present in 12.2.
  - `Accumulator` is write-only now; `size()` was replaced with
    `getProgress()` (GP-6539). Not used by REA.
  - `Option` constructors used by Loaders/Exporters are deprecated in favor
    of typed builder options (GP-6483). Not used by REA.

## Server/client compatibility notes

The Ghidra 12.2 server accepts clients back to 11.3.2, with the caveat that
newer link-files may not be handled by clients before 12.0. Ghidra 12.1
clients require Ghidra Server 12.1/12.0.5 or newer. Older installations are
encouraged to upgrade for the security fixes above. None of this affects
REA's local-only workflows.
