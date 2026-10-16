# MTNDD — Parallel Implementation

**Multi-Terminal Network Decision Diagram (MTNDD)** represents symbolic functions
whose values may be Boolean, rational, or floating point. Like NDD, each node
branches on a field, and each edge represents a set of field values rather than
one concrete value. Numeric terminals extend this structure to weighted counts,
probabilities, costs, and other algebraic computations.

The `parallel` branch provides a **C11 core + JNI + Java instance API**.
It supports BDD, BCDD, ZDD, BITSET, and mixed fields, with parallel arithmetic,
sum abstraction, fused multiplication and abstraction, and explicit root ownership.
The implementation shares the compact native manager and Lace scheduling
infrastructure with the Boolean [`parallel`](https://github.com/XJTU-NetVerify/NDD/tree/parallel)
branch.

## Choose a Branch

| Branch | Implementation | Numeric domains | Java entry point |
| --- | --- | --- | --- |
| [`main`](https://github.com/XJTU-NetVerify/MTNDD/tree/main) | Java | Rational, double, interval | `org.ants.jndd.diagram.NDD` |
| [`parallel`](https://github.com/XJTU-NetVerify/MTNDD/tree/parallel) (this branch) | C11 + JNI, Lace worker pool | Checked rational, finite double | `org.ants.jndd.parallel.ParallelMTNDD` |

MTNDD was extracted from [NDD](https://github.com/XJTU-NetVerify/NDD), preserving the commit
history of `mtndd` as `main` and `parallel-MTNDD` as `parallel`. Boolean-only NDD remains in
that repository. Historical Java package names and the Maven artifact `org.ants:ndd:1.0.1`
are retained for compatibility. The two MTNDD branches have different APIs and numeric policies;
see [migration notes](mtndd/docs/MIGRATION.md) before porting an application.

## Build

Requirements:

- Linux, GCC or Clang with C11 and `__int128` support
- CMake 3.23 or newer, Python 3, and Git
- JDK 11 or newer and Maven 3.x

```bash
git clone --branch parallel https://github.com/XJTU-NetVerify/MTNDD.git
cd MTNDD
export JAVA_HOME=/path/to/jdk # replace with your JDK installation

cmake -S native -B native/build -DCMAKE_BUILD_TYPE=Release
cmake --build native/build -j6
mvn -DskipTests package
```

The build produces `native/build/libpmtndd.a`,
`native/build/libpmtnddjni.so`, and `target/ndd-1.0.1.jar`.
CMake fetches pinned Sylvan and Lace dependencies. For an offline build,
`PNDD_SYLVAN_SOURCE` and `FETCHCONTENT_SOURCE_DIR_LACE` may point to local
checkouts of the pinned revisions.

Run the included example:

```bash
java -Dpmtndd.library="$PWD/native/build/libpmtnddjni.so" \
  -cp target/ndd-1.0.1.jar application.examples.ParallelMTNDDExample
```

## Minimal Example

```java
import org.ants.jndd.parallel.*;
import static org.ants.jndd.parallel.ParallelMTNDD.*;

try (ParallelMTNDD engine = new ParallelMTNDD(
        new ParallelNDDConfig(6), Domain.RATIONAL,
        new Field(2, Backend.BCDD), new Field(2, Backend.ZDD));
     MTNDDNode selected = engine.literal(0, 0, true);
     MTNDDNode weight = engine.fraction(1, 3);
     MTNDDNode weighted = engine.times(selected, weight);
     MTNDDNode sum = engine.sumAbstract(weighted, 0);
     MTNDDNode fused = engine.multiplySumAbstract(selected, weight, 0)) {

    System.out.println(engine.terminalValue(sum)); // 2/3
    System.out.println(sum.equals(fused));         // true
    System.out.println(engine.satCount(weighted)); // 8.0
}
```

Fields are numbered from zero, and bits are numbered from most to least
significant by default. Each field above has four possible assignments. Selecting
one bit leaves two assignments in the first field; summing a weight of `1/3` over
that field therefore gives `2/3`.

Every returned `MTNDDNode` owns one native root reference. Operations preserve
their inputs; `retain()` creates another owned reference. Use
try-with-resources or `close()` to release nodes and engines. Unclosed nodes
remain rooted until the engine closes; there is no automatic Cleaner.

`satCount` counts assignments with a **nonzero value**, not the sum of terminal
values. To compute a weighted total, use `sumAbstract` to eliminate every field.

## Label Backends

Backend selection changes the edge representation, while a width-`w` field
always denotes the same `2^w` bit-vector assignments. Fields using the same
backend share a variable pool and a right-aligned variable layout by default. Different fields
may select different backends within one diagram.

| Backend | Representation | Domain limit |
| --- | --- | --- |
| `BDD` | Reduced ordered BDD without complemented edges | 1–4096 bits |
| `BCDD` | Sylvan BDD with complemented edges | 1–4096 bits |
| `ZDD` | Set-family ZDD with an explicit field universe | 1–4096 bits |
| `BITSET` | Canonical truth table in two 64-bit words | 1–7 bits |

The numeric domain is selected per engine:

- `Domain.RATIONAL` uses normalized signed 64-bit numerators and positive
  64-bit denominators, with checked `__int128` intermediates.
- `Domain.DOUBLE` uses finite IEEE doubles without approximate clustering or
  near-zero pruning. Signed zero is canonicalized to zero.

Overflow, division by zero, and nonfinite values or intermediate weights fail
explicitly. Rational arithmetic is bounded rather than arbitrary precision;
floating-point reductions may differ with evaluation order. See
[capabilities and numerical limits](mtndd/docs/STATUS.md) for details.

## Core API

| Purpose | Main methods |
| --- | --- |
| Construction | `constant`, `literal`, `value`, `fraction`, `number` |
| Boolean operations | `apply`, `and`, `or`, `not`, `diff`, `exist` |
| Arithmetic | `plus`, `minus`, `times`, `divide` |
| Abstraction | `sumAbstract`, `multiplySumAbstract`, `sumAbstractBatch`, `multiplySumAbstractBatch` |
| Batch operations | `applyBatch`, `reduce` |
| Dense import | `fromArray`, `fromFractions` |
| Inspection | `evaluate`, `evaluateBits`, `terminalValue`, `maxTerminal`, `satCount`, `stats` |
| Lifetime | `retain`, `close`, `collect` |

Boolean operations and `exist` accept only diagrams with values `0` and `1`.
Dense import uses one array entry per assignment, with field zero varying
slowest and the last field varying fastest. `reduce` supports AND, OR, PLUS,
and TIMES with either a left fold or a balanced tree.

Runtime settings are supplied through `ParallelNDDConfig`, including worker
count, core memory budget, cache capacity, task thresholds, and independent
switches for NDD, label, and batch parallelism. The core budget does not include
the JVM, JNI array copies, or thread stacks.

## Parallel Implementation

Recursive arithmetic partitions work by intersections of edge labels and
processes independent child problems in Lace tasks. Boolean operations use the
shared optimized Boolean kernels. All task layers use the engine's worker pool.
Small problems execute directly to avoid task overhead.

`applyBatch` and `reduce` each enter one top-level computation RUN, with
CALL/SPAWN inside it. Boundary collections are counted separately in
`externalRuns`. Dense import also uses one RUN and creates terminals directly
in C, avoiding a Java wrapper for each input value. Fused
`multiplySumAbstract` aggregates weighted children at the eliminated field
without materializing that layer of the product diagram.

The shared manager uses contiguous edge storage, sharded canonical tables,
versioned caches, and allocation cookies to detect stale node handles.
Numeric leaves participate in the same canonical table and collection protocol.

NDD collection takes place at operation boundaries after tasks finish. Sylvan
label collection may occur during computation; Lace pauses workers while the
collector marks node labels and temporary worker roots. This is cooperative
stop-the-world collection. Failed batches do not return partially owned results,
and budget exhaustion produces an error rather than a corrupted diagram.

Java calls and close operations serialize admission; computation inside an
operation can run in parallel. `ParallelNDD` and `ParallelMTNDD` share an
admission gate, allowing one active native engine at a time. An engine can be
closed before switching to the other API.

### Relationship to NDD and the Java Core

The [Java MTNDD core](https://github.com/XJTU-NetVerify/MTNDD/tree/main) offers the historical
static `NDD` API with rational, double, and interval modes. Its `satCount` counts matches to a
target value (default `1`); this branch's `satCount` counts nonzero assignments. Its rational
implementation may round fractions that outgrow exact storage, while this native core reports
overflow. The [Boolean parallel NDD core](https://github.com/XJTU-NetVerify/NDD/tree/parallel)
provides `ParallelNDD` and `NDDNode` without numeric terminals.

This branch draws backend dispatch, dense import, and fused abstraction designs
from the Java implementation (formerly `NDD/mtndd`). It reuses `native/src/pndd.c` and
the production optimization settings from `NDD/parallel`, adding the numeric algorithms
in `pmtndd_impl.inc`.
Interval terminals, approximate clustering, and n-ary contraction have not been
ported. [Migration notes and provenance](mtndd/docs/MIGRATION.md) describe these
choices and semantic differences.

The earlier MTPNDD C/JNI prototype is retained under [`mtndd/`](mtndd/README.md).
Its borrowed-pointer Java API is deprecated and should run in a separate
process. The new API's ownership and GC guarantees do not apply to that prototype.

## Batch abstraction and exact queries

`sumAbstractBatch(nodes, field)` and
`multiplySumAbstractBatch(left, right, field)` return owned arrays in one
computation RUN, with no partially owned output on failure. `evaluateBits`
accepts one MSB-first Boolean array per field, including fields wider than 64
bits. `maxTerminal` returns the exact maximum over assignments, accounting for
implicit zero regions even when every stored terminal is negative.

Scalar arithmetic identities, terminal calculations and memo hits return from
native admission without a computation RUN; a boundary collection may still
add an external RUN. Division remains strict, including implicit-zero holes.
The companion LLVF `parallel` branch uses its active-group
mask to make ECMP denominators nonzero outside the selected group.

The new interfaces and admission path passed the seven native Release cases,
ASan/UBSan numeric rechecks, and 43 Java tests. These checks do not establish
application speedup; LLVF carries separate fixed-input, paired-process reports.

## Benchmark and Validation Records

The [historical benchmark report](mtndd/results/upstream-20260422/README.md) contains four
April 22, 2026 N-Queens sweeps (168 observations) from the earlier MTPNDD prototype. The
[archived SRE report](mtndd/docs/optimization-results-summary.md) also measures that prototype.
These reports use one sample per configuration or incomplete machine metadata, and do not
measure the current shared core. N-Queens exercises Boolean constraints rather than arithmetic
throughput. Cross-version times should not be divided to infer speedup.

The [shared-core validation record](mtndd/results/validation-20261013-shared-core/README.md)
contains correctness and resource-management results. Application-level performance reports
are maintained separately in the companion LLVF project.

## Tests

```bash
ctest --test-dir native/build --output-on-failure
mvn -DskipTests=false -Dpndd.tests=true \
  -Dpndd.library="$PWD/native/build/libpnddjni.so" \
  -Dpmtndd.library="$PWD/native/build/libpmtnddjni.so" verify
```

For address and undefined-behavior checks:

```bash
cmake -S native -B native/build-asan -DCMAKE_BUILD_TYPE=Debug \
  -DPNDD_SANITIZE=ON -DPNDD_BUILD_JNI=OFF
cmake --build native/build-asan -j6
ASAN_OPTIONS=detect_leaks=1 UBSAN_OPTIONS=halt_on_error=1 \
  ctest --test-dir native/build-asan --output-on-failure
```

The October 13, 2026 validation passed **7 CTest cases in both Release and
ASan/UBSan builds**, and **41 Java tests** with no failures or skips. Coverage
includes all four backends and mixed fields, rational/double enumeration,
fused abstraction, dense import, cache collisions, stale handles, budget
failure recovery, and label GC during 32,768-element numeric batches.
Six-worker GC tests also passed with affinity restricted to two CPUs.
[Validation records and reproduction commands](mtndd/results/validation-20261013-shared-core/README.md)
are retained in the repository.

## Repository Structure

```text
native/
├── include/pmtndd.h             C multi-terminal API
├── src/pndd.c                  Shared compact manager and Boolean kernels
├── src/pmtndd_impl.inc          Numeric algorithms and terminals
├── src/pmtndd_jni.c             JNI binding
└── tests/test_pmtndd.c          Numeric and resource-management tests

src/main/java/org/ants/jndd/parallel/
├── ParallelMTNDD.java           Java engine API
├── MTNDDNode.java               Owned native root
└── ParallelNDDConfig.java       Shared runtime configuration

mtndd/                          Compatibility prototype and historical results
```

## Resources

- [NDD wiki](https://github.com/XJTU-NetVerify/NDD/wiki)
- [NDD paper, NSDI 2025](https://www.usenix.org/system/files/nsdi25-li-zechun.pdf)
- [Native build and configuration reference](native/README.md)
- [Migration notes](mtndd/docs/MIGRATION.md) and [capabilities/GC contracts](mtndd/docs/STATUS.md) (Chinese)
- [Dependency and source notices](native/NOTICE)

## License

Apache-2.0. See [LICENSE](LICENSE). Upstream copyright and license notices
are retained with the compatibility prototype and fetched dependencies.
