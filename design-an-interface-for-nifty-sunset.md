# Path Interface Design for Spack

## Context

Spack's internal path handling is inconsistent. Paths arrive as `str` from argparse and YAML, are
processed through `canonicalize_path()` and stored in class attributes, then used in filesystem
operations, subprocess calls, and YAML serialization — often without clear separation between these
roles.

The goal is a consistent model where path types communicate intent:

- `pathlib.Path` — a path on the current local filesystem (concrete: can `.exists()`, `.open()`, etc.)
- `PurePosixPath` — a portable abstract path stored with forward slashes, not yet tied to the local
  filesystem (cross-platform specs, config values, build-cache entries)
- `Prefix` (`str` subclass) — unchanged; the package API surface that authors write against

The **structural interface is the typed class attribute**. Key Spack data structures declare
`pathlib.Path` (or `PurePosixPath`) fields; `__init__` converts `str` at construction time,
once per object. Internal code that reads the attribute gets a `Path` — mypy enforces that
no raw `str` can flow through without an explicit conversion at a defined boundary. This is already
the pattern in `database.py` (`self.database_directory = pathlib.Path(root_dir) / _DB_DIRNAME`);
the task is to apply it consistently.

No wrapper functions. Use `pathlib.Path(p)`, `str(p)`, and `p.as_posix()` directly.

---

## Type Alias (`lib/spack/spack/util/path.py`)

```python
StrPath = Union[str, "os.PathLike[str]"]
```

Used only on `__init__` parameter annotations at Spack's external-facing classes (CLI consumers,
config readers, package API). Internal functions never accept `str` for paths — they accept the
class's already-converted attribute type. Export from `spack.util.path.__all__`; all modules import
it from there.

---

## Note on Windows / `PurePosixPath`

`PurePosixPath` cannot represent Windows drive paths as absolute paths:
`PurePosixPath("C:/spack").is_absolute()` is `False`. For concrete local paths on Windows, use
`pathlib.Path` (= `WindowsPath`). For abstract paths that must be written to cross-platform YAML
(build-cache manifests, environment files), use `PurePosixPath` but only for paths whose separator
handling doesn't depend on `is_absolute()` — e.g., relative paths within a prefix, or paths that
were already POSIX-normalized before storage.

**Rule of thumb:** if the path lives on the local machine and may be absolute with a Windows drive
letter, use `pathlib.Path`. If the path is a portable string value in a YAML document (relative or
always-POSIX), use `PurePosixPath`. At YAML serialization, call `.as_posix()` on either type.

---

## The Boundary Pattern (applied to each class)

```python
# Entry: str from YAML/CLI → pathlib.Path once, in __init__
class Repository:
    root: pathlib.Path

    def __init__(self, root: StrPath, ...):
        self.root = pathlib.Path(canonicalize_path(root))  # concrete; on current fs
        ...

# Internal code reads self.root as pathlib.Path — no conversion needed
# mypy rejects self.root = "some/string"
# mypy rejects open(self.root)  ← actually open() accepts PathLike so this works
# mypy rejects subprocess_call([self.root])  ← catches missing str() conversion

# Exit: explicit serialization at output boundary
yaml_value = self.root.as_posix()   # for YAML (portable)
subprocess_arg = str(self.root)      # for subprocess (native separators)
```

This is identical to the pattern `database.py` already uses and already has test coverage.

---

## Affected Data Structures (Priority Order)

Apply the pattern to each class's `__init__`, then update internal usages to drop `os.path.*` calls
in favor of `pathlib.Path` operators (use `/` instead of `os.path.join`, `.exists()` instead of
`os.path.exists()`, etc.):

### Phase 1: Config-rooted path chain
These are the most-called paths; test coverage is dense.

| Class / attribute | File | Type | Note |
|---|---|---|---|
| `ConfigScope.path` | `config.py:224` | `pathlib.Path` | Local filesystem; already passed to `open()` |
| `Repository.root` | `repo.py:1124` | `pathlib.Path` | ~40 callers of `canonicalize_path` feed this |
| `Store.root` / `Store.unpadded_root` | `store.py:79,88` | `pathlib.Path` | Root of install tree |
| `Environment.root` / `manifest_dir` | `environment.py:701,720` | `pathlib.Path` | Local environment |
| `BootstrapProvider.metadata_dir` | `bootstrap/core.py:96,602` | `pathlib.Path` | Bootstrap store |

