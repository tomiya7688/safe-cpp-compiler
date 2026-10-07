# Build Configuration Specification

> **English translation**
>
> The Japanese `docs/jp/BUILD.md` is authoritative.

## 1. Purpose

Safe C++ Compiler interoperates with CMake without reimplementing CMake's full build language.

Native projects use lightweight declarative JSON:

```text
safe-build.json
```

- `safe-cpp.json` — safety policy, unsafety diagnostics, LLVM validation
- `safe-build.json` — platform, sources, targets, output, linking, resource staging

## 2. Basic example

```json
{
  "version": 1,
  "project": {
    "name": "sample"
  },
  "platform": {
    "arch": "x64",
    "os": "windows",
    "abi": "msvc"
  },
  "build": {
    "profile": "debug",
    "optimization": "none",
    "debug_info": true,
    "output_directory": "build"
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
      "output_name": "sample",
      "links": {
        "libraries": ["user32"],
        "library_directories": ["vendor/lib"]
      },
      "application_root": "Application/root",
      "copy": [
        { "from": "assets", "to": "assets" },
        { "from": "config/default", "to": "config" }
      ]
    }
  }
}
```

## 3. Platform

Human-readable target selection:

```json
{
  "platform": {
    "arch": "x64",
    "os": "windows",
    "abi": "msvc"
  }
}
```

Initial architecture names: `x64`, `x86`, `arm64`, `arm32`.

Initial OS names: `windows`, `linux`, `macos`.

Representative ABI names: `msvc`, `gnu`, `musl`, `apple`.

The build system normalizes these to LLVM target information. Unsupported combinations fail clearly.

An advanced direct `triple` setting may be added later. Conflicting triple and arch/os/abi settings are errors.

## 4. Build settings

```json
{
  "build": {
    "profile": "debug",
    "optimization": "none",
    "debug_info": true,
    "output_directory": "build"
  }
}
```

Initial profiles: `debug`, `release`.

Profiles never weaken safety. Release builds must not disable required safety checks or Safe LLVM Validator.

Optimization values may include `none`, `basic`, `speed`, and `size`. MVP1 may implement only `none` and `basic` initially.

## 5. Inputs

`source_directories` defines source search roots.

`include_directories` defines C/C++ include search paths.

## 6. Targets

Initial target types:

- `application`
- `static_library`
- `shared_library`

MVP prioritizes `application`.

Targets may define sources, include directories, defines, output name, links, application root, and copy rules.

## 7. Sources

Target sources may use globs such as:

```json
{
  "sources": ["src/**", "lib/math/*.cpp"]
}
```

## 8. Defines

```json
{
  "defines": ["APP_VERSION=1", "FEATURE_X"]
}
```

Both name-only and `NAME=value` forms are supported.

## 9. Output name

`output_name` is the logical output name. Platform suffixes/prefixes are derived by the build system.

For MVP1 it also names validated LLVM outputs such as `build/sample.ll` and `build/sample.bc`.

## 10. Linking

```json
{
  "links": {
    "libraries": ["user32", "mylib"],
    "library_directories": ["vendor/lib"]
  }
}
```

MVP1 may initially parse and retain these settings even though machine-code linking is not a completion requirement.

## 11. Application root

`application_root` is the application staging/package root.

MVP may restrict it to project-relative paths.

## 12. Copy rules

Files or directories may be copied into the application root.

Absolute destinations, `..` escapes, unsafe symlink escapes, destination collisions, and missing sources are errors by default.

## 13. Runtime and toolchain

Platform-specific settings may be added incrementally, for example:

```json
{
  "toolchain": {
    "linker": "default",
    "runtime": "dynamic"
  }
}
```

Future settings may cover linker selection, static/dynamic runtime, sysroot, SDK path, Windows subsystem, and deployment target.

This must not become a general-purpose scripting language.

## 14. Build artifacts

MVP1 primarily stages validated LLVM IR. Future machine-code builds may stage executables and resources under the application root.

## 15. CMake compatibility

Do not reimplement the CMake language.

Use an adapter to import machine-readable CMake target/codemodel information. Native projects use `safe-build.json`.

CMake export may be added later.

## 16. Separation from safe-cpp.json

`safe-cpp.json`: safety diagnostics and LLVM validation.

`safe-build.json`: platform, build profile, source/include paths, targets, output, linking, staging/copy, and CMake integration.

## 17. Path normalization

JSON paths use `/` canonically. Paths are normalized and checked for root escape before file operations.

## 18. CLI overrides

Frequently changed values may be overridden without editing JSON, for example:

```text
safe-cpp build --arch x64 --os windows --profile release
```

CLI overrides take precedence over JSON.

## 19. MVP build scope

Initial implementation includes:

1. safe-build.json parser
2. platform arch/os/abi
3. profile/optimization/debug info/output directory
4. source/include directories
5. application target
6. source globs
7. defines
8. output name
9. link settings parsing
10. application root
11. file/directory copy
12. collision/root-escape/symlink validation
13. CMake import adapter

Package management, install rules, arbitrary custom commands, and a general scripting language may be deferred.
