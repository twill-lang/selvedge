<p align="center">
  <img alt="twill" src="https://raw.githubusercontent.com/twill-lang/twill/main/assets/twill-mark.png" width="120">
</p>

<h1 align="center">selvedge</h1>

<p align="center">
  <b>Model serialisation and the model registry for <a href="https://github.com/twill-lang/twill">twill</a>.</b><br>
  Written in twill.
</p>

<p align="center">
  <img alt="selvedge" src="https://img.shields.io/badge/selvedge-v0.1.0-7FE3C4?style=flat-square&labelColor=12332C">
  <img alt="written in twill" src="https://img.shields.io/badge/written%20in-twill-D2F0E4?style=flat-square&labelColor=12332C">
  <img alt="archive format" src="https://img.shields.io/badge/archive%20format-v1-4FB79B?style=flat-square&labelColor=12332C">
  <img alt="status" src="https://img.shields.io/badge/tests-passing-4FB79B?style=flat-square&labelColor=12332C">
  <img alt="MIT" src="https://img.shields.io/badge/license-MIT-D2F0E4?style=flat-square&labelColor=12332C">
</p>

---

## It runs

`selvedge` is written in twill, in `.tw` files, using `mode systems`. That subset
did not exist when this library was written, so for a long time none of the code
here executed and this section said so. twill 1.6 is the release that closed it:
the 6 test suites under `tests/` pass, the example publishes and resolves a
model, and CI runs both against a released twill on every push rather than
gating on the prose in this file.

You need twill 1.7.0 or newer. Get one:

```bash
curl -fsSL -o twill https://github.com/twill-lang/twill/releases/download/v1.7.1/twill-v1.7.1-linux-amd64
chmod +x twill
```

The asset name is `twill-v1.7.1-<os>-<arch>`: `linux-amd64`, `linux-arm64`,
`darwin-amd64`, `darwin-arm64`, `windows-amd64.exe`.

The suite, from the repository root:

```
$ twill test tests
ok    tests/archive_test.tw
ok    tests/digest_test.tw
ok    tests/lineage_test.tw
ok    tests/registry_test.tw
ok    tests/rstr_test.tw
ok    tests/version_test.tw

6 file(s): 6 passed, 0 failed
```

And the example, which publishes a model, registers it and reads it back:

```
$ twill run examples/publish.tw
runs/blobs.params is absent, so these are untrained weights.
For the real thing, from a checkout of loom beside this one:
  cd ../loom && twill run examples/classifier.tw
  cp ../loom/examples/runs/blobs.params examples/runs/blobs.params
resolved blobs ^1.0.0 to blobs@1.2.0
  accuracy 0.9583
  digest   6eb801f966d173c7a9b6cf20313258cd4c74c807635415c89a796c3efae7694c
  derived from 1.1.0: 1 model(s)
```

That is the standalone run: with no trained parameters to hand, the example
generates a tree of the right shape, publishes it, and says on the way past that
the weights are not a model. Do what it tells you and the first three lines go
away and the digest changes, because the digest is over the weights:

```
$ cd ../loom && twill run examples/classifier.tw
$ cp ../loom/examples/runs/blobs.params examples/runs/blobs.params
$ twill run examples/publish.tw
resolved blobs ^1.0.0 to blobs@1.2.0
  accuracy 0.9583
  digest   1af561e0982a69df955691f179dedaea7216397b811afec2655acf3e0b97553e
  derived from 1.1.0: 1 model(s)
```

The `accuracy 0.9583` is recorded by hand in the example and is the number
loom's run prints for its best epoch. selvedge stores a metric and never checks
one, which is stated at `mf.record_metric` and is worth seeing here: the
standalone run above reports the same figure over weights that were never
trained.

`docs/needs.md` is still worth reading -- it is the list of what this library
asked the language for, and it now records which of those arrived and which are
still open.

## What selvedge is

Two things, and they are the same thing looked at twice.

**An archive format.** A finished model written to one file: parameters, the
input and output contract, a description of the architecture, the metrics it was
accepted on, the lineage that produced it, and a content hash over the bytes.

**A local registry.** An index of archives by name and version, with constraint
resolution, a second copy of every digest, and lineage links you can walk
backwards and forwards.

Every row below names the test or the example that runs it. A row that names
nothing is a row that claims nothing.

