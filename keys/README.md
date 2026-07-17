# Agent signing keys

Commits from the native-Windows program are SSH-signed with per-agent keys
registered on the owner's GitHub account (so GitHub shows Verified). The
committer email is the owner's; **which agent authored a commit is proven by
which key signed it**, not by names anyone can type.

Verify locally, per commit:

```
git config gpg.ssh.allowedSignersFile <path-to>/keys/allowed_signers
git verify-commit <sha>       # names claude-coder@ or codex-auditor@
```

Private keys never leave the program machine. Key registration: 2026-07-17,
owner-authorized.
