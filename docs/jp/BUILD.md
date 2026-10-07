# Build Configuration 仕様

> **正本 (Authoritative Specification)**
>
> この文書は Safe C++ Compiler の build configuration の日本語正本である。
> `docs/en/BUILD.md` は翻訳版。

## 1. 目的

Safe C++ Compiler は CMake と連携できるが、CMake と同じ複雑な build language を再実装しない。

通常 project では、軽量な宣言的 JSON:

```text
safe-build.json
```

を使用する。

`safe-cpp.json` は安全性 policy、`safe-build.json` は build/project 構成として役割を分離する。

## 2. 基本例

```json
{
  "version": 1,
  "project": {
    "name": "sample"
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

## 3. inputs

### 3.1 source_directories

compiler/build system が source を探索する directory。

```json
{
  "inputs": {
    "source_directories": [
      "src",
      "lib"
    ]
  }
}
```

各 path は project root 相対を基本とする。

source file の実際の選択は target の `sources` で絞り込める。

### 3.2 include_directories

C/C++ の include search path に追加する directory。

```json
{
  "inputs": {
    "include_directories": [
      "include",
      "vendor/foo/include"
    ]
  }
}
```

MVP では user include directory として扱う。

将来 `system_include_directories` を別に追加してよい。

## 4. targets

初期 target type:

- `application`
- `static_library`
- `shared_library`

MVP では `application` を最優先で実装する。

target は最低限次を持てる。

- `sources`
- `include_directories`
- `defines`
- `links`
- `application_root`
- `copy`

project-level inputs と target-level設定が両方ある場合、target-level設定を追加分として扱う。

## 5. application_root

application target の staging/package root。

例:

```json
{
  "application_root": "Application/root"
}
```

build artifact や resource copy の配置先基準になる。

`application_root` は project root 相対 path を基本とする。

絶対 path を許可するかは将来の明示 option とし、MVP では相対 path のみでよい。

## 6. directory copy

application root へ directory/file をコピーできる。

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

上記は概念的に:

```text
assets/**          -> Application/root/assets/**
config/default/**  -> Application/root/config/**
```

となる。

### 6.1 copy.from

project root 相対 source path。

file または directory を指定できる。

### 6.2 copy.to

`application_root` 相対 destination。

以下は禁止:

- absolute path
- `..` による application root 外への escape
- platform-specific path trick で root 外へ出る指定

### 6.3 collision

既定では destination collision は build error。

将来:

```json
{
  "copy_policy": {
    "on_collision": "error"
  }
}
```

のような設定を追加可能。

### 6.4 symlink

MVP では symlink が application root 外を指す場合は error とする。

安全に解決できない symlink traversal は許可しない。

### 6.5 missing source

`copy.from` が存在しない場合は build error。

将来 optional resource を追加する場合は明示的な `optional: true` を用いる。

## 7. build artifact

将来 machine code を生成する application target では、executable/shared resources を application root に配置できる。

MVP1 は validated LLVM IR が中心なので、artifact staging は最小実装でもよい。

```text
Application/root/
  app.ll
  assets/
  config/
```

のような staging を許可する。

## 8. CMake compatibility

CMake の言語そのものを Safe C++ Compiler 内で再実装しない。

互換性の目的は:

1. 既存 CMake project を Safe C++ Compiler から利用できること
2. CMake target の source/include/define/link 情報を取り込めること
3. Safe C++ Compiler 固有の単純 project では CMake を書かなくてもよいこと

CMake integration は adapter として実装する。

初期方針:

- 既存 CMake project: CMake の machine-readable project/codemodel 情報から import
- Safe project: `safe-build.json` を直接利用
- CMake への export は将来機能

CMake の全 command / macro / generator expression を独自解釈しない。

## 9. safe-cpp.json との関係

`safe-cpp.json`:

- unsafety diagnostics
- ignore rules/files
- LLVM validation
- safety policy

`safe-build.json`:

- source directories
- include directories
- targets
- defines
- links
- application root
- copy/staging
- CMake adapter

安全 policy と build description を混ぜない。

## 10. path normalization

すべての JSON path は `/` を canonical separator とする。

実OSの path separator への変換は build system が行う。

path 比較前に lexical normalization を行い、root escape を検査する。

## 11. MVP build scope

最初に必要なもの:

1. `safe-build.json` parser
2. source_directories
3. include_directories
4. application target
5. source glob
6. defines
7. application_root
8. copy directory/file
9. collision/root-escape 検査
10. CMake project import adapter

library target、install、package manager、custom command、script language は後回しでよい。
