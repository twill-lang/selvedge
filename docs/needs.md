# What selvedge needs from twill

selvedge is written in twill and does not run yet. This file is the reason: the
language and runtime features the source uses that twill does not provide today,
with the file and the function that needs each one, and what selvedge does in
the meantime.

It is a work queue for the language, not a complaint. Every entry was reached by
writing real code and hitting the wall, which is the only way a list like this
is worth anything. Where selvedge has a workaround the workaround is described,
because how ugly a workaround is measures how badly the feature is wanted.

The baseline is milestone 1 of `docs/self-hosting.md` in the twill repository:
`mode systems`, `I64` with bitwise operations and defined wrapping, `Str` with
length and byte indexing, `Arr[T]`, `Dict[Str, V]` with insertion-ordered
iteration, `struct`, and `read_file`. Everything below is on top of that.

Entries 1, 3, 6, 9, 13 and 14 are the same walls loom and spool hit. They are
restated here with selvedge's call sites rather than cross-referenced, because
a work queue that makes you read three repositories to find out what is blocking
is a work queue nobody reads.

## Blocking: selvedge cannot run at all without these

### 1. `mode systems` itself

**Used by:** every file
**Status:** designed in `docs/self-hosting.md`, not implemented.

Nothing else on this list matters until this does.

### 2. `Bytes`, and the byte-level surface

**Needs:** `bytes_new()`, `bytes_push(Bytes, I64)`, `bytes_to_str(Bytes)`, and
`Str` byte indexing with `len`
**Used by:** `src/bytes_compat.tw`, which is the only file that touches them, and
therefore `src/rstr.tw` and `src/digest.tw` through it
**Status:** specified in section 1.2 of the self-hosting design, not implemented.

This is the load-bearing one for this repository. A format library that cannot
get a byte out of a file is not a format library. Every function in
`src/bytes_compat.tw` is one line over a primitive, on purpose, so that when a
primitive is renamed one file changes.

`f64_bits(F64) -> I64` and `f64_from_bits(I64) -> F64` are needed alongside them,
by `src/rstr.tw` (`read_f64`, `put_f64`). They are NEEDS-84 in twill's own list.
There is no substitute: the format stores IEEE 754 bit patterns and going
through a decimal rendering loses the guarantee that a saved model reloads bit
for bit, which is the guarantee the format exists to make.

### 3. Multiple return values, or `Res[T, E]`

**Used by:** `src/archive.tw` (`write`, `read`, `verify`), `src/registry.tw`
(`register`, `resolve`, `load_index`), `src/card.tw` (`parse`), `src/rstr.tw`
(`read_header`, and the sticky `Reader.err`)
**Status:** `Res[T, E]` needs generics; multiple returns are not designed
anywhere.

Every fallible function in selvedge either returns a `Str` that is empty on
success, which is loom's and spool's convention and has their problem that the
compiler does not make anyone read it, or returns a struct declared for that one
call site. `Loaded`, `Parsed`, `Resolved`, `Integrity`, `DigestResult` and
`Lookup` are all that struct. Six of them. Not one is a type anyone wanted.

`src/rstr.tw` shows the cost most clearly. Its `Reader` carries a sticky `err`
field and every read is a no-op once it is set, because the alternative is
checking a return value at forty read sites in a byte parser. That pattern is
defensible and it is a hand-rolled error monad, which is what `Res` would be
if the language had it.

`std/json` already uses `Res` and `Opt`, and `src/card.tw` consumes them, so
this repository is written against a language where they exist in `std/` and not
in a user's file. That is the actual current state and it is the strongest
argument for the entry.

### 4. Structural reflection over a parameter tree

**Needs:** a way to ask a `Tree` whether it is a tensor, a list or a record, and
to walk it
**Used by:** `src/archive.tw` (`write`), which wants it and does not have it
**Status:** not designed anywhere. `map_leaves` and `zip_leaves` walk a tree and
neither exposes its shape.

