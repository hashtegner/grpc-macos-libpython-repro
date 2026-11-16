# grpc-macos-libpython-repro

Minimal reproduction: on macOS, `import grpc` segfaults when grpcio is built
from source with Bazel against the hermetic rules_python interpreter.

```sh
bazelisk run //:import_grpc
```

- macOS arm64: segfault (exit 139) while loading `cygrpc.so` in `import grpc`.
- Linux: `grpc 1.85.1 imported OK`.

Versions: Bazel 9.3.2, grpc 1.85.1, rules_python 2.5.1, Python 3.10.22
(python-build-standalone).

## Why

`private_key_signer_py_wrapper`
(`src/python/grpcio/grpc/_cython/_cygrpc/private_key_signing/BUILD.bazel`)
depends on `@rules_python//python/cc:current_py_cc_libs`, which links
`libpython3.10.dylib` into `cygrpc.so`. The rules_python interpreter has
libpython linked statically into the `python3` executable and does not load
that dylib. On macOS (two-level namespace), `cygrpc.so` binds its `Py*` calls
to the copy in `libpython3.10.dylib`, which was never initialized, and the
first C API call crashes. On Linux the symbols resolve to the running
interpreter first, so the extra library is loaded but unused.

## Workaround

[`patches/fix_cygrpc_macos_libpython.patch`](patches/fix_cygrpc_macos_libpython.patch)
drops `current_py_cc_libs` on macOS. `cygrpc.so` already links with
`-undefined dynamic_lookup` there, so the `Py*` symbols resolve to the running
interpreter and the import works. The Linux build does not change.

## CI

The [`repro` workflow](.github/workflows/repro.yml) runs `//:import_grpc` on
`macos-15` and `ubuntu-24.04`, with and without the workaround, and prints
`otool -L` / `ldd` for `cygrpc.so` and `python3`. Only `macos-15` without the
workaround fails.

## Build note

`--features=-layering_check` in `.bazelrc` is not related to the bug. On macOS,
grpc's `layering_check` fails in `chttp2_transport.cc` because the generated
module map is missing a declared dependency (see
[bazelbuild/bazel#32247](https://github.com/bazelbuild/bazel/issues/32247)).
