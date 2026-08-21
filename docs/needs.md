# What selvedge needs from twill

selvedge is written in twill and it runs: `twill test tests` passes six suites
and `twill run examples/publish.tw` publishes a model and reads it back. This
file is no longer the reason it does not run. It is the record of what this
library asked the language for, with the file and the function that needed each
one, what selvedge did in the meantime, and, for the ones that have since
arrived, whether selvedge has taken them up. An entry the language delivered and
selvedge has not wired up says exactly that.

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

## Was blocking: selvedge could not run at all without these

### 1. `mode systems` itself

**Used by:** every file
**Status:** DELIVERED in twill 1.6. Closed.

Nothing else on this list mattered until this did.

### 2. `Bytes`, and the byte-level surface

**Needs:** `bytes_new()`, `bytes_push(Bytes, I64)`, `bytes_to_str(Bytes)`, and
`Str` byte indexing with `len`
**Used by:** `src/bytes_compat.tw`, which is the only file that touches them, and
therefore `src/rstr.tw` and `src/digest.tw` through it
**Status:** DELIVERED. `bytes_new`, `bytes_push`, `bytes_to_str`, `f64_bits`
and `f64_from_bits` all exist in twill 1.7.1, and `tests/rstr_test.tw` round
trips a save file through them. Closed.

This was the load-bearing one for this repository. A format library that cannot
get a byte out of a file is not a format library. Every function in
`src/bytes_compat.tw` is one line over a primitive, on purpose, so that when a
primitive is renamed one file changes.

`f64_bits(F64) -> I64` and `f64_from_bits(I64) -> F64` are needed alongside them,
by `src/rstr.tw` (`read_f64`, `put_f64`). They are NEEDS-84 in twill's own list.
There is no substitute: the format stores IEEE 754 bit patterns and going
through a decimal rendering loses the guarantee that a saved model reloads bit
for bit, which is the guarantee the format exists to make.

### 3. Multiple return values, or `Res[T, E]`

**Used by:** `src/archive.tw` (`read`, `digest_of_params`), `src/registry.tw`
(`resolve`, `resolve_stage`, `load_index`), `src/card.tw` (`parse`, `parse_doc`),
`src/rstr.tw` (`find_field`)
**Status:** done for `Res[T, E]` (2026-08, on twill 1.7); multiple return values
are still not designed anywhere and are a separate want.

`Res[T, E]`, `Opt[T]` and postfix `?` landed in twill 1.6 as checked types, and
selvedge has now moved onto them. Every one-off result struct this entry named
is gone:

| was | is |
| --- | --- |
| `Lookup { span, found }` | `Opt[Span]` |
| `Parsed { man, err }` | `Res[mf.Manifest, Str]` |
| `Loaded { arc, err }` | `Res[Archive, Str]` |
| `Resolved { entry, found, err }` | `Res[Entry, Str]` |
| `DigestResult { digest, size, err }` | `Res[Digest, Str]` |
| `Registry.err` | `load_index -> Res[Registry, Str]` |

`Resolved` is the one worth naming: it carried a `found` Bool *and* an `err`
Str, two fields encoding one bit, since every way of failing to resolve
already carried a message. `Registry.err` is the other -- a load failure stored
on the registry value and kept there forever, when what failed was the load.

**Two things deliberately did not move.**

`Integrity { status, declared, computed, detail }` stays. This entry listed it
with the others and that was wrong: it is not an error channel. Its `status` is
an enum of four outcomes and a caller wants all four fields whatever the outcome
is, so it is a report, and a report is a type someone wanted. `Digest` is the
same shape for the same reason -- two values wanted together at both call sites
-- and what it lost was only the `err` field that made it a failure channel too.

`Reader.err` stays sticky, and the reason is now a measurement rather than a
limitation. A byte parser reads a length, then a tag, then a span, using each
value in arithmetic immediately; `?` at every one of forty read sites puts a
failure check between every pair of bytes and buries what the parser is doing.
One check at the end is what a byte reader looks like in a language that has
both options. The comment above `struct Reader` records that as a choice.

**What the compiler now enforces, which is the whole point.** A caller of
`resolve` cannot reach the entry without handling the failure, because there is
no entry to reach until the `match` takes the `Ok` arm. Under `Resolved` the
same caller could read `.entry` and get an empty manifest, and nothing said so.

`std/json` already used `Res` and `Opt` and `src/card.tw` already consumed them,
so the argument this entry ended on -- that selvedge was written against a
language where they existed in `std/` and not in a user's file -- no longer
applies in either direction.

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
**Status:** DELIVERED in twill 1.7, and selvedge has not taken it up.
`clock_now_ms()` is the wall clock and `mono_ns()` the monotonic one; both were
checked against the 1.7.1 binary. `src/lineage.tw` still makes the caller type
`trained_at` by hand, so what is open here is selvedge's work and not twill's.

Setting it automatically is not a one-liner, because a timestamp selvedge writes
is DESCRIPTIVE and a timestamp the caller supplies may be IDENTIFYING, and this
file's whole argument is that the two are never mixed. `clock_now_ms` returns
milliseconds since the epoch and nothing here renders that as a date yet either.

`trained_at` is a free-text field the caller sets, which means in practice it
will be empty. A timestamp nobody has to type is a timestamp that is there when
it is needed. It is marked DESCRIPTIVE in `src/lineage.tw` and never compared,
so nothing is wrong today; it is just less useful than it should be.

