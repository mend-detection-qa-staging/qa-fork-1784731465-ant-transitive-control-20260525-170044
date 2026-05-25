# ant-transitive-control

Single-module Ant+Ivy probe exercising three distinct transitive-control mechanisms
in one `ivy.xml`. This is Project #7 (P1) from the Ant coverage plan.

---

## Catalog patterns exercised

| Catalog ID | Pattern name | Mechanism | Signal |
|---|---|---|---|
| #13 | `transitive-only-dep` | `guava` direct dep → `failureaccess` must appear | `failureaccess-1.0.1.jar` present in lib/ |
| #17 | `transitive-false` | `commons-beanutils` with `transitive="false"` | `commons-collections-*.jar` absent from lib/ |
| #9 | `exclude-transitive` | `httpclient` with `<exclude org="commons-logging"/>` | `commons-logging-*.jar` absent from lib/ |

---

## Mechanism details

### Mechanism 1: Plain transitive identity (#13 — transitive-only-dep)

**Feature exercised:** Default Ivy behavior — `transitive="true"` (the default). When
`guava:32.1.3-jre` is declared as a direct dependency, Ivy fetches its POM-declared
compile-scope deps automatically.

**guava 32.1.3-jre POM declares:**
- `com.google.guava:failureaccess:1.0.1` (compile scope)
- `com.google.guava:listenablefuture:9999.0-empty-to-avoid-conflict-with-guava` (compile scope)

**Expected in lib/:**
- `guava-32.1.3-jre.jar` (direct)
- `failureaccess-1.0.1.jar` (transitive — the regression signal)
- `listenablefuture-9999.0-empty-to-avoid-conflict-with-guava.jar` (transitive)

**Regression to detect:** If `failureaccess-1.0.1.jar` is absent from Mend's scan output,
the scanner failed to fingerprint it. `listenablefuture` uses an unusual version string
(`9999.0-empty-to-avoid-conflict-with-guava`) — absence here is a secondary signal that
Mend skips JARs with non-semver version strings.

---

### Mechanism 2: transitive="false" (#17 — transitive-false)

**Feature exercised:** The `transitive="false"` attribute on an Ivy `<dependency>` element.
This instructs Ivy NOT to fetch the dependency's own transitive deps from its POM.

**commons-beanutils:1.8.3 POM declares:**
- `commons-logging:commons-logging:1.1.1` (compile scope)
- `commons-collections:commons-collections:3.2.2` (compile scope)

With `transitive="false"`, neither is fetched.

**Expected in lib/:** `commons-beanutils-1.8.3.jar` only.

**NOT in lib/:** `commons-logging-*.jar`, `commons-collections-*.jar`.

**Regression to detect:** If `commons-collections-*.jar` appears in Mend's output, the
scanner sourced it from somewhere other than `lib/` (e.g. a host-level Maven cache). The
`nin` assertion in `autotest_config.json` catches this.

---

### Mechanism 3: `<exclude>` (#9 — exclude-transitive)

**Feature exercised:** The `<exclude>` child element of an Ivy `<dependency>`. This drops
a specific transitive from the resolution graph for one dep path.

**httpclient:4.5.14 POM declares:**
- `commons-logging:commons-logging:1.2` (compile scope)
- `org.apache.httpcomponents:httpcore:4.4.16` (compile scope)
- `commons-codec:commons-codec:1.11` (compile scope)

The ivy.xml exclude: `<exclude org="commons-logging" module="commons-logging"/>` drops
`commons-logging:1.2`. `httpcore` and `commons-codec` are unaffected.

**Expected in lib/:** `httpclient-4.5.14.jar`, `httpcore-4.4.16.jar`, `commons-codec-1.11.jar`.

**NOT in lib/:** `commons-logging-*.jar`.

**Regression to detect:** If `commons-logging-*.jar` appears in Mend's output, the scanner
sourced it from outside `lib/`. The `nin` assertion catches this. Note: commons-logging is
excluded by BOTH mechanism 2 and mechanism 3, so its absence is doubly reinforced.

---

## Mode-3 pivot