`canonicalize_path()` keeps returning `str` (URL compat), but callers wrap with `pathlib.Path()`
at assignment. All tests that install, activate environments, or load config will exercise this.

### Phase 2: Stage and build paths
| Class / attribute | File | Type | Note |
|---|---|---|---|
| `Stage.path` | `stage.py:249` | `pathlib.Path` | Stage scratch dir |
| `Stage.source_path` | `stage.py:540` | `pathlib.Path` | Extracted source |
| `Stage.archive_file` | `stage.py:526` | `Optional[pathlib.Path]` | Downloaded archive |
| `BaseBuilder.build_directory` | `build_systems/cmake.py:85`, `autotools.py:85` | `pathlib.Path` | Build dir |
| `BaseBuilder.configure_directory` | `build_systems/autotools.py:74` | `pathlib.Path` | Configure dir |

Existing `spack install` tests on any package exercise this end-to-end.

### Phase 3: Package metadata paths
| Property | File | Type |
|---|---|---|
| `PackageBase.log_path` | `package_base.py:1319` | `pathlib.Path` |
| `PackageBase.env_path` | `package_base.py:1288` | `pathlib.Path` |
| `PackageBase.metadata_dir` | `package_base.py:1301` | `pathlib.Path` |
| `PackageBase.configure_args_path` | `package_base.py:1344` | `pathlib.Path` |
| `PackageBase.times_log_path` | `package_base.py:1349` | `pathlib.Path` |
| `PackageBase.install_log_path` | `package_base.py:1332` | `pathlib.Path` |

These all derive from `self.stage.path` (already `pathlib.Path` after Phase 2) via `/` operator.

### Phase 4: Subprocess output and `filesystem.py` public API
- `Executable.__call__(*args: StrPath)` — convert `PathLike` args to `str(a)` before `Popen`
- `filesystem.py` public functions — change `str` params to `StrPath`; call `pathlib.Path(p)` at
  top of each function. The ~45 functions decorated with `@system_path_filter` should also accept
  `StrPath` (the decorator already handles the conversion for Windows separators).

### Phase 5: `spack.paths` constants (deferred)
- Convert `spack.paths.prefix`, `lib_path`, etc. from `str` to `pathlib.Path`.
- Most breaking for extensions; audit external consumers before starting.

---

## `Prefix` (no change needed; `__truediv__` optional)

`Prefix` is a `str` subclass, so `os.fspath(prefix)` and `pathlib.Path(prefix)` already work —
`os.fspath` returns its argument unchanged for `str` instances. No `__fspath__` needed.

`__truediv__` enabling `prefix / "bin"` syntax is a small quality-of-life addition (returns a
`Prefix`) that package authors can use immediately, but it is not part of the boundary design.
Include it in Phase 1 as a low-risk addition.

Phase methods continue to receive `prefix` as `Prefix`. Internal Spack code that needs a
`pathlib.Path` for the prefix calls `pathlib.Path(spec.prefix)` locally.

---

## Output Boundary Summary

| Destination | Conversion | Location |
|---|---|---|
| `subprocess.Popen` args | `str(p)` | `util/executable.py` (Phase 4) |
| YAML / config files | `p.as_posix()` | emit sites in `config.py`, `environment.py`, `binary_distribution.py` |
| CI pipeline YAML | `p.as_posix()` | `ci/gitlab.py` (already done; standardize) |
| User-facing `print` / `tty.*` | `str(p)` | `cmd/location.py` and other command outputs |
| Module files | `p.as_posix()` | `modules/*.py` |
| Environment variables for build | `str(p)` | `build_environment.py:PrependPath` etc. |

---

## Verification

1. `pytest lib/spack/spack/test/` — full test suite passes after each phase
2. `./bin/spack style` — mypy passes; no `Union[str, pathlib.Path]` written inline; `StrPath`
   imported from `spack.util.path`
3. `spack install zlib` on Linux, macOS, Windows — installs with no path regressions
4. `spack location -p zlib` on Windows — prints native-separator path
5. On Windows, build-cache manifest entries use forward slashes in YAML
6. New tests per phase: pass `pathlib.Path` objects where `StrPath` is accepted; verify typed
   attributes hold `pathlib.Path` after construction
