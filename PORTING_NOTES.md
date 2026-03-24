# Envoy v1.35.0 ppc64le Porting Notes

## Key differences from v1.34.0

### What upstream v1.35.0 already has (no patching needed)
1. **`rules_rust_ppc64le.patch`** — exists in the repo; `_rust_deps()` already applies it
2. **`v8_ppc64le.patch`** — exists in repo; `_v8()` already applies it
3. **`linux_ppc64le` config_setting** — uses correct `@platforms//cpu:ppc64le` (Bazel 7.6.0 supports this)
4. **`enable/disable_http3_on_linux_ppc64le` in `bazel/BUILD`** — already present

### What the patch fixes
| Area | Problem | Fix |
|------|---------|-----|
| **aws-lc** | No ppc64le support; breaks builds | Remove aws-lc entirely; simplify BoringSSL aliases |
| **BoringSSL** | Missing `_ARCH_PPC64` target detection | `boringssl_ppc64le.patch` adds `OPENSSL_64_BIT`/`OPENSSL_PPC64` |
| **Python** | 3.12 prebuilt binary not on ppc64le | Downgrade to 3.11.9 |
| **rules_go** | 0.53.0 prebuilt binary not on ppc64le | Downgrade to 0.46.0; add `rules_go.patch` |
| **rules_buf** | Doesn't recognize ppc64le CPU | `rules_buf.patch` adds `ppc64le` to allowed CPUs |
| **Rust toolchain** | `rust_linux_powerpc64le` not registered | Add to `dependency_imports.bzl` |
| **Lua** | LuaJIT doesn't support ppc64le | Mark `lua_lib`/`wrappers_lib` incompatible |
| **Async files** | `unlink(fd)` fails on some ppc64le filesystems | Defer unlink to close time |
| **aws-lc headers** | `aws_lc_compat.h` included in TLS code | Remove includes |
| **Protobuf patch** | Uses wrong `ppc64le` platform constraint | Change to `@platforms//cpu:ppc` |

### Bazel BUILD changes (v1.34.0 vs v1.35.0)
- v1.34.0 patch removed `linux_ppc64le` config_setting entirely — **we keep it in v1.35.0**
- v1.34.0 patch removed `not_x86_ppc` group — **we keep it in v1.35.0**
- v1.34.0 patch removed exe/BUILD ppc64le conditions — **we keep them in v1.35.0**
- We only remove the aws-lc routing groups (`boringssl_fips_ppc`, `boringssl_fips_not_ppc`, `not_ppc`)

### Lua incompatibility approach
In v1.35.0, `@platforms//cpu:ppc64le` is now correctly detected (Bazel 7.6.0).
The Lua `target_compatible_with` uses this directly, instead of the `@platforms//cpu:ppc` workaround from v1.34.0.
If on your test machine the constraint maps differently, change to:
```python
target_compatible_with = select({
    "@platforms//cpu:ppc": ["@platforms//:incompatible"],
    "//conditions:default": [],
}),
```

## Applying the patch

```bash
cd /path/to/envoy
git checkout v1.35.0
git apply /path/to/envoy_1.35.0.patch
```

If any hunks fail, use:
```bash
git apply --reject /path/to/envoy_1.35.0.patch
# then manually fix the .rej files
```

## Known potential issues

### 1. bazel/BUILD hunk offsets
The patch for `bazel/BUILD` is large. If the context doesn't match exactly (line offsets changed), apply with:
```bash
git apply --ignore-whitespace -C1 envoy_1.35.0.patch
```

### 2. envoy_library.bzl context
The function bodies of `envoy_cc_library` may have different internal content in v1.35.0. If the `native.cc_library` hunk fails, manually add `target_compatible_with = target_compatible_with,` after `local_defines = local_defines,` in each `native.cc_library(...)` call, and add `sanitizer_deps()` reference check.

### 3. async_file_manager_thread_pool.cc
The patch comments out a block with `/*` and `*/`. If `mkstemp`-based fallback changed in v1.35.0, the hunk might need manual adjustment. The key change: defer unlink from open-time to close-time.

### 4. cargo-bazel for rules_rust 0.56.0
The build script checks out rules_rust at `0.56.0` to build cargo-bazel natively for ppc64le.
If the cross-compilation fails, try a newer version of `cross`:
```bash
cargo install cross --version 0.2.5
```

### 5. rules_go.patch context
`rules_go.patch` is applied to the downloaded rules_go 0.46.0 archive. The file paths in the patch (`go/private/platforms.bzl`, `go/private/rules/binary.bzl`) must match the archive layout. If they don't, update the paths in `bazel/rules_go.patch`.

### 6. boringssl_ppc64le.patch context
The patch targets `include/openssl/target.h`. If the BoringSSL version `0.20250514.0` has a slightly different `target.h`, the context lines may not match. Verify with:
```bash
grep -n "myriad2\|OPENSSL_32_BIT\|OPENSSL_64_BIT" \
  $(bazel info output_base)/external/boringssl/include/openssl/target.h
```
And manually insert the three `#elif defined(_ARCH_PPC64)` lines if the patch fails.

## Build command
```bash
bazel build -c opt --config=libc++ envoy --config=clang \
  --define=wasm=disabled --cxxopt=-fpermissive
```