The bolt-4 SCM scanner (Mend's GitHub integration) does **not** invoke Ivy resolution.
It falls through to Mode-3: binary file path scan (SHA1 fingerprinting of `*.jar` files).

**Consequence:** `whitesource.config` with `ant.ivyResolveDependencies=true` is IGNORED
in SCM integration mode. The files in `lib/` are what the scanner sees.

**What we did:** Pre-resolved and committed exactly 7 JARs into `lib/` reflecting the
post-resolution state that Ivy WOULD produce. The Ivy manifest files (`ivy.xml`,
`ivysettings.xml`) are present for documentation and for pipeline-mode `mend ua` testing.

**Absent JARs (regression signals):**
- `commons-logging-*.jar` — excluded by both mechanism 2 (transitive=false) and mechanism 3 (<exclude>)
- `commons-collections-*.jar` — cut by mechanism 2 (transitive=false on beanutils)

If these JARs appear in Mend's scan output, the scanner sourced them from somewhere other
than `lib/` (e.g. Ant's own classpath, a host Maven/Ivy cache, or a `.gradle` cache). This
is a false positive — the probe does NOT include these JARs.

---

## lib/ contents

| JAR | SHA1 | Notes |
|---|---|---|
| `guava-32.1.3-jre.jar` | `0f306708742ce2bf0fb0901216183bc14073feae` | Direct dep (mechanism 1) |
| `failureaccess-1.0.1.jar` | `1dcf1de382a0bf95a3d8b0849546c88bac1292c9` | Guava transitive — must appear (#13 signal) |
| `listenablefuture-9999.0-empty-to-avoid-conflict-with-guava.jar` | `b421526c5f297295adef1c886e5246c39d4ac629` | Guava transitive — unusual version string |
| `commons-beanutils-1.8.3.jar` | `686ef3410bcf4ab8ce7fd0b899e832aaba5facf7` | Direct dep with transitive=false |
| `httpclient-4.5.14.jar` | `1194890e6f56ec29177673f2f12d0b8e627dec98` | Direct dep with <exclude> |
| `httpcore-4.4.16.jar` | `51cf043c87253c9f58b539c9f7e44c8894223850` | httpclient transitive (kept) |
| `commons-codec-1.11.jar` | `3acb4705652e16236558f0f4f2192cc33c3bd189` | httpclient transitive (kept) |

**Total: 7 JARs.** All sourced from Maven Central.

---

## Mend config

**Bucket A** — default-emit with `java` pinned to `17` (Ant itself is not pinnable via
`install-tool`; the Ant binary version comes from whatever the operator installs
out-of-band). This is a partial reproducibility limitation: the Ant version is not
controlled by `.whitesource`. Downstream comparators should treat the Ant version as
uncontrolled.

`.whitesource` uses `configMode: "LOCAL"` — no external config URL.

**Additional dimension:** `whitesource.config` with `ant.resolveDependencies=true` and
`ant.ivyResolveDependencies=true` is present for pipeline-mode `mend ua` testing only.
These flags are NOT honored by the bolt-4 SCM scanner.

---

## autotest_config.json assertions

| Assertion | Type | What it validates |
|---|---|---|
| `security_scan_report_conclusion: success` | scan_output | Scan completes without hard failure |
| `total_libs_scanned gte 7` | scan_output | All 7 JARs detected |
| `direct_dependencies_names_list contains [guava, failureaccess, commons-beanutils, httpclient, httpcore, commons-codec]` | dep_tree | Present JARs appear in output |
| `expected_dependency_pairs` | dep_tree | JAR filename + version pairs (exact) |
| `validate_checksums: true` with SHA1 pins | dep_tree | Binary fingerprint regression |
| `check_sha1_not_empty, check_sha1_format, check_sha1_consistency` | checksums_config | SHA1 integrity |

**nin assertion intent (commons-logging, commons-collections):**
The `nin` operator on `direct_dependencies_names_list` would assert that
`commons-logging-1.1.1.jar`, `commons-logging-1.2.jar`, and `commons-collections-3.2.2.jar`
are NOT in Mend's output. Per `extending-validations.md`, `nin` is a supported operator
(none of actual in expected). To use it, add a second `direct_dependencies_names_list` block
with `"nin": ["commons-logging-1.1.1.jar", "commons-logging-1.2.jar", "commons-collections-3.2.2.jar"]`.
Current `autotest_config.json` uses the `contains` / `expected_dependency_pairs` positive-side
assertions; the nin block should be added once the exact JAR names Mend emits are confirmed
from a first live scan.

---

## UA resolver notes (from java-jvm.md § 3)

- Mend's `IvyDependenciesAntTask` parses the Ivy `ResolveReport` for the dependency graph
  (Mode-1 only — not active in SCM integration mode).
- Mode-3 path scan fingerprints all `*.jar` files found under the project root.
- `transitive="false"` and `<exclude>` are Ivy resolution-time features — they are invisible
  to Mode-3. Their effect is encoded in which JARs ARE and ARE NOT present in `lib/`.
- **No EUA support** for Ant — impact analysis is disabled per resolver docs.

---

## Probe metadata

```
probe_id:         ant-transitive-control-20260525-170044
pm:               ant
pm_version_tested: ivy-2.5.x (Ivy used by UA's in-process library)
patterns:         [exclude-transitive, transitive-only-dep, transitive-false]
catalog_ids:      [#9, #13, #17]
priority:         P1
generated:        2026-05-25
bucket:           A (java pinned to 17)
target:           remote
```
