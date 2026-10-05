# Remaining challenges for CPython free-threading

Summary of the main challenges, based on knowledge of the project and a look at CPython `main` (3.16 development). Free-threading has been officially supported but optional since 3.14 (PEP 779, "phase II"). Most of the remaining work is about performance, the C-extension ecosystem, and hardening, not basic correctness.

## 1. Single-threaded performance and memory cost

- **Speed:** Single-threaded code runs slower than on the default build. The gap was large in 3.13, where the specializing interpreter was disabled. In 3.14 specialization was made thread-safe and the gap shrank to roughly single digits to around 10%. Closing the rest is ongoing work: atomic refcount paths, critical-section overhead, and cache lookups.
- **Memory:** Objects have bigger headers (`ob_tid`, `ob_ref_local`, `ob_ref_shared`). The build also uses mimalloc, and QSBR (safe memory reclamation) delays freeing, all of which raises memory use.
- **JIT:** The experimental JIT and free-threading have not been fully combined yet.

## 2. Multi-threaded scaling bottlenecks

- **Refcount contention:** When many threads share hot objects (functions, types, module globals, constants), they all update the same refcount. That makes CPU cache lines bounce between cores. This is handled with deferred refcounting, immortalization, and per-thread refcounts for heap types, but not every pattern is covered.
- **Shared structures:** Shared dicts and type/method caches can become hot spots. So can some locks inside modules.
- **Garbage collection:** The free-threaded GC pauses all threads ("stop-the-world"), so collection pause time and heuristics on large heaps are still being tuned.

## 3. C extension ecosystem (the biggest practical issue)

- **GIL fallback:** An extension that doesn't declare `Py_mod_gil` / `Py_MOD_GIL_NOT_USED` **re-enables the GIL** when it's imported. This silently removes the benefit.
- **Borrowed references:** APIs that return borrowed references, such as `PyList_GetItem` and `PyDict_GetItem`, are unsafe when other threads can change the container. Extensions need to move to `*Ref` variants (`PyList_GetItemRef`, `PyDict_GetItemRef`) and use `Py_BEGIN_CRITICAL_SECTION`.
- **No stable ABI yet:** The free-threaded build doesn't support the limited API (`abi3`), so packages must ship separate `cp3XXt` wheels. A free-threading-compatible stable ABI (PEP 803 / related work) is in progress.
- **Hidden assumptions:** Many libraries relied on the GIL for thread safety without saying so, for example in global caches and lazy initialization. Those bugs only show up under real parallel load.

## 4. Thread safety of the stdlib and built-ins

- **What's protected:** Built-in containers use per-object critical sections, so a single operation won't corrupt the object.
- **What isn't:** Compound operations aren't atomic (e.g. `d[k] = d.get(k, 0) + 1`). Iterators shared between threads, and some modules (`itertools`, `io`, `ctypes`, `sqlite3`, `functools` caches and others), have needed case-by-case fixes. Documenting which guarantees actually hold is still a work in progress.
- **Python code:** Code that "worked" because of GIL timing now exposes races more often.

## 5. Testing and tooling

- **Test suite:** In the 3.16 `main` checkout, about 30 test files still use `thread_unsafe` / `requires_gil_enabled` markers. Those tests are skipped or can't be run under parallel stress.
- **TSan:** CI runs ThreadSanitizer on the free-threaded build. Notably, [`Tools/tsan/suppressions_free_threading.txt`](https://github.com/python/cpython/blob/main/Tools/tsan/suppressions_free_threading.txt) now contains only comments, so all previously known data races there have been fixed.
- **Debuggers and profilers:** Tools such as gdb helpers, `sys.monitoring`, and external inspection had to be adapted, and some edge cases remain.

## 6. Ecosystem and distribution

- **Availability:** Many projects still don't publish free-threaded wheels, and some distributors and installers treat it as a separate build.
- **Default build:** There's no decision yet on making it the default ("phase III"). That decision depends on the performance and ecosystem points above.

## Further reading

For an up-to-date list, filter the CPython issue tracker by the `topic-free-threading` label, and see the HOWTOs:

- [Python experimental support for free threading](https://docs.python.org/3/howto/free-threading-python.html) (`Doc/howto/free-threading-python.rst`)
- [C API extension support for free threading](https://docs.python.org/3/howto/free-threading-extensions.html) (`Doc/howto/free-threading-extensions.rst`)
