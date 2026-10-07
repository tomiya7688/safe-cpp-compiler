# safe-cpp-compiler

A C/C++ compiler project focused on preventing optimization-driven semantic bugs and removing undefined behavior from the safe compilation path.

## Language baseline

- C: ISO/IEC 9899:2024 (C23)
- C++: ISO/IEC 14882:2024 (C++23)

## MVP1

```text
C23 / C++23
  -> Safety Analyzer / Rewriter
  -> LLVM IR
  -> LLVM Verifier
  -> Safe LLVM Validator
  -> Validated LLVM IR
```

Source diagnostics use simple `unsafety error` / `unsafety warning` categories.

## Specifications

Japanese documents are authoritative. English documents are translations.

| Topic | Japanese (authoritative) | English |
| --- | --- | --- |
| Overall specification | [docs/jp/SPEC.md](docs/jp/SPEC.md) | [docs/en/SPEC.md](docs/en/SPEC.md) |
| JSON policy | [docs/jp/CONFIG.md](docs/jp/CONFIG.md) | [docs/en/CONFIG.md](docs/en/CONFIG.md) |
| Build configuration | [docs/jp/BUILD.md](docs/jp/BUILD.md) | [docs/en/BUILD.md](docs/en/BUILD.md) |
| Unsafety rules | [docs/jp/RULES.md](docs/jp/RULES.md) | [docs/en/RULES.md](docs/en/RULES.md) |
| Safe LLVM IR / Validator | [docs/jp/IR.md](docs/jp/IR.md) | [docs/en/IR.md](docs/en/IR.md) |
| MVP1 / MVP2 | [docs/jp/MVP.md](docs/jp/MVP.md) | [docs/en/MVP.md](docs/en/MVP.md) |

Implementation tracking:
- [Issue #1: MVP1 compiler / validated LLVM IR](https://github.com/tomiya7688/safe-cpp-compiler/issues/1)
- [Issue #2: safe-build.json / CMake compatibility](https://github.com/tomiya7688/safe-cpp-compiler/issues/2)

## License

[MIT License](LICENSE)
