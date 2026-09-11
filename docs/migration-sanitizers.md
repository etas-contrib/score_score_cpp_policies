# Migration: from the v0.x sanitizer API to the policy-owned API

See [`sanitizers/README.md`](../sanitizers/README.md) for the full feature,
constraint, suppression, and runtime reference. This guide covers the API
migration itself.

## Scope and version assumptions

The current sanitizer API is owned by `score_cpp_policies`. It uses the
`extra_known_features` toolchain registration API and the policy's
`rules_cc` feature definitions. Use a C++ toolchain that supports
`extra_known_features` and the current feature API. This repository does not
claim a minimum toolchain version for that support.

The repository tests currently use these versions:

- `score_bazel_cpp_toolchains` 1.0.2
- `rules_cc` 0.2.17
- `toolchains_llvm` 1.7.0
- GCC 12.2.0 and 15.3.0
- Clang 19.1.7

These are repository test versions, not minimum version requirements.

## Old model

The v0.x API exposed one string flag:

```text
--@score_cpp_policies//sanitizers/flags:sanitizer=<value>
```

The recognised legacy values were `asan_ubsan_lsan` and `tsan`. The old model
also used aggregate GCC feature names such as `asan_ubsan_lsan_gcc` and
`tsan_gcc`.

The current `sanitizers/flags:sanitizer` target remains only as a deprecated
compatibility shim so Bazel can parse those legacy values. Non-default use is
rejected by `sanitizer_deprecated_check`; it is not a migration path.

| Legacy invocation | Current invocation |
| --- | --- |
| `--@score_cpp_policies//sanitizers/flags:sanitizer=asan_ubsan_lsan` | `--config=asan_ubsan_lsan` |
| `--@score_cpp_policies//sanitizers/flags:sanitizer=tsan` | `--config=tsan` |

## New model

The policy defines one bool flag per sanitizer and exposes these config
aliases through [`sanitizers.bazelrc`](../sanitizers/sanitizers.bazelrc):

| Sanitizer | Current config |
| --- | --- |
| AddressSanitizer | `--config=asan` |
| UndefinedBehaviorSanitizer | `--config=ubsan` |
| LeakSanitizer | `--config=lsan` |
| ThreadSanitizer | `--config=tsan` |
| ASan + UBSan + LSan | `--config=asan_ubsan_lsan` |
| TSan + UBSan | `--config=tsan_ubsan` |

The single-sanitizer configs activate exactly their named sanitizer. Composite
configs are convenience aliases that compose those single-sanitizer configs.
The aliases also configure the policy's test wrapper and debug-symbol setup.

### Feature registration

Register the policy labels in the toolchain's `extra_known_features`. Register
the compiler-appropriate UBSan variant:

| Purpose | Label | Feature name | Toolchain |
| --- | --- | --- | --- |
| Debug symbols | `@score_cpp_policies//sanitizers/features:debug_symbols` | `debug_symbols` | GCC and Clang |
| AddressSanitizer | `@score_cpp_policies//sanitizers/features:asan` | `score_asan` | GCC and Clang |
| UBSan common flags | `@score_cpp_policies//sanitizers/features:ubsan_base` | `score_ubsan_base` | GCC and Clang |
| UBSan GCC variant | `@score_cpp_policies//sanitizers/features:ubsan_gcc` | `score_ubsan_gcc` | GCC |
| UBSan Clang variant | `@score_cpp_policies//sanitizers/features:ubsan_clang` | `score_ubsan_clang` | Clang |
| LeakSanitizer | `@score_cpp_policies//sanitizers/features:lsan` | `score_lsan` | GCC and Clang |
| ThreadSanitizer | `@score_cpp_policies//sanitizers/features:tsan` | `score_tsan` | GCC and Clang |

The former aggregate GCC features `asan_ubsan_lsan_gcc` and `tsan_gcc` are
removed. Replace them with the per-sanitizer labels above. In particular,
register `asan`, `lsan`, and `tsan` directly, and register `ubsan_gcc` for GCC
or `ubsan_clang` for Clang. Do not register both UBSan variants for one
toolchain.

Example GCC registration:

```starlark
gcc.toolchain(
    ...,
    extra_known_features = [
        "@score_cpp_policies//sanitizers/features:debug_symbols",
        "@score_cpp_policies//sanitizers/features:asan",
        "@score_cpp_policies//sanitizers/features:ubsan_base",
        "@score_cpp_policies//sanitizers/features:ubsan_gcc",
        "@score_cpp_policies//sanitizers/features:lsan",
        "@score_cpp_policies//sanitizers/features:tsan",
    ],
)
```

Example Clang registration:

```starlark
llvm.toolchain(
    llvm_version = "...",
    extra_known_features = [
        "@score_cpp_policies//sanitizers/features:debug_symbols",
        "@score_cpp_policies//sanitizers/features:asan",
        "@score_cpp_policies//sanitizers/features:ubsan_base",
        "@score_cpp_policies//sanitizers/features:ubsan_clang",
        "@score_cpp_policies//sanitizers/features:lsan",
        "@score_cpp_policies//sanitizers/features:tsan",
    ],
)
```

### Dependency and Bazel configuration

Add `score_cpp_policies` to the consuming workspace:

```starlark
bazel_dep(name = "score_cpp_policies")
```

Import the policy config file from the workspace `.bazelrc`:

```text
try-import %workspace%/path/to/score_cpp_policies/sanitizers/sanitizers.bazelrc
```

Use the import path appropriate for how the dependency is available in the
workspace. The config file is the source of the `--config=` aliases.

### Valid and invalid combinations

Supported combinations are:

| Combination | Current invocation |
| --- | --- |
| ASan | `--config=asan` |
| UBSan | `--config=ubsan` |
| LSan | `--config=lsan` |
| TSan | `--config=tsan` |
| ASan + UBSan + LSan | `--config=asan_ubsan_lsan` |
| TSan + UBSan | `--config=tsan_ubsan` |
| ASan + LSan | `--config=asan --config=lsan` |

ASan + TSan and LSan + TSan are invalid because their runtime libraries are
incompatible; TSan also provides its own leak detection. The feature layer
declares these pairs mutually exclusive, and the flag layer provides a
secondary combination check. Do not create configs for these pairs.

### Constraints and duplicate flags

In the current API, `no_asan_ubsan_lsan` means no ASan, UBSan, or LSan is
active: it matches when any one of those three flags is set. This differs from
the old single-flag behavior, where it was satisfied unless the combined
`asan_ubsan_lsan` preset was active. For new targets, prefer the granular
`no_asan`, `no_ubsan`, or `no_lsan` constraints. If a target must be skipped
only when all three are active, use the `asan_ubsan_lsan` match-all flag group
in a custom `config_setting`.

Remove duplicate custom sanitizer flags from `copts`, custom features, or
toolchain forks. The policy features already provide the compile and link
flags, while the config aliases select them. Duplicates can conflict with the
policy's feature composition and runtime checks.

## Migration procedure

1. Add `score_cpp_policies` as a `bazel_dep`.
2. Import `sanitizers/sanitizers.bazelrc` from the consuming workspace's
   `.bazelrc`.
3. Replace legacy string-flag use with the equivalent `--config=` alias:
   `asan_ubsan_lsan` maps to `--config=asan_ubsan_lsan`, and `tsan` maps to
   `--config=tsan`.
4. Replace registrations of `asan_ubsan_lsan_gcc` and `tsan_gcc` with the
   per-sanitizer labels, including the matching `ubsan_gcc` or `ubsan_clang`.
5. Remove duplicate sanitizer flags from project options and custom features.
6. Review `no_asan_ubsan_lsan` uses and replace them with granular constraints
   where that expresses the intended compatibility rule.
7. Choose a supported config, run the validation commands below, and fix any
   sanitizer or runtime issues exposed by the new policy.

## Validation

This repository does not provide a dedicated Markdown-link checker. Use the
Markdown diagnostics available in the editor or the documentation tooling used
by the consuming workspace to verify relative links.

From `tests/`, run the sanitizer suites:

```bash
bazel test --config=asan_ubsan_lsan //...
bazel test --config=tsan //...
```

Also validate a Clang UBSan configuration when using the Clang toolchain:

```bash
bazel test --config=ubsan //...
```
