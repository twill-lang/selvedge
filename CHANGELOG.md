# Changelog

## v0.1.0 (unreleased)

First cut of selvedge, model serialisation and the model registry for twill,
written in twill.

It runs. `twill test tests` passes six suites and
`twill run examples/publish.tw` publishes a model and reads it back, both on
twill 1.7.1. This paragraph said the opposite until `mode systems` landed in
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
