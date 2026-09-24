# cache

[![CI](https://github.com/alya-lang/cache/actions/workflows/ci.yml/badge.svg)](https://github.com/alya-lang/cache/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/alya-lang/cache?color=blue&label=License)](LICENSE)
[![Alya](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fcache%2Fmain%2Falya.toml&query=%24.package.alya-version&label=Alya&color=orange&prefix=%3E%3D)](https://github.com/alya-lang/alya)
[![Package Version](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fcache%2Fmain%2Falya.toml&query=%24.package.version&label=Version&color=brightgreen)](alya.toml)

High-performance in-memory cache with LRU eviction, TTL expiry, and statistics for Alya

---

## 🌟 Features

- ⚡ **Zero-Dependency & Pure Alya**: No C code, no FFI, no external services. Values of any type (string, int, array, map) side by side.
- 🧩 **LRU / FIFO / Unbounded Flavors**: Least-recently-used, first-in-first-out, or no-eviction caches from one API.
- ⏱️ **Per-Entry TTL**: Default TTL per cache plus per-write override, remaining-time queries, refresh, and eager expiry on read.
- 📦 **Batch Helpers**: `set_many` / `get_many` bulk operations and `get_or_insert` memoize helper.
- 📊 **Built-In Statistics**: Hits, misses, evictions, hit-rate, and one-line summaries via `CacheStats`.
- 🧹 **Maintenance APIs**: `prune` expired entries, snapshot `keys`, `clear` with counter reset.
- 🔒 **Public/Private Visibility (`pub`)**: Clean facade in `src/lib.alya`; storage engine isolated in `src/core/store.alya`.
- 🧪 **Enterprise Test & Benchmark Suite**: 90+ assertions (`std/test`) and micro-benchmarks for set/get/evict paths.

---

## 📁 Project Architecture

```
cache/
├── .alyalint               # Linter configuration (rules, exclusions, severity overrides)
├── .editorconfig           # Uniform formatting rules across IDEs and editors
├── .gitignore              # Ecosystem standard ignore filters
├── .vscode/                # VS Code workspace settings, DAP launch configurations & tasks
├── alya.toml               # Package manifest with dependencies and optional [build]
├── src/
│   ├── lib.alya            # Public API facade (constructors, Cache methods, functional facades)
│   ├── types.alya          # EvictionPolicy, CacheOptions, CacheMeta, Cache, CacheStats + factories
│   └── core/
│       └── store.alya      # Storage engine: set/get/has/delete/TTL/eviction/prune leaves
├── examples/
│   └── demo.alya           # Comprehensive runnable walkthrough of all package capabilities
├── tests/
│   └── test_basic.alya     # Automated test suite with full feature coverage
└── benches/
    └── bench_basic.alya    # Micro-benchmarks measuring set/get/evict throughput
```

> [!NOTE]
> **Storage Layout:** User values live directly in the `store` map (maps handle mixed `any` values correctly). Expiry and hit counters live as statically-typed `CacheMeta` structs in the parallel `meta` map. Recency order is tracked in the `order` array (front = eviction candidate).
>
> **Iteration Discipline:** Loops that iterate order snapshots live in the `src/lib.alya` facade and call loop-free `store` leaves once per key; iterated keys are copied (`"" + k`) before struct calls. This layout works around runtime crashes observed with deeper struct-threading chains.

---

## 📦 Installation

Add `cache` to the `[dependencies]` section in your `alya.toml`:

```toml
[dependencies]
cache = { git = "https://github.com/alya-lang/cache", branch = "main" }
```

Or install it directly using the Alya package CLI:

```bash
alya add cache --git https://github.com/alya-lang/cache --branch main
alya install
```

---

## 🚀 Quick Start

```alya
import "cache" as cache

function main()
    # 1. LRU cache with 128 slots and 60s default TTL
    let c = cache::new_cache(128, 60)
    c.set("user:1", "Ada")
    c.set("session:token", "abc123", 3600)

    say c.get_or("user:1", "missing") # "Ada"
    say c.ttl("session:token")        # ~3600

    # 2. Eviction flavors
    let fifo = cache::fifo_cache(2, 0)
    let big = cache::unbounded_cache()

    # 3. Batch + memoize
    c.set_many({ "a": 1, "b": 2 })
    let many = c.get_many(["a", "b", "nope"]) # {"a": 1, "b": 2}
    let v = c.get_or_insert("memo", 99)

    # 4. Stats & maintenance
    say c.summary()
    say c.prune() # expired entries removed
end

main()
```

TTL semantics: `ttl < 0` uses the cache default, `ttl == 0` never expires, `ttl > 0` expires that many seconds from now. `capacity <= 0` means unbounded.

---

## 📖 API Reference

### Constructors

| Symbol | Visibility | Description |
|---|---|---|
| `new_cache(capacity = 128, default_ttl = 0, policy = EvictionPolicy.LRU)` | `pub function` | Creates a cache with explicit settings. |
| `lru_cache(capacity = 128, default_ttl = 0)` | `pub function` | Creates an LRU cache. |
| `fifo_cache(capacity = 128, default_ttl = 0)` | `pub function` | Creates a FIFO cache. |
| `unbounded_cache(default_ttl = 0)` | `pub function` | Creates a cache with no eviction. |
| `cache_from_options(opts)` | `pub function` | Creates a cache from a `CacheOptions` instance. |
| `options_new(capacity, default_ttl, policy)` | `pub function` | Builds a `CacheOptions` record. |
| `options_lru(capacity, default_ttl)` | `pub function` | Builds LRU options. |
| `options_fifo(capacity, default_ttl)` | `pub function` | Builds FIFO options. |
| `options_unbounded(default_ttl)` | `pub function` | Builds unbounded options. |
| `meta_new(ttl = 0)` | `pub function` | Builds a `CacheMeta` record stamped with the current time. |
| `make_stats(size, capacity, hits, misses, evictions, policy)` | `pub function` | Builds a `CacheStats` snapshot record. |

### Cache Methods

| Symbol | Visibility | Description |
|---|---|---|
| `Cache.set(self, key, value, ttl = -1)` | `pub method` | Inserts or overwrites `key`. Returns `1`. |
| `Cache.get(self, key)` | `pub method` | Returns the value, or null on miss/expiry. |
| `Cache.get_or(self, key, fallback)` | `pub method` | Returns the value, or `fallback` on miss/expiry. |
| `Cache.get_or_insert(self, key, value, ttl = -1)` | `pub method` | Returns the live value, inserting `value` first on miss. |
| `Cache.has(self, key)` | `pub method` | Returns `1` when `key` is live, `0` otherwise. |
| `Cache.delete(self, key)` | `pub method` | Deletes `key`. Returns `1` when removed. |
| `Cache.remove(self, key)` | `pub method` | Alias for `delete`. |
| `Cache.clear(self)` | `pub method` | Removes all entries and resets counters. |
| `Cache.size(self)` | `pub method` | Returns the entry count. |
| `Cache.len(self)` | `pub method` | Returns the entry count. |
| `Cache.is_empty(self)` | `pub method` | Returns `1` when empty. |
| `Cache.keys(self)` | `pub method` | Returns a snapshot array of live keys (prunes first). |
| `Cache.prune(self)` | `pub method` | Removes expired entries. Returns pruned count. |
| `Cache.stats(self)` | `pub method` | Returns a `CacheStats` snapshot. |
| `Cache.ttl(self, key)` | `pub method` | Remaining TTL seconds (`-1` missing/expired, `0` persist). |
| `Cache.expire(self, key, ttl)` | `pub method` | Refreshes a live entry's TTL. Returns `1` on success. |
| `Cache.set_many(self, values, ttl = -1)` | `pub method` | Bulk insert from a map. Returns inserted count. |
| `Cache.get_many(self, key_list)` | `pub method` | Bulk read into a live key -> value map. |
| `Cache.hit_rate(self)` | `pub method` | Hit-rate percentage string (e.g. `"80%"`). |
| `Cache.summary(self)` | `pub method` | One-line size/counters/hit-rate summary. |

### Functional Facades

`cache_set`, `cache_get`, `cache_get_or`, `cache_has`, `cache_delete`, `cache_clear`, `cache_size`, `cache_keys`, `cache_prune`, `cache_stats`, `cache_set_many`, `cache_get_many` — same behavior with an explicit `Cache` first argument.

### Types

| Symbol | Visibility | Description |
|---|---|---|
| `EvictionPolicy` | `pub enum` | `None = 0`, `LRU = 1`, `FIFO = 2`. |
| `CacheOptions` | `pub struct` | `capacity`, `default_ttl`, `policy`. |
| `CacheMeta` | `pub struct` | `expires_at`, `inserted_at`, `entry_hits` (+ `is_expired`, `is_alive`, `ttl_remaining`). |
| `Cache` | `pub struct` | Container with `store` / `meta` maps, `order`, `capacity`, `default_ttl`, `policy`, `hits`, `misses`, `evictions`. |
| `CacheStats` | `pub struct` | Snapshot with `total_requests`, `hit_rate`, `summary`. |

> [!TIP]
> **Internal Helpers & Documentation:** Public symbols are documented with `##` Markdown docstrings, enabling automatic API documentation generation via `alya doc`. Private functions such as `resolve_ttl`, `detach_key`, and `prune_key` in `src/core/store.alya` are not annotated with `pub` and remain encapsulated.

---

## 🧪 Running Tests & Benchmarks

Run the automated test suite using `alya test`:

```bash
alya test
```

Generate static API documentation:

```bash
alya doc . -o docs --markdown
```

Run the benchmark suite:

```bash
alya run benches/bench_basic.alya
```

Run the example demo:

```bash
alya run examples/demo.alya
```

Check code formatting:

```bash
alya fmt . --check
```

Run static code linter:

```bash
alya lint . --check
```

---

### 💻 Developer Tooling & VS Code Integration

This package comes preconfigured with recommended workspace settings and tasks for **Visual Studio Code**:
- **LSP & Formatting**: Auto-formatting on save and real-time Language Server diagnostics via `alya-lang.vscode-alya`.
- **DAP Debugging**: Launch configurations in `.vscode/launch.json` ready for interactive step-debugging via `F5`.
- **Predefined Tasks**: Press `Ctrl+Shift+B` or run tasks (`Test`, `Lint`, `Format`, `Build Docs`) directly from the Command Palette.

---

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository and clone it locally
2. Install dependencies:
   ```bash
   alya install
   ```
3. Create your feature branch (`git checkout -b feature/my-feature`)
4. Verify tests and formatting before opening a PR:
   ```bash
   alya test
   ```
5. Commit your changes (`git commit -m "feat: add feature"`) and open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
