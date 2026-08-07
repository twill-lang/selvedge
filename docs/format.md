# The selvedge archive format, version 1

This is the normative description. `src/archive.tw` implements it and
`src/rstr.tw` implements the encoding underneath it.

## The encoding is twill's, unchanged

A selvedge archive **is an ordinary twill save file**. It begins with the four
bytes `RSTR`, it is written by the `save` builtin, and `load` on it returns a
record you can read with no selvedge present.

Those four magic bytes are a compatibility contract. selvedge does not change
them, does not add a second accepted magic, and does not track the language's
name. The reference implementation states the same rule in
`internal/interp/serialize.go`, and a second implementation that quietly
disagreed with the first would be worse than no second implementation.

The encoding, transcribed from that file:

```
file    := "RSTR" version:u8 value
value   := 'T' rank:i32 dim:i32 * rank  n:i64  f64 * n      a tensor
         | 'L' count:i32 value * count                      a list
         | 'R' count:i32 (len:i32 key:bytes value) * count   a record
         | 'B' 0|1                                          a bool
         | 'S' len:i32 bytes                                 a string
         | 'U'                                               unit
         | 'G' len:i32 bytes                                 a boosted-tree model
```

Little-endian throughout. Floats are stored as IEEE 754 bit patterns, so a value
round-trips bit for bit.

Four details that are load-bearing and easy to get wrong:

1. A tensor's rank and dimensions are `i32`; its element count is `i64`. The
   widths are not uniform. A reader that treats the count as `i32` reads four
   bytes of the first float as a length.
2. A record's keys are written in insertion order, and the order is part of the
   bytes. Sorting on write produces a file that loads to an equal value and
   hashes differently.
3. A scalar is a rank-0 tensor: `'T'`, rank 0, no dimensions, `n = 1`, one
   float. There is no separate number tag.
4. `'G'` is a boosted-tree model, an opaque blob. selvedge carries it, measures
   it and hashes it. It does not interpret it.

## The archive value

The top-level value is a record with these fields **in this order**:

| Field | Kind | Contents |
| --- | --- | --- |
| `selvedge` | string | the literal `"selvedge"` |
| `format` | number | the archive format version, currently 1 |
| `meta` | string | the manifest, rendered as JSON by `src/card.tw` |
| `digest` | string | SHA-256 of the `params` field's exact stored bytes |
| `params` | tree | the parameters, whatever shape the model has |

`params` is last because it is the only large field, so every other field is
reachable by skipping four small values rather than a tensor.

The metadata is one JSON string rather than a nest of records. The argument for
that, since it is the decision most worth pushing on: the metadata has to be
readable by things that are not twill. Storing it structurally and rendering
JSON on demand would let the rendering drift from what was stored, and then the
digest covers something nobody else can reproduce. One representation.

## The digest

`digest` is the SHA-256, lowercase hex, of the bytes the `params` value occupies
in the file. Over the stored bytes, not over a re-encoding: two encoders that
agree on every value can still disagree on a byte, and a digest that depends on
which encoder produced it fails on files nobody corrupted.

The digest's construction is part of the format. Changing what is hashed is a
format change even if every field keeps its name, because an old reader would
otherwise compute a mismatch and report corruption on a sound file.

**What the digest proves.** That the file's parameters are the ones its own
header describes. That is integrity, and it catches a truncated download, a
corrupted disk, a file edited in place, and a payload swapped for another.

**What it does not prove.** Anything about origin. A digest stored inside the
thing it describes is internally consistent by construction for anyone willing
to rewrite both. The registry's copy of the digest, in a separate file, is what
turns this into a check with something behind it. Signing is not in v0.1 and is
not simulated.

## The compatibility rule

The format is versioned from the first commit. A format that gains versioning
later has already broken: the files written before the version field existed are
indistinguishable from the ones written after, and the reader that has to tell
them apart is guessing.

**Readers.** Accept `format <= FORMAT`. Refuse `format > FORMAT` by name, with
both numbers in the message, because "unsupported format" without both does not
tell anyone which end to upgrade. Ignore fields you do not know.

**Writers.** Never require a field a previous version did not write.

**What does not bump the version.** Adding a field to the top-level record after
the existing ones, or adding a key to the JSON metadata. These are the changes a
reader can absorb, which is why `find_field` in `src/rstr.tw` treats a missing
key as absence rather than as an error.

**What bumps the version.** Changing what an existing field means. Removing a
field. Reordering the top-level record, because the order is in the bytes and
the digest covers bytes. Changing what the digest covers, or the hash function.

**What a bump costs.** A major bump means old readers refuse new files, by
design and with a clear message. New readers keep reading old files. There is no
mechanism for a new reader to refuse an old file and there will not be one:
that is what the archive's own metrics and lineage are for.

## The JSON metadata

Rendered by `src/card.tw`, keys in a fixed order. Fixed because it is stored, and
because a card that reorders itself produces a diff every time it is
regenerated. `std/json` preserves insertion order, which is what makes that
possible.

```json
{
  "selvedge_format": 1,
  "name": "blobs",
  "version": "1.2.0",
  "description": "three-class blob classifier",
  "architecture": "mlp",
  "architecture_detail": "dense 4->16, relu, dense 16->3",
  "signature": {
    "input": [null, 4],
    "output": [null, 3],
    "output_kind": "logits"
  },
  "metrics": {
    "dataset": "blobs/val.csv",
    "rows": 240,
    "values": { "accuracy": 0.9583, "loss": 0.118 }
  },
  "lineage": {
    "data_digest": "9f86d081...",
    "data_source": "blobs/train.csv",
    "data_rows": 960,
    "code_repo": "github.com/example/blobs",
    "code_commit": "1c9a4f0b...",
    "code_dirty": false,
    "hyperparameters": { "seed": "20260807", "lr": "0.01" },
    "parent_name": "blobs",
    "parent_version": "1.1.0",
    "parent_digest": "3f79bb7b...",
    "trained_at": "",
    "note": "",
    "lineage_digest": "...",
    "reproducible": true,
    "gaps": []
  },
  "params_bytes": 502,
  "note": ""
}
```

A free dimension is `null`, not `-1`. A card is read by things that are not
selvedge, and `-1` in a shape array is a value someone will multiply.

`lineage_digest`, `reproducible` and `gaps` are derived and written out rather
than left for a reader to recompute, so a card generator in another language
does not have to reimplement `src/lineage.tw`'s rules and get them slightly
different.

## Version numbers on models

Different from a package version, and the difference is the point.

| Component | Meaning |
| --- | --- |
| major | the input or output contract changed |
| minor | retrained, same contract, metrics moved |
| patch | the artifact changed and the weights did not |

The registry enforces the major rule and only the major rule: two versions
sharing a major must agree on `signature_digest`, which is a hash of the
declared input shape, output shape and output kind. Registering one that does
not is refused. That is what makes `^1.0.0` a safe thing to write.

The other two are conventions selvedge cannot check, and it does not pretend to.
