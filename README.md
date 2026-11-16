# grpc-macos-libpython-repro

Minimal reproduction: on macOS, `import grpc` segfaults when grpcio 1.80.0 is
built from source with Bazel against the hermetic rules_python interpreter.

```sh
bazelisk run //:import_grpc
```

- macOS arm64: `EXC_BAD_ACCESS` in `libpython3.10.dylib`'s `PyType_Ready` during `import grpc`.
- Linux: prints the grpc version.

## Why

grpc 1.80 added `private_key_signer_py_wrapper`
(`src/python/grpcio/grpc/_cython/_cygrpc/private_key_signing/BUILD.bazel`),
which depends on `@rules_python//python/cc:current_py_cc_libs`. That links
`libpython3.X.dylib` into `cygrpc.so`. The rules_python interpreter has
libpython linked statically into the `python3` executable, so on macOS
(two-level namespace) `cygrpc.so` binds its `Py*` calls to a second,
uninitialized copy of libpython and crashes. On Linux the symbols resolve to
the running interpreter first, so the extra library is loaded but unused.

Workaround: drop `current_py_cc_libs` on macOS; `cygrpc.so` already links with
`-undefined dynamic_lookup` there.

The [`repro` workflow](.github/workflows/repro.yml) runs it on `macos-15` and
`ubuntu-24.04` and prints `otool -L` / `ldd` for `cygrpc.so` and `python3`.
