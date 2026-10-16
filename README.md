# MTNDD

MTNDD (Multi-Terminal Network Decision Diagram) is a field-oriented decision diagram whose
terminal nodes may hold numeric values instead of only Boolean `false` and `true`. It is useful
for symbolic functions such as route weights, traffic or link load, probabilities, costs, and
integer matrices, while retaining NDD's field-level structure.

This repository provides Java and parallel native implementations of MTNDD, extracted from
[NDD](https://github.com/XJTU-NetVerify/NDD) with their commit history preserved. The Java core
originated in [`llvf`](https://github.com/XJTU-NetVerify/llvf). Both implementations support
BDD, complemented-edge BDD (BCDD), ZDD, BITSET, and mixed label backends.

## Choose a Branch

| Branch | Implementation | Numeric domains | Java entry point |
| --- | --- | --- | --- |
| [`main`](https://github.com/XJTU-NetVerify/MTNDD/tree/main) (this branch) | Java | Rational, double, interval | `org.ants.jndd.diagram.NDD` |
| [`parallel`](https://github.com/XJTU-NetVerify/MTNDD/tree/parallel) | C11 + JNI, Lace worker pool | Checked rational, finite double | `org.ants.jndd.parallel.ParallelMTNDD` |

Use `main` for the Java implementation and its richer numeric modes; use `parallel` for
native parallel operations and an instance API with explicit root ownership. The APIs and
numeric policies differ; switching branches requires adapting application code. Boolean-only
NDD development remains in the [NDD repository](https://github.com/XJTU-NetVerify/NDD).

The source branches were `NDD/mtndd` and `NDD/parallel-MTNDD`, respectively. Historical package
names and the Maven artifact `org.ants:ndd:1.0.1` are retained for compatibility.

## Build

Requirements:

- JDK 11 or newer (Java sources target Java 8 bytecode)
- Maven 3.x

```bash
git clone https://github.com/XJTU-NetVerify/MTNDD.git
cd MTNDD
mvn -DskipTests package
```

The build produces `target/ndd-1.0.1.jar` and a jar with dependencies. Run the core regression
test with:

```bash
mvn -DskipTests=false -Dtest=org.ants.jndd.diagram.NDDMixedBackendTest test
```

## Minimal Example

The historical Java package name `org.ants.jndd` is retained for source compatibility.

```java
import org.ants.jndd.diagram.NDD;

public class MTNDDExample {
    public static void main(String[] args) {
        NDD.initNDD(20_000, 2_000, 10_000, 1_000);
        int header = NDD.declareField(2, NDD.LabelMode.BDD);
        int links = NDD.declareField(2, NDD.LabelMode.ZDD);
        NDD.generateFields();

        NDD selected = NDD.getVar(header, 0).and(NDD.getVar(links, 0)).withRef();
        NDD weight = NDD.createTerminal(10).withRef();
        NDD weighted = selected.times(weight).withRef();
        try {
            System.out.println(NDD.satCount(selected));          // 4.0
            System.out.println(NDD.satCount(weighted, 0, 10));   // 4.0
        } finally {
            weighted.recursiveDeref();
            weight.recursiveDeref();
            selected.recursiveDeref();
        }
    }
}
```

Save the example as `MTNDDExample.java`, then compile and run it:

```bash
javac -cp target/ndd-1.0.1-jar-with-dependencies.jar MTNDDExample.java
java -cp .:target/ndd-1.0.1-jar-with-dependencies.jar MTNDDExample
```

Each two-bit field has four assignments. Selecting one bit in each field leaves four complete
assignments, with value `10` in `weighted` and `0` elsewhere. In this Java API, `satCount(f)`
counts assignments whose value matches `1`; `satCount(f, 0, target)` counts matches to the
specified integer terminal using the implementation's numeric tolerance. It does not sum weights.
The parallel API instead counts nonzero assignments. Use `sumAbstract` for weighted sums.

The required initialization order is:

1. call `initNDD(...)`
2. optionally configure per-backend capacities
3. declare every field and its label mode
4. call `generateFields()` once
5. construct and manipulate diagrams

`declareField(width)` uses BDD labels by default. Use the returned field IDs in subsequent calls. Fields using the same mode share one label
engine and a right-aligned variable layout; BDD, BCDD, and ZDD fields use separate engines.

## Backend Configuration

Each backend can be sized before its first field is declared:

```java
NDD.configureBackendCapacity(NDD.LabelMode.BDD, 2_000_000, 200_000);
NDD.configureBackendCapacity(NDD.LabelMode.ZDD, 4_000_000, 400_000);
NDD.configureBackendCapacity(NDD.LabelMode.COMPLEMENTED_BDD, 1_000_000, 100_000);
```

The modes have the same Boolean bit-vector field semantics:

| Mode | Edge-label representation | Main characteristic |
| --- | --- | --- |
| `BDD` | reduced ordered BDD | general-purpose default |
| `COMPLEMENTED_BDD` | BDD with complemented handles | constant-time label negation |
| `ZDD` | family of sets of true-bit variables | explicit per-field universe |
| `BITSET` | canonical two-word truth table | O(1) set operations for fields up to 7 bits |

Backend selection changes representation, not the logical domain of a field. A width-`w` field
always denotes `2^w` bit-vector assignments.

## Core API

MTNDD nodes are exposed as lightweight `NDD` wrappers over canonical integer node IDs.

| Purpose | Main methods |
| --- | --- |
| Initialization | `initNDD`, `configureBackendCapacity`, `declareField`, `generateFields` |
| Variables | `getVar`, `getNotVar`, `encodePrefix`, `encodePrefixs` |
| Boolean/set operations | `and`, `or`, `not`/`cmpl`, `diff`, `imp`, `exist` |
| Multi-terminal arithmetic | `add`/`plus`, `sub`/`minus`, `mul`/`times`, `div`/`divide`, `sumAbstract`, `multiplySumAbstract` |
| Terminals | `getFalse`, `getTrue`, `createTerminal(int/double/Rational)` |
| Inspection | `evaluate`, `satCount`, `toArray`, `print`, `printDot` |
| Lifetime | `ref`, `withRef`, `deref`, `recursiveDeref`, `gc`, `gcLabelEngines` |

Results are not permanently rooted by default. Protect values that must survive later allocation
or explicit collection with `withRef()`/`ref()`, then release them with
`recursiveDeref()`/`deref()`.

## Repository Structure

```text
src/main/java/
├── org/ants/jndd/
│   ├── diagram/       MTNDD operations, backend dispatch, manager and terminals
│   ├── nodetable/     array-backed canonical node/edge storage and MTNDD GC
│   ├── bdd/           primitive complemented-edge BDD implementation
│   ├── cache/         legacy generic operation-cache utility
│   └── utils/         rational terminals and BDD decomposition helpers
├── jdd/               bundled BDD/ZDD label engines
├── application/matrix symbolic matrix-multiplication example
├── application/wan/   retained network-verification research artifacts
└── org/ants/javandd/  legacy JavaBDD-compatible NDD facade

src/test/java/org/ants/jndd/
└── diagram/           mixed-backend and multi-terminal regression tests
```

The primary implementation is `org.ants.jndd.diagram.NDD`. `NDDManager` is a small object-style
facade. The WAN paths and atomized prototypes are retained for research reference and are excluded
from the default Maven compilation where noted in `pom.xml`.

## Numeric Semantics

`TerminalMode.RATIONAL` is the default and stores normalized `long` numerators and denominators.
It is not arbitrary-precision arithmetic: operations use wider intermediates where needed, and
fractions that exceed the exact representation may fall back to decimal rounding at
`Rational.getDefaultScale()` (configurable with `Rational.setDefaultScale(...)`). Values still
outside the storage range can fail. This policy differs from the parallel core's checked rational
arithmetic, which reports overflow.

`TerminalMode.DOUBLE` supports optional clustering and near-zero pruning.
`TerminalMode.INTERVAL` supports outward-rounded bounds and configurable enclosure policies.
These modes and their approximation settings must be selected during initialization; the
parallel branch does not implement interval terminals or approximate clustering.

The Java core has static global state. Initialize it before constructing diagrams; reinitializing
replaces that state. Protect every intermediate that must survive later allocations or collection.
See the [mixed-backend regression tests](src/test/java/org/ants/jndd/diagram/NDDMixedBackendTest.java)
for numeric modes, abstraction, and collection examples.

## Implementation Notes

- MTNDD nodes and physical edges are stored in primitive arrays and canonicalized by a unique
  table.
- Recursive operations collect edges in a shared stack, then sort, merge equal targets, and create
  one canonical node at a safe point.
- Dense importers can use `NDD.BulkBuilder` to avoid allocating a wrapper per input terminal.
- Unreferenced numeric terminals participate in safe-point garbage collection; edge and terminal
  indices are compacted without changing live handles.
- Each field records the backend that owns its edge-label handles; label operations are dispatched
  through that backend and handles are never mixed between engines.
- Rational terminal values are canonicalized, so equal numeric leaves share a terminal node.
- Opt-in primitive-double and interval terminal modes avoid per-terminal `Rational` objects.
  Double terminals support configurable mantissa/grid clustering, tracked approximation bounds,
  and explicit near-zero pruning. Interval terminals canonicalize outward-rounded lower/upper
  pairs and provide compact or tight arithmetic-enclosure policies. An experimental nominal-bucket
  plus multiplicative-radius-class encoding is also available for representation studies.
  Rational remains the default.
- MTNDD and label engines expose live/created node counts plus collection and growth metrics for
  performance diagnosis.

See the [MTNDD wiki page](https://github.com/XJTU-NetVerify/NDD/wiki/MTNDD) for the algorithm and
interface model. The [original NDD paper (NSDI 2025)](https://www.usenix.org/system/files/nsdi25-li-zechun.pdf)
and NDD-specific design material remain available through the original project. The wiki link
above intentionally points to NDD; repository extraction does not copy GitHub wiki pages.

## License

Apache-2.0. See [`LICENSE`](LICENSE).
