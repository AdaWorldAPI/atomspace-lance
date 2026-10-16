AtomSpace Lance StorageNode
===========================

**Status: planned. This repository holds no code yet.**

A `StorageNode` for the [OpenCog AtomSpace](https://github.com/opencog/atomspace)
that saves and restores AtomSpace contents through
[lance-graph](https://github.com/AdaWorldAPI/lance-graph), with
[Lance](https://lancedb.github.io/lance/) columnar files as the default
backing store. It follows the same pattern as
[atomspace-rocks](https://github.com/opencog/atomspace-rocks), and implements the
same `StorageNode` API, so an AtomSpace program can switch backends without
changing its own code.

```
 +---------------------+
 |      AtomSpace      |
 +-- StorageNode API --+
 |  LanceStorageNode   |   this repository (AGPL-3.0 + OpenCog linking exception)
 +---------------------+
 |   lance-graph-cog   |   lance-graph's API for OpenCog (Apache-2.0)
 +---------------------+
 | Lance / ordered KV  |
 +---------------------+
```

Each box is a library; calls go downwards. This repository stays a thin
adapter: the storage layout, the fold execution and the label codebook live in
`lance-graph-cog`, so they remain usable by any caller.

## Storage layout

atomspace-rocks keys each Atom by its s-expression text and joins Values and
incoming sets through string-prefixed keys, one lookup at a time. This backend
instead stores parallel tables addressed by one row index (`tableA[index]`,
`tableB[index]`), written sparsely and joined by a zipper merge on read. Any
store that can be read in index order supports that layout, including an
ordered key-value store such as RocksDB, so Lance is the default backend rather
than a requirement.

Planned mapping (not built, not measured):

| atomspace-rocks key | meaning | here |
|---|---|---|
| `n@` / `l@` / `a@` | Atom s-expression ↔ 64-bit id | deterministic address derived from the Atom's content |
| `h@` | hash → ids | not needed with a content-derived address |
| `i@sid:type` | incoming set by type | a sorted run per type, merged on read |
| `k@sid:key` | Values (truth values and others) | one table per key, at the Atom's row index |
| `d@` / `f@` / `o@` | frames (layered AtomSpaces) | versions and layers of the tables |

## StorageNode API to implement

As in `RocksStorage.h`: `open`, `close`, `getAtom`, `getLink`,
`fetchIncomingSet`, `fetchIncomingByType`, `storeAtom`, `removeAtom`,
`storeValue`, `updateValue`, `loadValue`, `loadType`, `loadAtomSpace`,
`storeAtomSpace`, `loadFrameDAG`, `storeFrameDAG`, `deleteFrame`, `barrier`.

## Validation

Before this backend is relied on, it is checked against atomspace-rocks on the
same AtomSpace: every Atom, Value and incoming set loaded through this backend
must equal what atomspace-rocks returns. Timings are reported per element.

## License

This repository is based on OpenCog and is licensed like it: AGPL-3.0 with the
OpenCog linking exception (see [LICENSE](LICENSE)). `lance-graph-cog` and the
rest of lance-graph are Apache-2.0 and contain no OpenCog code; code written
here is not moved into them.