selvedge would like to encode a parameter tree to bytes so it can hash it before
writing. It cannot, so `write` writes the file, reads it back, hashes the
`params` span and writes again. Two full writes of every model.

For a small model that is invisible. For a 400 MB model it is 400 MB of extra
IO on every publish, and publishing is not a hot path, which is the only reason
this entry is not higher up the list.

The narrow fix is `save_bytes(value) -> Bytes`, the encoder the `save` builtin
already has, exposed without the file. That is a smaller ask than reflection and
it would close this entry completely. Reflection would also let selvedge report
"this archive holds 4 tensors totalling 12,032 parameters" in a model card,
which it currently cannot.

### 5. A clock

**Needs:** a wall clock returning something renderable, and separately a
monotonic `mono_ms() -> I64`
**Used by:** `src/lineage.tw` (`trained_at`, which the caller must fill in by
hand)
**Status:** not in the language. loom entry 16 and bobbin's first entry are the
same requirement.

`trained_at` is a free-text field the caller sets, which means in practice it
will be empty. A timestamp nobody has to type is a timestamp that is there when
it is needed. It is marked DESCRIPTIVE in `src/lineage.tw` and never compared,
so nothing is wrong today; it is just less useful than it should be.

Not blocking. Listed because it is the same primitive three repositories want
and it should be satisfied once.

### 6. `Dict[Str, V]` where `V` is not a scalar, with guaranteed ordering

**Would improve:** `src/manifest.tw` (`Metrics`), `src/lineage.tw`
(hyperparameters)
**Status:** milestone 1 gives `Dict[Str, V]` with insertion-ordered iteration;
whether `V` may be a struct is not stated.

Both are parallel `Arr` pairs where they want to be one dictionary. The parallel
arrays can go out of step and only a convention keeps them together.

The ordering guarantee is the part that matters more, and it has to be a
guarantee rather than an implementation detail. Both of these render into the
model card, and the card is stored in the archive. A dictionary whose iteration
order changed between twill versions would change the metadata of a model nobody
retrained, and the digest is over the parameters rather than the metadata only
because it was not safe to hash something with that property.

## Blocking: features the source assumes exist

### 7. `std/hash`, and a SHA-256 that is not interpreted

**Needs:** SHA-256 as a `std/` module, and a builtin for large inputs
**Used by:** `src/digest.tw`, which is the whole file
**Status:** none. spool has `src/sha256.tw`; this is the second copy.

Two SHA-256 implementations in one ecosystem drift, and the drift is invisible
until two tools disagree about a hash and neither is obviously wrong. Neither
repository can import the other's, because twill resolves a non-`std/` import as
a path relative to the importing file, so only `std/` modules are reachable from
an installed package. The fix is `std/hash`, at which point `src/digest.tw` is
deleted.

The performance half is separate and is real. This runs the compression function
in a tree-walking interpreter, one 32-bit word at a time, on a 64-bit value
type: on the order of a megabyte per second. Hashing a 12 kB parameter tree is
free. Hashing a 400 MB archive at that rate is several minutes, which makes
`verify` unusable on exactly the models where verifying matters most. selvedge
hashes on write and on an explicit `verify`, never on an ordinary `read`,
specifically because of this, and that is a design shaped around a missing
builtin rather than around what the design wanted.

### 8. Bitwise operators, with their spelling fixed

**Needs:** `and or xor shl shr not` on I64, and a decision about whether they
are infix or calls
**Used by:** `src/digest.tw` (every line of the compression function),
`src/rstr.tw` (`read_i32`, `read_i64`, `put_i32`, `put_i64`, `ushr`),
`src/bytes_compat.tw` (`push_hex_byte`)
**Status:** section 1.2 promises them and does not say how they are spelled, and
the ecosystem has already split.

