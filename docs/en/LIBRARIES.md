# Dangerous Library / API Catalog Specification

> **English translation**
>
> The Japanese `docs/jp/LIBRARIES.md` is authoritative.

## 1. Purpose

Safe C++ Compiler maintains a safety catalog for C/C++ library APIs, including the standard library, and can emit `unsafety warning` or `unsafety error` diagnostics when risky APIs are used.

Libraries and headers are not blanket-banned. Classification is performed at the API/symbol level.

For example, `std::vector` itself is normally allowed, while an unproven `std::vector::operator[]` access is checked.

## 2. Severity

Catalog entries may be:

- `safe` — no special diagnostic
- `checked` — compiler proves safety or inserts a runtime check
- `warning` — defined lowering is possible but misuse is common
- `error` — safety cannot be guaranteed

## 3. Initial standard-library catalog

### Unchecked access

Examples include:

- `std::vector::operator[]`
- `std::array::operator[]`
- `std::basic_string::operator[]`
- `std::basic_string_view::operator[]`
- `std::span::operator[]`
- unchecked `std::optional` / `std::expected` dereference

Bounds or state are proven statically or lowered to checks/traps.

### Raw memory and C strings

Examples include `std::memcpy`, `std::memmove`, `std::memset`, `std::strcpy`, `std::strcat`, and related C APIs.

Size, lifetime, overlap, and object-representation conditions determine whether the call is safe, checked, warning, or error.

### Formatted I/O

`printf` / `scanf` families and `sprintf` / `snprintf` are analyzed for format types and destination capacities.

### Ownership transfer

Examples:

- `std::unique_ptr::release()` -> warning
- raw-pointer smart-pointer construction that may create duplicate ownership -> warning/error
- raw pointers obtained from smart pointers escaping owner lifetime -> warning/error

### Non-owning lifetime views

Examples include `std::string_view`, `std::span`, iterators, pointer ranges, and raw pointers returned by `.data()`.

Potential dangling produces warning/error; statically proven dangling is an error.

### Manual lifetime/object model

Placement new, `std::launder`, allocator raw-storage APIs, `std::construct_at`, `std::destroy_at`, and uninitialized-memory algorithms require lifetime tracking.

### Raw allocation

`malloc/calloc/realloc/free` and direct `new/delete` are tracked rather than automatically forbidden. Unresolved ownership hazards produce warning/error.

### Concurrency

Threading APIs are not blanket-banned. Unproven races produce warning/error; definite data races are errors.

## 4. Third-party libraries

Projects may load additional catalogs, for example:

```json
{
  "library_catalogs": [
    "safe-library-rules.json"
  ]
}
```

Project catalogs may add or strengthen rules. Weakening built-in safety rules requires an explicit safety exception.

## 5. Catalog entry model

Conceptual example:

```json
{
  "symbol": "std::unique_ptr::release",
  "category": "ownership_transfer",
  "severity": "warning",
  "message": "ownership is transferred to a raw pointer"
}
```

Conditional checks are supported, for example for `std::memcpy` destination/source size and representation compatibility.

## 6. Symbol resolution

The checker uses resolved declarations/canonical symbols rather than source-text matching.

Namespaces, members, overloads, templates, C linkage, and macro wrappers must be distinguishable.

## 7. Library summaries

Functions may have safety summaries describing reads, writes, preconditions, ownership, lifetime escape, and trap behavior.

MVP may start with built-in summaries and later infer summaries for project functions.

## 8. Suppression

Library warnings use stable rule keys and may be suppressed through `ignore.rules`.

Safe LLVM Validator cannot be bypassed by suppressing a library warning.

## 9. Versioning

The standard-library catalog tracks the C23/C++23 baseline and compiler version.

Third-party catalog entries should eventually support library-version conditions.

## 10. Initial implementation

1. resolved-symbol library checker
2. built-in standard/C library catalog
3. safe/checked/warning/error classification
4. cstring/cstdio families
5. unchecked container/string/span access
6. smart-pointer ownership escape
7. string_view/span lifetime escape
8. raw allocation/free
9. diagnostics
10. project-local catalog loading
