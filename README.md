# cswap-wake

[Русская версия](README.ru.md)

Keep the 5-hour usage windows of **all** your Claude Code accounts started, on a staggered schedule, without touching the active one.

Claude Code counts usage in 5-hour windows that start with your first message. If a window is already running when you sit down to work, it resets sooner, and you get a fresh limit sooner. `claude-wake` sends a tiny "qq" to every account managed by [claude-swap](https://github.com/realiti4/claude-swap) (`cswap`) as soon as its window has closed. Accounts are spaced 30 minutes apart, so their limits don't all reset at once.

## How it works

Cron runs `claude-wake` every minute. Most ticks only read the state file and exit.

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
- **Survives restarts.** State lives in `~/.local/state/claude-wake/state.json`. A lock file prevents overlapping ticks.

## Requirements

- Linux or macOS with cron
- Python 3.9+ (stdlib only)
- [Claude Code](https://docs.claude.com/en/docs/claude-code) (`claude` on `PATH`)
- [claude-swap](https://github.com/realiti4/claude-swap) with your accounts added (`cswap add`)

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/fUS1ONd/cswap-wake/main/claude-wake -o ~/.local/bin/claude-wake
chmod +x ~/.local/bin/claude-wake
```

Add to `crontab -e`. Cron has a minimal `PATH`, so pass the one where `claude` and `cswap` live:

```cron
* * * * * env HOME=$HOME PATH=$HOME/.local/bin:/usr/local/bin:/usr/bin:/bin $HOME/.local/bin/claude-wake >> $HOME/claude-wake.log 2>&1
```

## Usage

```bash
claude-wake status   # schedule: who gets pinged next and when (MSK)
claude-wake          # one tick, the same thing cron does
```

Example log:

```
[claude-wake 2026-10-20 09:01 UTC] new account a@example.com, first ping 2026-10-20T09:01:01+00:00
[claude-wake 2026-10-20 09:01 UTC] Account-1 a@example.com: ok «Привет!», next 2026-10-20T14:02:01+00:00
[claude-wake 2026-10-20 10:01 UTC] Account-3 c@example.com: window already open, next 2026-10-20T13:01:00+00:00
```

Log messages are in Russian.

## Configuration

| Variable | Default | |
|---|---|---|
| `CLAUDE_WAKE_PROMPT` | `qq` | message sent to open the window |
| `CLAUDE_WAKE_MODEL` | `claude-haiku-5-5` | cheapest model is enough |

Timing constants (`CYCLE`, `STEP`, `RETRY`, `DISCOVERY`) are at the top of the script.

## License

[MIT](LICENSE)
