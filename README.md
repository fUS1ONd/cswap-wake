# cswap-wake

[Русская версия](README.ru.md)

Keep the 5-hour usage windows of **all** your Claude Code accounts started, on a staggered schedule, without touching the active one.

Claude Code counts usage in 5-hour windows that start with your first message. If a window is already running when you sit down to work, it resets sooner, and you get a fresh limit sooner. `cswap-wake` sends a tiny "qq" to every account managed by [claude-swap](https://github.com/realiti4/claude-swap) (`cswap`) as soon as its window has closed. Accounts are spaced 30 minutes apart, so their limits don't all reset at once.

## How it works

Cron runs `cswap-wake` every minute. Most ticks only read the state file and exit.

When an account's next ping is due:

| Account state (from `cswap list --json`) | Action | Next check |
|---|---|---|
| 5-hour window already open (you used it, or auto-switch did) | no ping | window reset + 1 min |
| weekly limit at 100% | no ping | weekly reset + 1 min |
| no open window | `cswap run N -- -p "qq"` (empty context, ~540 tokens) | now + 5 h 1 min |
| ping failed | — | retry in 15 min |

- **Any number of accounts.** New `cswap` accounts are picked up within 30 minutes and take the nearest free 30-minute slot. Removed accounts are forgotten. Accounts disabled with `cswap disable` are skipped.
- **Stable identity.** Accounts are keyed by email + organization, so `cswap swap` / `cswap move` don't confuse the schedule.
- **Never switches your login.** Pings go through `cswap run`, which uses a separate session profile, so running Claude Code sessions stay on their account.
- **One ping at a time.** If several accounts are due at once (say, after downtime), one gets pinged and the rest shift by 30 minutes.
- **Survives restarts.** State lives in `~/.local/state/cswap-wake/state.json`. A lock file prevents overlapping ticks.

## Requirements

- Linux or macOS with cron
- Python 3.9+ (stdlib only)
- [Claude Code](https://docs.claude.com/en/docs/claude-code) (`claude` on `PATH`)
- [claude-swap](https://github.com/realiti4/claude-swap) with your accounts added (`cswap add`)

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/fUS1ONd/cswap-wake/main/cswap-wake -o ~/.local/bin/cswap-wake
chmod +x ~/.local/bin/cswap-wake
```

Add to `crontab -e`. Cron has a minimal `PATH`, so pass the one where `claude` and `cswap` live:

```cron
* * * * * env HOME=$HOME PATH=$HOME/.local/bin:/usr/local/bin:/usr/bin:/bin $HOME/.local/bin/cswap-wake >> $HOME/cswap-wake.log 2>&1
```

## Usage

```bash
cswap-wake status   # schedule: who gets pinged next and when (MSK)
cswap-wake          # one tick, the same thing cron does
```

Example log:

```
[cswap-wake 2026-10-20 09:01 UTC] new account a@example.com, first ping 2026-10-20T09:01:01+00:00
[cswap-wake 2026-10-20 09:01 UTC] Account-1 a@example.com: ok «Привет!», next 2026-10-20T14:02:01+00:00
[cswap-wake 2026-10-20 10:01 UTC] Account-3 c@example.com: window already open, next 2026-10-20T13:01:00+00:00
```

Log messages are in Russian.

## Configuration

| Variable | Default | |
|---|---|---|
| `CLAUDE_WAKE_PROMPT` | `qq` | text of the ping; any short message works, it only has to reach the model |
| `CLAUDE_WAKE_MODEL` | `claude-haiku-5-5` | model for the ping; any model opens the window, the cheapest is enough |

Both are optional: if unset, the defaults above are used. To override, add them to the cron line next to `HOME` and `PATH`:

```cron
* * * * * env HOME=$HOME PATH=$HOME/.local/bin:/usr/local/bin:/usr/bin:/bin CLAUDE_WAKE_PROMPT=hi $HOME/.local/bin/cswap-wake >> $HOME/cswap-wake.log 2>&1
```

Constants at the top of the script:

| Constant | Default | What it controls |
|---|---|---|
| `CYCLE` | 5 h 1 min | time from a successful ping to the next one; the window lasts 5 h, the extra minute keeps the ping after the reset |
| `STEP` | 30 min | spacing between accounts, so their limits don't all reset at once |
| `GRACE` | 1 min | margin after a 5-hour or weekly reset before pinging the account |
| `RETRY` | 15 min | delay before retrying a failed ping |
| `DISCOVERY` | 30 min | how often to re-read cswap for added, removed or disabled accounts |
| `STATE_DIR` | `~/.local/state/cswap-wake` | where the schedule (`state.json`) and the lock file live |

A new `STEP` applies to newly assigned slots; pings already in `state.json` keep their times, so the spacing evens out within one cycle. To re-plan from scratch, delete `state.json`.

## License

[MIT](LICENSE)
