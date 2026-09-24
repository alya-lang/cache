# cache

[![CI](https://github.com/alya-lang/cache/actions/workflows/ci.yml/badge.svg)](https://github.com/alya-lang/cache/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/alya-lang/cache?color=blue&label=License)](LICENSE)
[![Alya](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fcache%2Fmain%2Falya.toml&query=%24.package.alya-version&label=Alya&color=orange&prefix=%3E%3D)](https://github.com/alya-lang/alya)
[![Package Version](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fcache%2Fmain%2Falya.toml&query=%24.package.version&label=Version&color=brightgreen)](alya.toml)

High-performance in-memory cache with LRU/LFU/FIFO eviction, TTL expiry, hooks, snapshots, and statistics for Alya

---

## 🌟 Features

- ⚡ **Zero-Dependency & Pure Alya**: No C code, no FFI, no external services. Values of any type (string, int, array, map) side by side.
- 🧩 **LRU / LFU / FIFO / Unbounded Flavors**: Least-recently-used, least-frequently-used, first-in-first-out, or no-eviction caches from one API.
- ⏱️ **Per-Entry TTL**: Default TTL per cache plus per-write override, remaining-time queries, refresh (`expire`/`touch`), TTL jitter against stampedes, and eager expiry on read.
- 👀 **Peek & Touch**: Non-intrusive reads (`peek`) and recency/TTL refresh without value changes (`touch`).
- 🔧 **Runtime Reconfiguration**: Resize capacity (`set_capacity`, evicts excess on shrink) and change the default TTL live.
- 🗂️ **Batch & Prefix Helpers**: `set_many` / `get_many` bulk operations, `delete_by_prefix` invalidation, and `get_or_insert` / `get_or_compute` memoize helpers.
- 🔔 **Eviction & Expiry Hooks**: `on_evict` / `on_expire` callbacks (`fn(key, value)`) for external indexes and cleanup.
- 💾 **Disk Snapshots**: `save` / `load` line-based snapshots for string/int/float entries written via typed setters.
- ✍️ **Typed Setters**: `set_str` / `set_int` / `set_float` record kind tags (snapshots, `get_float` reads).
- 📊 **Built-In Statistics**: Hits, misses, evictions, hit-rate, and one-line summaries via `CacheStats`.
- 🧹 **Maintenance APIs**: `prune` expired entries, snapshot `keys`, `clear` with counter reset.
- 🔒 **Public/Private Visibility (`pub`)**: Clean facade in `src/lib.alya`; storage engine (`src/core/store.alya`) and snapshots (`src/core/snapshot.alya`) isolated.
- 🧪 **Enterprise Test & Benchmark Suite**: 150+ assertions (`std/test`) and micro-benchmarks for set/get/peek/evict paths.

> [!NOTE]
> **Performance characteristics:** Point operations (`set`/`get`/`has`/`delete`) are O(1) map operations, except LRU reads which refresh recency in O(n) over the order array. LRU/FIFO eviction is O(1) amortized; LFU eviction scans frequencies in O(n). Best suited for small-to-medium caches (hundreds to low thousands of entries).

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
│       ├── store.alya      # Storage engine: set/get/has/delete/TTL/eviction/prune leaves
│       └── snapshot.alya   # Snapshot format: escaping, float codec, line builder/parser
├── examples/
│   └── demo.alya           # Comprehensive runnable walkthrough of all package capabilities
├── tests/
│   └── test_basic.alya     # Automated test suite with full feature coverage
└── benches/
    └── bench_basic.alya    # Micro-benchmarks measuring set/get/evict throughput
```

> [!NOTE]
> **Storage Layout:** User values live directly in the `store` map (maps handle mixed `any` values correctly). Expiry, hit counters, and kind tags live as statically-typed `CacheMeta` structs in the parallel `meta` map. Recency order is tracked in the `order` array (front = eviction candidate). Floats set via `set_float` are kept in fixed-point string form and decoded transparently by `get_float`.
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

    # 2. Eviction flavors + hooks
    let fifo = cache::fifo_cache(2, 0)
    let lfu = cache::lfu_cache(64, 0)
    let big = cache::unbounded_cache()
    c.on_evict(note_evicted)

    # 3. Batch, prefix & memoize
    c.set_many({ "a": 1, "b": 2 })
    let many = c.get_many(["a", "b", "nope"]) # {"a": 1, "b": 2}
    let v = c.get_or_insert("memo", 99)
    let w = c.get_or_compute("lazy", make_value)
    c.delete_by_prefix("tmp:")

    # 4. Typed values + snapshot
    c.set_float("score", 9.5)
    say c.get_float("score") # 9.5
    c.save("cache.snap")
    c.load("cache.snap")

    # 5. Stats & maintenance
    say c.summary()
    say c.prune() # expired entries removed
end

function note_evicted(key, value)
    return 1
end

function make_value(key)
    return "made:" + key
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
| `lfu_cache(capacity = 128, default_ttl = 0)` | `pub function` | Creates an LFU cache. |
| `unbounded_cache(default_ttl = 0)` | `pub function` | Creates a cache with no eviction. |
| `cache_from_options(opts)` | `pub function` | Creates a cache from a `CacheOptions` instance. |
| `options_new(capacity, default_ttl, policy)` | `pub function` | Builds a `CacheOptions` record. |
| `options_lru(capacity, default_ttl)` | `pub function` | Builds LRU options. |
| `options_fifo(capacity, default_ttl)` | `pub function` | Builds FIFO options. |
| `options_lfu(capacity, default_ttl)` | `pub function` | Builds LFU options. |
| `options_unbounded(default_ttl)` | `pub function` | Builds unbounded options. |
| `meta_new(ttl = 0, kind = "")` | `pub function` | Builds a `CacheMeta` record stamped with the current time. |
| `make_stats(size, capacity, hits, misses, evictions, policy)` | `pub function` | Builds a `CacheStats` snapshot record. |

### Cache Methods

| Symbol | Visibility | Description |
|---|---|---|
| `Cache.set(self, key, value, ttl = -1)` | `pub method` | Inserts or overwrites `key`. Returns `1`. |
| `Cache.set_str(self, key, value, ttl = -1)` | `pub method` | Inserts a string, tagged for snapshots. |
| `Cache.set_int(self, key, value, ttl = -1)` | `pub method` | Inserts an int, tagged for snapshots. |
| `Cache.set_float(self, key, value, ttl = -1)` | `pub method` | Inserts a float, tagged for snapshots. |
| `Cache.set_jittered(self, key, value, ttl, max_jitter = 0)` | `pub method` | Inserts with TTL jitter against stampedes. |
| `Cache.get(self, key)` | `pub method` | Returns the value, or null on miss/expiry. |
| `Cache.get_float(self, key)` | `pub method` | Returns the float for a `set_float` entry, or null. |
| `Cache.peek(self, key)` | `pub method` | Reads without recency or counter effects. |
| `Cache.get_or(self, key, fallback)` | `pub method` | Returns the value, or `fallback` on miss/expiry. |
| `Cache.get_or_insert(self, key, value, ttl = -1)` | `pub method` | Returns the live value, inserting `value` first on miss. |
| `Cache.get_or_compute(self, key, producer, ttl = -1)` | `pub method` | Returns the live value, computing via `fn(key)` first on miss. |
| `Cache.has(self, key)` | `pub method` | Returns `1` when `key` is live, `0` otherwise. |
| `Cache.delete(self, key)` | `pub method` | Deletes `key`. Returns `1` when removed. |
| `Cache.remove(self, key)` | `pub method` | Alias for `delete`. |
| `Cache.delete_by_prefix(self, prefix)` | `pub method` | Deletes keys starting with `prefix`. Returns count. |
| `Cache.clear(self)` | `pub method` | Removes all entries and resets counters. |
| `Cache.size(self)` | `pub method` | Returns the entry count. |
| `Cache.len(self)` | `pub method` | Returns the entry count. |
| `Cache.is_empty(self)` | `pub method` | Returns `1` when empty. |
| `Cache.keys(self)` | `pub method` | Returns a snapshot array of live keys (prunes first). |
| `Cache.prune(self)` | `pub method` | Removes expired entries. Returns pruned count. |
| `Cache.stats(self)` | `pub method` | Returns a `CacheStats` snapshot. |
| `Cache.ttl(self, key)` | `pub method` | Remaining TTL seconds (`-1` missing/expired, `0` persist). |
| `Cache.expire(self, key, ttl)` | `pub method` | Refreshes a live entry's TTL. Returns `1` on success. |
| `Cache.touch(self, key, ttl = -1)` | `pub method` | Refreshes recency/TTL without changing value. |
| `Cache.set_capacity(self, capacity)` | `pub method` | Resizes capacity, evicting excess on shrink. |
| `Cache.set_default_ttl(self, ttl)` | `pub method` | Updates the default TTL for future `set` calls. |
| `Cache.on_evict(self, cb)` | `pub method` | Registers eviction callback `fn(key, value)`. |
| `Cache.on_expire(self, cb)` | `pub method` | Registers expiry callback `fn(key, value)`. |
| `Cache.set_many(self, values, ttl = -1)` | `pub method` | Bulk insert from a map. Returns inserted count. |
| `Cache.get_many(self, key_list)` | `pub method` | Bulk read into a live key -> value map. |
| `Cache.save(self, path)` | `pub method` | Persists typed entries to a snapshot file. Returns count. |
| `Cache.load(self, path)` | `pub method` | Restores entries from a snapshot file. Returns count. |
| `Cache.hit_rate(self)` | `pub method` | Hit-rate percentage string (e.g. `"80%"`). |
| `Cache.summary(self)` | `pub method` | One-line size/counters/hit-rate summary. |

### Functional Facades

`cache_set`, `cache_set_str`, `cache_set_int`, `cache_set_float`, `cache_set_jittered`, `cache_get`, `cache_get_float`, `cache_peek`, `cache_get_or`, `cache_get_or_insert`, `cache_get_or_compute`, `cache_has`, `cache_delete`, `cache_delete_by_prefix`, `cache_clear`, `cache_size`, `cache_keys`, `cache_prune`, `cache_stats`, `cache_set_many`, `cache_get_many`, `cache_touch`, `cache_set_capacity`, `cache_set_default_ttl`, `cache_on_evict`, `cache_on_expire`, `cache_save`, `cache_load` — same behavior with an explicit `Cache` first argument.

### Types

| Symbol | Visibility | Description |
|---|---|---|
| `EvictionPolicy` | `pub enum` | `None = 0`, `LRU = 1`, `FIFO = 2`, `LFU = 3`. |
| `CacheOptions` | `pub struct` | `capacity`, `default_ttl`, `policy`. |
| `CacheMeta` | `pub struct` | `expires_at`, `inserted_at`, `entry_hits`, `kind` (+ `is_expired`, `is_alive`, `ttl_remaining`). |
| `Cache` | `pub struct` | Container with `store` / `meta` maps, `order`, `capacity`, `default_ttl`, `policy`, `hits`, `misses`, `evictions`, `evict_hook`, `expire_hook`. |
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