twill's own `src/float.tw` writes `shl(a, b)` and `and(a, b)`. spool's
`src/sha256.tw` writes `a shl b` and `a and b`, infix. loom writes calls. Both
spellings appear in code that is meant to be the reference for what the subset
is. selvedge writes calls, following twill's own source, and one of the three
repositories is going to need editing whichever way this lands.

The substantive part underneath the bikeshed: `shr` must be specified. twill's
`src/float.tw` carries a hand-written `ushr` because `shr` is arithmetic, and
`src/rstr.tw` carries the same helper for the same reason. SHA-256's message
schedule is wrong with an arithmetic shift, and so is every negative integer
written by `put_i64`. This is a place where the wrong answer is silent.

### 9. A `Tree` type

**Needs:** a spelling for "a tensor, or a list or record nesting tensors"
**Used by:** `src/archive.tw` (`Archive.params`, `promote`, `to_record`)
**Status:** the concept exists at runtime and has no name in the type language.
loom entry 2 is the same wall.

selvedge writes `Tree` and assumes it will be the name. Systems mode makes
annotations mandatory, so `Archive` is currently undeclarable.

### 10. Signing, or the primitives for it

**Needs:** an asymmetric signature primitive, or a documented decision that
twill will not have one
**Used by:** nothing, which is the point
**Status:** not in the language and not designed.

`src/archive.tw` `verify` is careful to say what it does not prove: a digest
stored inside the thing it describes says nothing about origin. A registry of
models is exactly the sort of thing that eventually wants a signature, and
selvedge deliberately does not simulate one, because a field called `signature`
holding something that is not a signature is worse than an absent field.

This entry exists so that the absence is recorded as a decision rather than an
oversight. It is not v0.1 work.

### 11. A module that can hold a type two other modules share

**Would improve:** `src/manifest.tw`, which exists partly for this
**Status:** twill's import is a path and there is no forward declaration.

`src/archive.tw` writes manifests and `src/card.tw` renders them, so whichever
one owned the type, the other would import it and the cycle would be real.
`src/manifest.tw` breaks it. It has an independent justification, which is that a
registry holds manifests and no weights, and that justification is why the split
is defensible rather than a workaround. It would still be nicer not to have been
pushed.

## Not blocking, but the source is worse without them

### 12. File locking, or an atomic rename

**Needs:** `rename(from, to)` at minimum
**Used by:** `src/registry.tw` (`save_index`)
**Status:** `write_file` exists in the design; nothing else does.

The index is rewritten whole on every change, with no lock, so two processes
registering at once lose one of the writes. Worse, a crash during the write
leaves a truncated index rather than the previous one.

An atomic rename fixes the second problem completely and is a much smaller ask
than locking: write to a temporary path, rename over the target. The first
problem stays, and stays acceptable, because this is a single-operator local
registry and says so in the README.

### 13. A generic sort, or a comparison-function parameter

**Would improve:** `src/registry.tw` (`versions`)
**Status:** no generic sort; function parameters are undesigned.

`versions` is an insertion sort by version, written out by hand. loom has one,
spool has four. This is the sixth in the ecosystem.

### 14. A test runner

**Would improve:** `tests/`
**Status:** none. `tests/harness.tw` is a hand-rolled counter and `report` calls
`exit(1)`.

Every test file is a program that has to be run individually. `tests/harness.tw`
is now the third identical copy of the same file across three repositories. A
`twill test` that collected `*_test.tw`, ran each in a fresh interpreter and
reported once would delete all three.

### 15. Temporary files, and cleaning up after a test

**Would improve:** `tests/archive_test.tw`, `tests/registry_test.tw`
**Status:** there is no `remove_file` and no temporary directory.

The archive and registry tests write into the working directory and leave the
files there. They are named `tmp_*` and `.gitignore` covers them, which is a
convention doing a runtime's job. A test that cannot clean up is a test that
fails differently on a second run if it ever leaves state behind, and the only
reason these do not is that every one of them overwrites rather than appends.
