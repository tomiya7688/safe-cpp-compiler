# Build Configuration 仕様

> **正本 (Authoritative Specification)**
>
> この文書は Safe C++ Compiler の build configuration の日本語正本である。
> `docs/en/BUILD.md` は翻訳版。

## 1. 目的

Safe C++ Compiler は CMake と連携できるが、CMake と同じ複雑な build language を再実装しない。

通常 project では軽量な宣言的 JSON:

```text
safe-build.json
```

を使用する。

- `safe-cpp.json` — 安全性、unsafety diagnostics、LLVM validation
- `safe-build.json` — platform、source、target、output、link、resource staging

安全 policy と build description を分離する。

## 2. 基本例

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
    "source_directories": [
      "src",
      "lib"
    ],
    "include_directories": [
      "include",
      "third_party/example/include"
    ]
  },

  "targets": {
    "app": {
      "type": "application",
      "sources": [
        "src/**",
        "lib/**"
      ],
      "defines": [
        "APP_VERSION=1"
      ],
      "output_name": "sample",
      "links": {
        "libraries": [
          "user32"
        ],
        "library_directories": [
          "vendor/lib"
        ]
      },
      "application_root": "Application/root",
      "copy": [
        {
          "from": "assets",
          "to": "assets"
        },
        {
          "from": "config/default",
          "to": "config"
        }
      ]
    }
  }
}
```

## 3. platform

compile target を人間が読みやすい形で指定する。

```json
{
  "platform": {
    "arch": "x64",
    "os": "windows",
    "abi": "msvc"
  }
}
```

### 3.1 arch

初期名称:

- `x64`
- `x86`
- `arm64`
- `arm32`

内部では必要に応じて LLVM の canonical architecture 名へ正規化する。

例:

```text
x64   -> x86_64
arm64 -> aarch64
```

設定形式として名称を定義することと、その target を現在の compiler build が実際にサポートすることは別である。
未対応 target は明確な build error とする。

### 3.2 os

初期名称:

- `windows`
- `linux`
- `macos`

将来必要に応じて追加する。

### 3.3 abi

任意指定。

例:

- `msvc`
- `gnu`
- `musl`
- `apple`

省略時は os/toolchain の既定値を使用できる。

### 3.4 LLVM target triple

通常利用者は arch/os/abi を指定するだけでよい。
build system が LLVM target triple を導出する。

将来、上級者向けに:

```json
{
  "platform": {
    "triple": "x86_64-pc-windows-msvc"
  }
}
```

の直接指定を許可できる。

`triple` と arch/os/abi が同時指定され矛盾する場合は build error。

## 4. build

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

### 4.1 profile

初期値:

- `debug`
- `release`

profile は便利な preset であり、安全性の強弱を意味しない。

**release でも safety check や Safe LLVM Validator を無効化してはならない。**

### 4.2 optimization

初期値候補:

- `none`
- `basic`
- `speed`
- `size`

MVP1 では `none` / `basic` だけ実装してもよい。

optimization は Safe C++ Compiler の意味論を弱めてはならない。

### 4.3 debug_info

debug metadata を生成するか。

```json
{
  "debug_info": true
}
```

### 4.4 output_directory

build artifact の base directory。

project root 相対 path を基本とする。

## 5. inputs

### 5.1 source_directories

source 探索 directory。

```json
{
  "source_directories": ["src", "lib"]
}
```

### 5.2 include_directories

include search path。

```json
{
  "include_directories": ["include", "vendor/foo/include"]
}
```

将来 `system_include_directories` を追加可能。

## 6. targets

初期 target type:

- `application`
- `static_library`
- `shared_library`

MVP では `application` を最優先で実装する。

target が持てる基本設定:

- `sources`
- `include_directories`
- `defines`
- `output_name`
- `links`
- `application_root`
- `copy`

## 7. sources

target が実際に compile する source を glob で指定できる。

```json
{
  "sources": [
    "src/**",
    "lib/math/*.cpp"
  ]
}
```

`source_directories` は探索 root、`sources` は target への選択とする。

## 8. defines

preprocessor define。

```json
{
  "defines": [
    "APP_VERSION=1",
    "FEATURE_X"
  ]
}
```

値なし define と `NAME=value` を許可する。

## 9. output_name

target の論理出力名。

```json
{
  "output_name": "sample"
}
```

platform に応じた suffix/prefix は build system が付ける。

例:

```text
windows application -> sample.exe
linux application   -> sample
```

MVP1 では validated LLVM IR の出力名にも利用できる。

例:

```text
build/sample.ll
build/sample.bc
```

## 10. links

```json
{
  "links": {
    "libraries": [
      "user32",
      "mylib"
    ],
    "library_directories": [
      "vendor/lib"
    ]
  }
}
```

### 10.1 libraries

system library または project library の論理名。

### 10.2 library_directories

linker の library search path。

MVP1 は machine code/link が必須ではないため、設定の parse/保持だけ先に実装してもよい。

将来必要なら `frameworks`、`link_options` 等を追加するが、raw linker option は通常設定より後回しとする。

## 11. application_root

application target の staging/package root。

```json
{
  "application_root": "Application/root"
}
```

build artifact と resource copy の配置基準。

MVP では project root 相対 path のみでよい。

## 12. directory/file copy

```json
{
  "copy": [
    {
      "from": "assets",
      "to": "assets"
    },
    {
      "from": "config/default",
      "to": "config"
    }
  ]
}
```

概念:

```text
assets/**          -> Application/root/assets/**
config/default/**  -> Application/root/config/**
```

### 12.1 from

project root 相対の file/directory。

### 12.2 to

`application_root` 相対 path。

禁止:

- absolute destination
- `..` による root escape
- platform-specific trick による root escape

### 12.3 collision

既定では destination collision は build error。

### 12.4 symlink

application root 外へ解決される symlink は error。

### 12.5 missing source

存在しない `from` は build error。

将来 optional resource は明示 `optional: true` を追加可能。

## 13. runtime / toolchain

通常必要になりやすいが platform-specific なので段階的に追加する。

将来候補:

```json
{
  "toolchain": {
    "linker": "default",
    "runtime": "dynamic"
  }
}
```

候補設定:

- linker 選択
- static/dynamic runtime
- sysroot
- SDK path
- Windows subsystem
- deployment target

ただし CMake のように任意 command を実行する script language にはしない。

## 14. build artifact

MVP1 は validated LLVM IR が中心。

例:

```text
build/
  sample.ll
  sample.bc

Application/root/
  assets/
  config/
```

将来 machine code generation を行う場合:

```text
Application/root/
  sample.exe
  assets/
  config/
```

のように配置できる。

## 15. CMake compatibility

CMake language を再実装しない。

互換性の目的:

1. 既存 CMake project を利用できる
2. target の source/include/define/link/platform 情報を import できる
3. Safe native project は CMake 不要

初期方針:

- CMake project: machine-readable codemodel/target information を adapter で import
- Safe project: `safe-build.json`
- CMake export: 将来機能

CMake command/macro/generator expression を独自に全面解釈しない。

## 16. safe-cpp.json との関係

`safe-cpp.json`:

- unsafety diagnostics
- ignore rules/files
- LLVM validation
- safety policy

`safe-build.json`:

- platform / architecture / OS / ABI
- build profile / optimization / debug info
- source/include
- targets
- defines
- output
- link
- application root
- copy/staging
- CMake adapter

## 17. path normalization

JSON path は `/` を canonical separator とする。

OS path への変換は build system が行う。

file operation 前に normalize し root escape を検査する。

## 18. CLI override

通常の build system と同様、頻繁に変える値は CLI override を将来提供できる。

例:

```text
safe-cpp build --arch x64 --os windows --profile release
```

JSON を書き換えず CI matrix / cross compile を行えるようにする。

CLI と JSON が競合する場合は CLI を優先する。

## 19. MVP build scope

最初に必要:

1. `safe-build.json` parser
2. platform.arch / platform.os / platform.abi
3. build.profile / optimization / debug_info / output_directory
4. source_directories
5. include_directories
6. application target
7. source glob
8. defines
9. output_name
10. links の parse/保持
11. application_root
12. file/directory copy
13. collision/root-escape/symlink 検査
14. CMake import adapter

package manager、install rule、custom command、general scripting language は後回し。
