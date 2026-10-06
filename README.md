# yass — Yet Another Seat Switcher

Keeps Claude Code on whichever of your subscription seats still has room. It watches the 5-hour and 7-day limits of every seat, decides by rules you state **in plain words**, and swaps the login all your sessions share.

There is no code in it: a skill, an agent prompt, three one-line hooks and a Markdown file with your rules.

## How it works

```
compaction, or every N minutes of activity (you pick N)
        │  hook (a one-line shell command)
        ▼
claude -p --agent seat-check      ← a model in the background, your session isn't touched
        │  1. which seat is in the Keychain now
        │  2. probe the seats: one tiny request each, read back the rate-limit windows
        │  3. apply ~/.yass/policy.md, your rules in your words
        │  4. switch: save the current login, write the target into the Keychain
        ▼
every Claude Code session on this Mac moves to the new seat within ~30 s, no restart
```

- **Seats** are `/login` accounts (they renew themselves) and `claude setup-token` keys (valid for a year).
- **Your rules** live in `~/.yass/policy.md`. Onboarding starts from a default and rewrites it with what you say. The default:
  - spread the load evenly, looking at what each seat used last period;
  - keys never above 70% of either window, with the weekly cap released toward 90% on a key's last day if its owner isn't using it;
  - logins stop at 90%, before paid extra usage;
  - switch at cheap moments.
- **Cheap moments.** The prompt cache is per account, so every open conversation and running subagent is re-sent once after a switch. A switch happens right after a compaction or when few subagents run. If many big subagents are running, the switch waits for them:
  1. the main thread gets a one-time note not to start new subagents;
  2. the switch happens as soon as the running ones finish (20 minutes at most);
  3. a second note tells it to resume.

  Near a cap, it switches at once.

## Install

```bash
claude plugin marketplace add yunoshev/yass
claude plugin install yass@yass
```

Or clone it and link it as a user plugin, which loads in every session:

```bash
git clone https://github.com/yunoshev/yass ~/src/yass
ln -s ~/src/yass ~/.claude/skills/yass
```

Start a new session and say "set up my seats" (or run `/yass:seats`). The onboarding goes like this:

1. It saves the current login and asks you to `/login` with each further account.
2. It takes your setup-token keys from the clipboard.
3. It asks how you want them used and writes `policy.md`.
4. It asks which model makes the decisions (Sonnet recommended) and how often to check while you work (15/30/60 min).

The hooks do nothing until onboarding is finished.

Requirements:
- macOS, because the login lives in the Keychain;
- `jq` and `xxd`, both ship with macOS;
- written against Claude Code 2.1.288.

## Safety

- Tokens never reach a model. Prompts move them file-to-file inside shell commands and compare them only by hash. The seat files are `0600` in a `0700` folder.
- `/logout` revokes the login it ends, so the prompts never use it. `/login` alone is safe.
- A login rotates its refresh token. Before every switch the current login is saved back to its seat, with the previous copy kept as `credentials.prev.json`. Swaps also wait while a login is about to renew.
- Sessions that run on `CLAUDE_CODE_OAUTH_TOKEN` or an API key don't use the Keychain. The tool leaves them alone.
- Only the official `claude` binary talks to Anthropic. There is no proxy and no direct API call.
- Use only seats you are entitled to use. Borrowed keys need their owner's consent, and the reserve in the default rules exists for that owner.

## Files

| | |
|---|---|
| `skills/seats/SKILL.md` | onboarding and everyday requests: status, switch, pause, rules |
| `skills/seats/default-policy.md` | the default rules onboarding starts from |
| `agents/seat-check.md` | the background checker: measure, decide, switch |
| `hooks/hooks.json` | three one-line hooks: compaction and the activity pulse start the checker; `PostToolUse` delivers the wind-down/resume notes to the main thread; `SubagentStop` starts a planned switch once subagents have wound down |
| `~/.yass/` | your seats, `policy.md`, `usage.jsonl`, `journal.md` |

## License

MIT
