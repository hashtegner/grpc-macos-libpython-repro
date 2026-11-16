# grpc-macos-libpython-repro

Minimal reproduction: on macOS, `import grpc` segfaults when grpcio 1.80.0 is
built from source with Bazel against the hermetic rules_python interpreter.

```sh
bazelisk run //:import_grpc
```

- macOS arm64: segfault (exit 139) while loading `cygrpc.so` in `import grpc`.
- Linux: `grpc 1.80.0 imported OK`.

Versions: Bazel 9.1.0, grpc 1.80.0 (BCR), rules_python 2.4.6, Python 3.10.22
(python-build-standalone). With rules_python 1.6.3 (an older 3.10 build whose
`python3` links `libpython3.10.dylib` dynamically) the import works, since only
one libpython is loaded.

`patches/grpc_bazel9_address_sorting.patch` and the `--incompatible_autoload_externally`
flag are unrelated to the bug; they only make grpc 1.80.0 build on Bazel 9.

## Why

grpc 1.80 added `private_key_signer_py_wrapper`
(`src/python/grpcio/grpc/_cython/_cygrpc/private_key_signing/BUILD.bazel`),
which depends on `@rules_python//python/cc:current_py_cc_libs`. That links
`libpython3.X.dylib` into `cygrpc.so`. The rules_python interpreter has
libpython linked statically into the `python3` executable, so on macOS
(two-level namespace) `cygrpc.so` binds its `Py*` calls to a second,
uninitialized copy of libpython and crashes. On Linux the symbols resolve to
the running interpreter first, so the extra library is loaded but unused.

Workaround: [`patches/fix_cygrpc_macos_libpython.patch`](patches/fix_cygrpc_macos_libpython.patch)
drops `current_py_cc_libs` on macOS; `cygrpc.so` already links with
`-undefined dynamic_lookup` there. With it, the import works.

The [`repro` workflow](.github/workflows/repro.yml) runs it on `macos-15` and
`ubuntu-24.04`, with and without the workaround, and prints `otool -L` / `ldd` for `cygrpc.so` and `python3`.
