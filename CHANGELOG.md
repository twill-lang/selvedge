# Changelog

## v0.1.0 (unreleased)

### Changed

- **The assertions are `std/test`.** twill 1.11 ships the assertions the test
  runner already assumed, and `docs/needs.md` entry 14 said a `std/test` was
  what would delete selvedge's copy of them. It has: every suite imports
  `std/test` as `t`, with the same five names it called before, and
  `tests/harness.tw` keeps only the scratch directory (`tmp`, `cleanup`) that
  the two file-writing suites share. `report` returns the status rather than
  calling `exit`, and it prints the summary in the shape `twill test` reads, so
  the runner now shows the counts beside every file: 146 assertions across six
  suites, where before it showed none.
- **The logical shift is the builtin.** twill 1.11's `ushr` is the shift
  `src/rstr.tw` built by hand out of four operations and a sign test, so
  `put_i64` calls it and the copy is gone. `docs/needs.md` entry 8 records the
  reason it existed.
- **The digest is the builtin.** twill 1.11's `sha256` is the same digest as
  `std/hash`, held to it by a test in twill over every message length from 0
  to 70 bytes, at machine speed: 16 MB in 6.5 ms on one machine, where
  `std/hash` took 0.69 s for 65536 bytes. `docs/needs.md` entry 7 asked for
  exactly this, and `verify` was unusable above a few megabytes without it.
  `src/digest.tw` keeps its two exports; the pin, the README's install line
  and CI move to 1.12.0.
- **`versions` sorts with the builtin.** twill 1.9.0's `sort` takes a
  comparison, which is what `docs/needs.md` entry 13 asked for, so the
  hand-written insertion sort in `src/registry.tw` is gone. A version has no
  order the language could know; `ver.compare` is the one that matters. The
  builtin is a stable merge sort, so two entries registering the same version
  still come back in the order they were registered.


First cut of selvedge, model serialisation and the model registry for twill,
written in twill.

It runs. `twill test tests` passes six suites and
`twill run examples/publish.tw` publishes a model and reads it back, both on
twill 1.9.0. This paragraph said the opposite until `mode systems` landed in
twill 1.6. See `docs/needs.md` for what the language still owes this library and
`README.md` for the status table, which names the test or the example behind
every row.

Added:

- An archive format, versioned at 1 from this commit, with the compatibility
  rule written down in `docs/format.md` before there is a version 2 to apply it
  to. A file from a newer format is refused by name, with both version numbers
  in the message.
- A reader and writer for twill's on-disk value encoding, in twill, transcribed
  from `internal/interp/serialize.go`. The four magic bytes are a contract and
  are not changed. `tests/rstr_test.tw` asserts a byte-for-byte encoding against
  a layout derived from the reference rather than from this writer.
- SHA-256 over the parameter payload's exact stored bytes, verified against the
  FIPS 180-4 vectors, with `verify` stating what it proves and what it does not.
- A local registry: register, list, resolve an exact or caret constraint, stage
  labels that refuse to resolve when two entries carry the same one, and an
  audit that compares each entry against the archive it points at.
- Lineage split into identifying and descriptive fields, with `is_reproducible`,
  `gaps` and `diff`, plus `ancestry` and `children` walking parent links across
  the registry.
- The contract check: two versions sharing a major version must agree on their
  input shape, output shape and output kind. Registering one that does not is
  refused, which is what makes a caret constraint safe to write.
- JSON metadata export through `std/json`, so a model card can be generated
  without this library.

Deliberately not included in v0.1:

- Signing, or any claim about a model's origin. `verify` proves integrity and
  says so in those words. `docs/needs.md` entry 10.
- A remote registry, uploading, downloading, or any network operation. twill has
  no sockets.
- Anything that duplicates loom's `src/checkpoint.tw`. The boundary is stated in
  `README.md` and at the top of `src/archive.tw`.
