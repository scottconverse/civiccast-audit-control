# Canonical verdict false positive: `bcd43a88` hash-filter semantics

- **Detected:** 2026-07-16 (America/Denver), during the `fafdee3b` re-audit
- **Affected source:** `scottconverse/civiccast@bcd43a8868c1ce71840dbcc4a42ac365dec6c092`
- **Affected governance records:** the `bcd43a88` verdict committed at `ddaade7659b976259920b72c0b43e4e8b8b1d5bd`, and the initial `fafdee3b` verdict committed at `bd8b05e7b0c70b0555a767901d73085a395359b7`
- **Severity / gate:** Major evidence-integrity drift; WS1 release-truth gate
- **Owner:** Codex auditor; Scott Converse remains tie-breaker and advancement authority
- **Disposition:** corrected prospectively; historical verdict preserved without amendment

## Observed versus expected

The `bcd43a88` canonical verdict asserted that plain `git hash-object <file>` hashes raw working bytes and that `--path` is required to apply the clean filter. A second auditor conversation repeated that premise in the initial `fafdee3b` verdict while the resumed post-reboot audit was still running. That assertion is false for a named file in the tested Git version: named-file mode already uses the file's path and clean attributes. The expected audit record must distinguish filtered named-file hashing from explicit raw `--no-filters` or unqualified stdin hashing.

## Reproduction

In a disposable Git 2.54.0.windows.1 repository with `.gitattributes` containing `* text=auto eol=lf`, stage LF `sample.txt`, then replace the working bytes with CRLF and run:

```powershell
git -C $tmp rev-parse ':sample.txt'
git -C $tmp hash-object sample.txt
git -C $tmp hash-object --path sample.txt sample.txt
git -C $tmp hash-object --no-filters sample.txt
```

The index, plain named-file, and explicit-path forms all returned `fbbee861521bd5355538b096fa3998541cd33909`; only `--no-filters` returned the raw-CRLF hash `17f2fc0a7500e6b218190262d5a329086ba965ff`.

## Closing evidence

The latest version of the exact-SHA verdict at `verdicts/ws1-release-truth/fafdee3bed292618cbccd6291efd3605081fcd0f.md` records the correction and requires the coder-side evidence narrative plus a durable negative control to match the reproduced behavior. Earlier commits remain in Git history, and no descendant inherits a pass.
