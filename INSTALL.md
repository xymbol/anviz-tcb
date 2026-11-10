# Installing anviz-tcb

anviz-tcb ships no code. Paste this prompt into a coding agent (Claude Code,
Codex, Cursor), filling in the language and the location:

```
Implement the anviz-tcb library in [LANGUAGE].

1. Read SPEC.md for the complete behavior.
2. Parse tests.yaml and generate a test file that runs every case, as described
   in the "Testing" section of SPEC.md.
3. Implement every function in SPEC.md, following [LANGUAGE]'s naming and error
   conventions. Use only the standard library, plus a YAML parser for the tests
   and, where the standard library has no Unicode NFKD, a normalization library.
4. Run the tests until all of them pass.
5. Place the implementation in [LOCATION].

All tests.yaml cases must pass. Don't change tests.yaml or SPEC.md to make a
test pass; if they disagree or something is unclear, stop and say so.
```

The session flow at the end of SPEC.md is guidance for using the library, not
part of it.
