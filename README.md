# sbash

`sbash` adds lightweight policy checks before executing shell scripts piped on stdin.

## Publisher-friendly fallback command

If you are publishing an installer command in a website or README, you can prefer `sbash`
while still supporting users who only have `bash`:

```bash
curl -fsSL https://example.com/install.sh | (command -v sbash >/dev/null 2>&1 && exec sbash || exec bash)
```

This gives you an `sbash`-first default without breaking installs for users who have not
installed `sbash` yet.

If your installer expects positional arguments, pass them after `--`:

```bash
curl -fsSL https://example.com/install.sh \
  | (command -v sbash >/dev/null 2>&1 && exec sbash --channel stable || exec bash -s -- --channel stable)
```

## Heuristic ruleset

- Regex heuristics live in `rules/heuristic_rules.tsv` so they are easy to audit in one file.
- Rule severity levels:
  - `high-confidence malicious` → immediate block.
  - `suspicious` → forwarded to AI for tie-break.
  - `clean` (default when no regex matches) → requires AI review unless `SBASH_NO_AI` is set.

## Policy env vars

- `SBASH_MODE` (default: `normal`)
  - `normal`: low-friction defaults; blocks only explicit `block` decisions.
  - `cautious`: stricter defaults; blocks `uncertain` and provider-error paths unless explicitly overridden.
- `SBASH_NO_AI` (default: `0`)
  - When truthy (`1`, `true`, `yes`), skips model/provider calls.
- `SBASH_TIMEOUT_MS` (default: `6000`)
  - Max provider call latency in milliseconds.
- `SBASH_EXPLAIN` (default: `0`)
  - When truthy, prints concise decision context (mode, heuristic, provider, AI decision, final allow/block).
- `SBASH_ALLOW_ON_ERROR` (default depends on mode)
  - In `normal` mode default is allow on provider/parse error (`1`).
  - In `cautious` mode default is block via `uncertain` on provider/parse error (`0`).

## Defaults

- Without `SBASH_NO_AI`, scripts require AI review (unless blocked first by a `high-confidence malicious` rule).
- `normal` mode allows provider/parse error paths by default (`SBASH_ALLOW_ON_ERROR=1`).
- `cautious` mode blocks provider/parse error paths by default (`SBASH_ALLOW_ON_ERROR=0`).
