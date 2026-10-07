# Build Configuration Specification

> **English translation**
>
> The Japanese `docs/jp/BUILD.md` is authoritative.

## 1. Purpose

Safe C++ Compiler should interoperate with CMake without reimplementing CMake's full build language.

Normal projects use a lightweight declarative JSON file:

```text
safe-build.json
```

`safe-cpp.json` contains safety policy. `safe-build.json` contains build/project configuration.

## 2. Basic example

```json
{
  "version": 1,
  "project": {
    "name": "sample"
  },
  "inputs": {
    "source_directories": ["src", "lib"],
    "include_directories": ["include", "third_party/example/include"]
  },
  "targets": {
    "app": {
      "type": "application",
      "sources": ["src/**", "lib/**"],
      "defines": ["APP_VERSION=1"],
      "application_root": "Application/root",
      "copy": [
        { "from": "assets", "to": "assets" },
        { "from": "config/default", "to": "config" }
      ]
    }
  }
}
```

## 3. Input directories

`source_directories` defines directories searched for source input.

`include_directories` adds C/C++ include search paths.

Paths are relative to the project root by default.

## 4. Targets

Initial target types:

- `application`
- `static_library`
- `shared_library`

MVP prioritizes `application`.

Targets may define sources, include directories, defines, links, application root, and copy rules.

## 5. Application root

`application_root` is the staging/package root for an application target.

Example:

```json
{
  "application_root": "Application/root"
}
```

MVP may restrict it to project-relative paths.

## 6. Copy rules

Files or directories may be copied into the application root.

```json
{
  "copy": [
    { "from": "assets", "to": "assets" },
    { "from": "config/default", "to": "config" }
  ]
}
```

Conceptually:

```text
assets/**          -> Application/root/assets/**
config/default/**  -> Application/root/config/**
```

`from` is project-root-relative.

`to` is application-root-relative.

Absolute destinations and `..` escapes outside the application root are errors.

Destination collisions are errors by default.

Symlinks that resolve outside the application root are rejected in MVP.

Missing sources are build errors.

## 7. Build artifacts

Future executable/resource artifacts may be staged into the application root.

MVP1 primarily produces validated LLVM IR, so minimal staging such as the following is sufficient:

```text
Application/root/
  app.ll
  assets/
  config/
```

## 8. CMake compatibility

Safe C++ Compiler does not reimplement the CMake language.

Compatibility means importing existing CMake project/target information through an adapter, while native Safe projects can use `safe-build.json` directly.

Initial direction:

- existing CMake project: import machine-readable target/codemodel information
- native Safe project: use `safe-build.json`
- exporting back to CMake may be added later

The compiler does not independently interpret all CMake commands, macros, or generator expressions.

## 9. Separation from safe-cpp.json

`safe-cpp.json` contains safety diagnostics and LLVM validation.

`safe-build.json` contains build inputs, targets, staging, copy rules, and CMake integration.

## 10. Path normalization

JSON paths use `/` as the canonical separator.

The build system normalizes paths and checks for root escape before file operations.

## 11. MVP build scope

Initial implementation:

1. safe-build.json parser
2. source_directories
3. include_directories
4. application target
5. source globs
6. defines
7. application_root
8. file/directory copy
9. collision/root-escape validation
10. CMake import adapter

Libraries, install rules, package managers, custom commands, and a scripting language may be deferred.