| Piece | State |
| --- | --- |
| Archive format v1, versioned and specified from the first commit | runs. `tests/archive_test.tw` writes one and reads it back with its metadata intact |
| Full compatibility with twill's `RSTR` tensor encoding | runs. `tests/rstr_test.tw`, and `a_twill_save_file_that_is_not_an_archive_is_told_apart` |
| SHA-256 content hashing and integrity verification | runs. `tests/digest_test.tw` pins four published vectors; `a_tampered_payload_is_caught` covers the rest |
| A refusal, by name, when a file is from a newer format | runs. `an_archive_from_a_newer_format_is_refused_with_both_numbers` |
| Register, list, resolve a version constraint, stage labels | runs. `tests/registry_test.tw`, twelve cases including the unsatisfiable ones |
| Lineage: data, code commit, hyperparameters, parent chain | runs. `tests/lineage_test.tw`, and the example prints its gaps |
| Contract checking: two versions sharing a major must agree on shapes | runs. `a_changed_contract_within_a_major_is_refused` |
| JSON metadata export, so a model card needs neither selvedge nor twill | runs. `the_metadata_is_json_a_stranger_can_read`, and the example writes `models/blobs-1.2.0.json` |
| Signing, or any claim about a model's origin | **not in v0.1.** Nothing here does it. See below |
| A remote registry, or fetching | **not in scope.** selvedge is local |
| Concurrent writers | **not handled.** One operator, one machine. See below |
| Anything running end to end | runs. `twill run examples/publish.tw`, output above |

## An archive is not a checkpoint

