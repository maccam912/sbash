# sbash

`sbash` is a drop-in replacement for `bash` when running scripts from stdin. It adds lightweight checks before execution so risky one-liners are less likely to run by accident.

## Replace `bash` with `sbash` in copy/paste commands

If you see:

```bash
curl -fsSL https://example.com/install.sh | bash
```

use:

```bash
curl -fsSL https://example.com/install.sh | sbash
```

That is the main workflow: same command shape, safer default.

## Install

Install `sbash` somewhere on your `PATH` (for example `/usr/local/bin`):

```bash
curl -fsSL https://raw.githubusercontent.com/maccam912/sbash/main/sbash -o sbash
chmod +x sbash
sudo mv sbash /usr/local/bin/sbash
```

Verify:

```bash
sbash --help || true
```

## Table of contents

- [Quick demo: test malware signature block](#quick-demo-test-malware-signature-block)
- [Why this exists](#why-this-exists)
- [How decisions are made](#how-decisions-are-made)
- [Publisher-friendly fallback command](#publisher-friendly-fallback-command)
- [Heuristic ruleset](#heuristic-ruleset)
- [Policy environment variables](#policy-environment-variables)
- [Defaults](#defaults)

## Quick demo: test malware signature block

This repo includes `scary-demo-script.sh`, a harmless demo script with `SBASH_MALWARE_TEST_STRING` so you can see the before/after behavior via raw GitHub content.

Run with `bash` (executes and prints the warning):

```bash
curl -fsSL https://raw.githubusercontent.com/maccam912/sbash/main/scary-demo-script.sh | bash
```

Expected output:

```text
⚠️  If you are seeing this, the script actually executed.
⚠️  A real malicious script could have run commands here.
```

Run with `sbash` (same command shape, blocked by the high-confidence test-signature heuristic):

```bash
curl -fsSL https://raw.githubusercontent.com/maccam912/sbash/main/scary-demo-script.sh | sbash
```

Expected output:

```text
Blocked: contains test malware signature string (false-positive risk: low).
```

## Why this exists

Many people know they should not blindly pipe internet scripts to a shell, but still do it because it is fast and easy. `sbash` keeps the easy workflow while adding a review layer. It is not perfect, but it is better than running everything with no checks.

## How decisions are made

`sbash` uses two stages:

1. **Heuristic rules** from `rules/heuristic_rules.tsv`.
2. **Provider review** (CLI/API) unless `SBASH_NO_AI` is set.

If a high-confidence malicious rule matches, the script is blocked immediately.

## Publisher-friendly fallback command

If you are publishing an installer command in a website or README, you can prefer `sbash`
while still supporting users who only have `bash`:

```bash
curl -fsSL https://example.com/install.sh | (command -v sbash >/dev/null 2>&1 && exec sbash || exec bash)
```

If your installer expects positional arguments, pass them after `--`:

```bash
curl -fsSL https://example.com/install.sh \
  | (command -v sbash >/dev/null 2>&1 && exec sbash --channel stable || exec bash -s -- --channel stable)
```

## Heuristic ruleset

- Regex heuristics live in `rules/heuristic_rules.tsv` so they are easy to audit in one file.
- Rule severity levels:
  - `high-confidence malicious` → immediate block.
  - `suspicious` → forwarded to provider review.
  - `clean` (default when no regex matches) → requires provider review unless `SBASH_NO_AI` is set.

## Policy environment variables

- `SBASH_MODE` (default: `normal`)
  - `normal`: low-friction defaults; blocks only explicit `block` decisions.
  - `cautious`: stricter defaults; blocks `uncertain` and provider-error paths unless explicitly overridden.
- `SBASH_NO_AI` (default: `0`)
  - When truthy (`1`, `true`, `yes`), skips provider calls.
- `SBASH_TIMEOUT_MS` (default: `6000`)
  - Max provider call latency in milliseconds.
- `SBASH_EXPLAIN` (default: `0`)
  - When truthy, prints concise decision context (mode, heuristic, provider, AI decision, final allow/block).
- `SBASH_ALLOW_ON_ERROR` (default depends on mode)
  - In `normal` mode default is allow on provider/parse error (`1`).
  - In `cautious` mode default is block via `uncertain` on provider/parse error (`0`).

## Defaults

- Without `SBASH_NO_AI`, scripts require provider review (unless blocked first by a `high-confidence malicious` rule).
- `normal` mode allows provider/parse error paths by default (`SBASH_ALLOW_ON_ERROR=1`).
- `cautious` mode blocks provider/parse error paths by default (`SBASH_ALLOW_ON_ERROR=0`).