Not blocking. Listed because it is the same primitive three repositories want
and it should be satisfied once.

### 6. `Dict[Str, V]` where `V` is not a scalar, with guaranteed ordering

**Would improve:** `src/manifest.tw` (`Metrics`), `src/lineage.tw`
(hyperparameters)
**Status:** DELIVERED for the type, and selvedge has not taken it up. A
`Dict[Str, V]` holds a struct in twill 1.6, which loom's `MeterSet` relies on.
`src/manifest.tw` and `src/lineage.tw` are still parallel `Arr` pairs.

The ordering guarantee below is the half that is still a language question, and
it is the half that decides whether selvedge should move: an iteration order
that changed between twill versions would change the metadata of a model nobody
retrained. Insertion-ordered iteration is what milestone 1 describes; it is not
written down as a compatibility promise anywhere selvedge can point at.

Both are parallel `Arr` pairs where they want to be one dictionary. The parallel
arrays can go out of step and only a convention keeps them together.

The ordering guarantee is the part that matters more, and it has to be a
guarantee rather than an implementation detail. Both of these render into the
model card, and the card is stored in the archive. A dictionary whose iteration
order changed between twill versions would change the metadata of a model nobody
retrained, and the digest is over the parameters rather than the metadata only
because it was not safe to hash something with that property.

## Was blocking: features the source assumed exist

### 7. `std/hash`, and a SHA-256 that is not interpreted

**Needs:** SHA-256 as a `std/` module, and a builtin for large inputs
**Used by:** `src/digest.tw`, which is the whole file
**Status:** the `std/` half is DELIVERED and selvedge has not taken it up. The
builtin half is still open and matters more than this entry said.

`std/hash` exists in twill 1.7.1 and is SHA-256 written over I64, the same
algorithm `src/digest.tw` wraps. The drift argument below is therefore
selvedge's to act on now: `src/digest.tw` should become a wrapper over
`std/hash` or disappear, and the work in the way is checking that the two agree
on every vector in `tests/digest_test.tw`. That has not been done, so this
repository still carries the second copy it complains about.

The performance half is separate, is real, and was understated here by two
orders of magnitude. Timed with `mono_ns` on twill 1.7.1, `dg.hash` did 262144
bytes in 19.5 s, and 65536 bytes in 9.5 s and 6.2 s on two runs: about 10 to 15
kilobytes per second, not the megabyte per second this entry and the README both
claimed without measuring. At that rate a 12 kB parameter tree is about a
second, a 1 MB archive is a minute and a half, and a 400 MB archive is weeks. A
builtin is not a nice-to-have here: `verify` is unusable above a few megabytes,
which is every model anyone would want to verify. selvedge hashes on write and
on an explicit `verify`, never on an ordinary `read`, specifically because of
this.

### 8. Bitwise operators, with their spelling fixed

**Needs:** `and or xor shl shr not` on I64, and a decision about whether they
are infix or calls
**Used by:** `src/digest.tw` (every line of the compression function),
`src/rstr.tw` (`read_i32`, `read_i64`, `put_i32`, `put_i64`, `ushr`),
`src/bytes_compat.tw` (`push_hex_byte`)
**Status:** DELIVERED as calls, and the substantive half is settled. twill
1.7.1 has `band`, `bor`, `xor`, `shl`, `shr` and `bnot` as builtins, and `shr`
is an arithmetic shift, which is what `src/rstr.tw` and `src/digest.tw` already
assume. selvedge writes calls and does not need editing.

The bikeshed below is not settled and is now cosmetic: the infix spelling still
appears in spool. What follows is kept as the record of why the split mattered.

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
**Status:** DELIVERED. `Tree` is the name, `Archive.params` is declared with it,
and `tests/archive_test.tw` writes and reads one. Closed. loom entry 2 is the
same entry and closed the same way.

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
**Status:** the atomic-rename half is DELIVERED and selvedge has not taken it
up; locking is still absent. twill 1.7.1 has `rename`, so `save_index` could
write a temporary file beside the index and rename it over, which turns a
crash mid-write from a truncated index into no change at all. It does not: it
still writes the index in place. Two processes registering at once still lose
one of the writes, and no rename fixes that.

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
**Status:** DELIVERED. `twill test tests` collects `*_test.tw`, runs each in a
fresh interpreter and reports once. CI calls it and so does the README.

`tests/harness.tw` did not go away and the three copies are still three. The
runner says which file passed; the harness names the assertion inside it, which
is a different job. What would delete the three copies is a `std/test`.

### 15. Temporary files, and cleaning up after a test

**Would improve:** `tests/archive_test.tw`, `tests/registry_test.tw`
**Status:** DELIVERED. twill 1.7.1 has `remove_file`, `remove_dir`,
`remove_all`, `rename`, `temp_dir` and `path_exists`. `tests/harness.tw` has a
`cleanup` and the suites call it, so a run no longer leaves its files behind.
What is still convention rather than runtime is that the tests write beside the
source instead of under `temp_dir()`.

The archive and registry tests write into the test directory. They are named
`tmp_*` and `.gitignore` covers them, which is a convention doing a runtime's
job. A test that cannot clean up is a test that
fails differently on a second run if it ever leaves state behind, and the only
reason these do not is that every one of them overwrites rather than appends.