This is the boundary worth stating first, because selvedge sits next to
[loom](https://github.com/twill-lang/loom) and loom's `src/checkpoint.tw` also
writes a parameter tree to a file. They are different things and selvedge does
not duplicate, replace or read loom's.

**A checkpoint resumes a run.** It is allowed to be tied to that run and it is
useless without it. loom's carries adam's moments and its step count, the epoch,
the base seed, the batch size and the callbacks' patience counters, and it
refuses to restore into a different seed or batch size, correctly, because that
produces a run that is neither of the two runs. A checkpoint is a savepoint in a
process.

**An archive ships a result.** It carries no optimiser state, no seed, no epoch
and no batch size, because none of those mean anything to the person loading it.
It carries things a checkpoint has no reason to: the input and output contract,
the metrics, the lineage, and a hash. An archive is an artifact.

The consequence: a checkpoint is not an archive with fields missing, and an
archive is not a checkpoint with fields added. Converting one to the other is a
deliberate act, and the function is called `promote`, because that is the moment
someone decides a particular epoch is the model.

**selvedge does not depend on loom.** That looks like a missed reuse and it is
the opposite. Every consumer of an archive is on the inference side, where a
trainer is dead weight, and a dependency edge from the shipping format to the
training framework would put loom in the dependency tree of every serving
process in the ecosystem. `promote` takes a parameter tree and metadata, which
is exactly what a loom user has in their hand after `fit`.

## RSTR is a contract and selvedge does not change it

A selvedge archive **is an ordinary twill save file**. Same four magic bytes,
written by the same `save` builtin, and `load` on it with no selvedge present
returns a record you can read.

Those four bytes are a compatibility contract. They are not branding and they do
not track the language's name. Every file already written with `save` has to
keep loading, and `examples/model.bin` in the twill repository is one of those
files. The reference implementation states the same rule in
`internal/interp/serialize.go`; a second implementation that quietly disagreed
would be worse than no second implementation.

`src/rstr.tw` reads and writes that encoding in twill, transcribed from the Go.
[`docs/format.md`](docs/format.md) is the normative description, including the
four details that are easy to get wrong: rank and dimensions are `i32` while the
element count is `i64`, record key order is part of the bytes, a scalar is a
rank-0 tensor, and `'G'` is an opaque boosted-tree blob that selvedge carries
and does not interpret.

## The format is versioned from the first commit

Because a format that gains versioning later has already broken. The files
written before the version field existed are indistinguishable from the ones
written after, and the reader that has to tell them apart is guessing.

The rule, in full, is in [`docs/format.md`](docs/format.md). The short version:

- A reader accepts `format <= FORMAT` and refuses `format > FORMAT` **by name**,
  with both numbers in the message, because "unsupported format" without both
  does not tell anyone which end to upgrade.
- Adding a field does not bump the version. Readers ignore what they do not
  know; writers must not require what an older writer did not produce.
- Changing what a field means, removing one, reordering the top-level record, or
  changing what the digest covers does bump it. The order is in the bytes and
  the digest covers bytes.

## What the digest proves, and what it does not

`verify` recomputes the SHA-256 of the parameter payload and compares it against
the one the file declares.

**It catches** a truncated download, a corrupted disk, a file edited in place, a
parameter payload swapped for another.

**It does not catch** an attacker who rewrote the payload and the declared digest
together. A digest stored inside the thing it describes proves internal
consistency and nothing about origin. Saying so is the difference between
integrity and a claim of authenticity this library cannot make.

The registry's copy of the digest, in a separate file, is what puts something
behind the check: the index entry is a second copy an attacker would also have
to rewrite. That is a modest bar and it is described as modest.

Signing is not in v0.1 and is not simulated. A field called `signature` holding
something that is not a signature would be worse than an absent field.
[`docs/needs.md`](docs/needs.md) entry 10.

## Lineage is the part that earns the registry

A directory of files gives you storage, naming and listing. If a registry did
only that it would not be worth writing. Lineage answers the question anyone
actually asks six months later, which is never "where is the file" and always
"why is this one worse than the one we had in March".

The failure mode this is written against is decorative provenance: a `notes`
field, filled in by hand, unchecked, saying "trained on the usual data". Worse
than nothing, because it looks like an answer.

So every lineage field is one of two kinds, and each is marked in the source:

- **Identifying.** A value that pins something down and can be checked by
  someone holding the thing. A dataset digest, a row count, a commit sha, a
  hyperparameter, a parent's digest. selvedge compares these.
- **Descriptive.** Free text for a human. selvedge stores it, renders it, and
  never draws a conclusion from it.

`is_reproducible` returns true only when every identifying field is present.
`gaps` says why not, in specific terms, all of them at once. `diff` reports the
fields on which two lineages differ, which is the function the whole file exists
for: "why is this one worse" is answered with `commit` and `lr` rather than a
wall of text.

```
lineage gaps:
  no data digest: the training data cannot be identified
  the working tree was dirty: the named commit is not what ran
  no seed recorded: a repeat will be close and not identical
```

## Publishing a model

```rust
mode systems

import "twill_modules/selvedge/src/archive.tw" as arc
import "twill_modules/selvedge/src/manifest.tw" as mf
import "twill_modules/selvedge/src/registry.tw" as reg
import "twill_modules/selvedge/src/lineage.tw" as lin
import "twill_modules/selvedge/src/version.tw" as ver

let params = load("runs/blobs.params")

# The contract. -1 is the batch axis. "logits" is not decoration: it is the
# field that stops the next person applying softmax to something that had it.
let sig = mf.signature([-1, 4], [-1, 3], "logits")
let m = mf.manifest("blobs", ver.parse("1.2.0"), "mlp", sig)
m.architecture_detail = "dense 4->16, relu, dense 16->3"

# A metric without a dataset is a number without a unit, so both go in together.
m.met = mf.metrics("blobs/val.csv", 240)
mf.record_metric(m.met, "accuracy", 0.9583)

# What produced it. Every one of these is checkable by someone holding the
# thing it names.
# `data_digest_of_file` returns a Res, and that is the point: a misspelled path
# used to come back as an empty digest, which `gaps` then reported as "the
# training data cannot be identified", a true statement about a typo printed as
# a fact about the model.
let digest = match lin.data_digest_of_file("blobs/train.csv") {
  Ok(d) => d,
  Err(e) => abort("cannot digest the training data: " + e),
}
let prov = lin.from_data("blobs/train.csv", digest, 288)
lin.set_code(prov, "github.com/example/blobs", "1c9a4f0b7e2d5a6c8b3f1e0d9c8b7a6f5e4d3c2b", false)
lin.set_hyper(prov, "seed", "20260807")
lin.set_hyper(prov, "lr", "0.01")
lin.set_parent(prov, "blobs", "1.1.0", "3f79bb7b435b05321651daefd374cdc681dc06faa65e374e38337b88ca046dea")
m.prov = prov

let a = arc.promote(params, m)
let err = arc.write(a, "models/blobs-1.2.0.slv")

let r = match reg.load_index("models/index.json") {
  Ok(index) => index,
  Err(e) => abort("cannot read the registry index: " + e),
}
reg.register(r, a.man, "models/blobs-1.2.0.slv", "candidate")
reg.save_index(r)
```

And reading it back. `resolve` and `read` return a `Res` too, for the same
reason: a model that is not there and a model that is corrupt are different
answers and neither is a record with an empty field in it.

```rust
let entry = match reg.resolve(r, "blobs", "^1.0.0") {
  Ok(e) => e,
  Err(msg) => abort("selvedge: " + msg),
}
let integ = arc.verify(entry.path)
let loaded = match arc.read(entry.path) {
  Ok(back) => back,
  Err(msg) => abort("selvedge: " + msg),
}
```

```
resolved blobs ^1.0.0 to blobs@1.2.0
  accuracy 0.9583
  digest   1af561e0982a69df955691f179dedaea7216397b811afec2655acf3e0b97553e
  derived from 1.1.0: 1 model(s)
```

`examples/publish.tw` is that program, complete.

## The contract check

A caret constraint is only a safe thing to write if every version it can resolve
to takes the same input. Nothing else in a registry can promise that, so
selvedge checks it: two versions sharing a major must agree on their input
shape, output shape and output kind, and registering one that does not is
**refused**.

```
selvedge: blobs@1.3.0 declares input [_, 8] and output [_, 3], and blobs@1.0.0
shares its major version with input [_, 4] and output [_, 3]: a changed contract
is a new major version
```

Refused rather than warned about. A warning here is read once and then filtered.

A registered version also does not change. Re-registering an existing name and
version is refused; bump the patch, which is what patch is for. That immutability
is what makes a lineage chain a record of what happened rather than a claim
about what the files say today.

## Stated limits

- **One operator, one machine.** The index is rewritten whole on every change,
  with no locking, because twill has no locking. Two processes registering at
  once will lose one of the writes. twill 1.7 does have `rename`, so the
  narrower failure, a crash part-way through writing the index, could be made
  harmless by writing beside it and renaming over; `save_index` does not do
  that yet. `docs/needs.md` entry 12.
- **No fetching.** There is no remote registry, no upload, no download. selvedge
  indexes files that are already on your disk.
- **No signing.** See above.
- **Writing an archive writes it twice.** The digest covers the stored bytes of
  the parameter payload, and those bytes do not exist until `save` has produced
  them. selvedge cannot encode a parameter tree itself because the subset gives
  no way to walk one structurally, so it writes, reads back, hashes and writes
  again. `docs/needs.md` entry 4, whose narrow fix is `save_bytes`.
- **Hashing is interpreted and very slow.** Measured on twill 1.7.1, timing
  `dg.hash` with `mono_ns` on one machine: 262144 bytes in 19.5 s, and 65536
  bytes in 9.5 s and 6.2 s on two runs. That is about 10 to 15 kilobytes per
  second, not the megabyte per second this file used to claim, and the claim
  was never measured. A 100 MB archive is therefore not verifiable in
  practice, and that is a gate on selvedge being used for real models rather
  than a footnote. selvedge hashes on write and on an explicit `verify`, never
  on an ordinary read, and that is a design shaped around a missing builtin.
  `docs/needs.md` entry 7.

## Install

`mode systems` works. spool does not vendor selvedge for you yet, so until it
does the two ways in are a clone beside your project, or:

```
spool add selvedge https://github.com/twill-lang/selvedge
```

spool vendors into `twill_modules/`, and twill's import is a path, which is why
the import lines above are the long ones. **A path in twill, whether it is an
import or an argument to `read_file` or `save`, resolves against the directory
of the file that contains it, not against the working directory.** So
`twill run examples/publish.tw` reads and writes under `examples/` whatever
directory you invoke it from. That is twill's rule rather than selvedge's; see
spool's README.

## Repository layout

```
src/rstr.tw           twill's on-disk encoding, read and written in twill
src/digest.tw         SHA-256, and why it is here rather than in std
src/bytes_compat.tw   the one file that touches the subset's byte primitives
src/version.tw        model versions, and the two constraint forms
src/manifest.tw       Signature, Metrics, Manifest: an archive without weights
src/lineage.tw        identifying and descriptive, and the difference
src/archive.tw        the format, the compatibility rule, write, read, verify
src/registry.tw       register, list, resolve, ancestry, audit, the index file
src/card.tw           the manifest as JSON, and back
tests/                tests, named as sentences
examples/publish.tw   a complete publish and resolve
docs/format.md        the normative format description
docs/needs.md         what the language still has to provide
```

## Dependencies

twill, and nothing else. No loom, by the argument above. No spool. `std/json` is
the only standard-library module selvedge uses beyond the builtins.

## Contributing

The most useful contribution right now is not code. It is a correction to
[`docs/needs.md`](docs/needs.md): a feature listed there that the language
already has, a workaround that is worse than described, or a missing entry found
by reading the source. After that, the compatibility rule in
[`docs/format.md`](docs/format.md) is the part most worth arguing with, because
it is the part that cannot be changed later.

## License

MIT. See [LICENSE](LICENSE).
