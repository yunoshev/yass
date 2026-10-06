# TODO

## Windows support

Not started. What we know (Claude Code 2.1.290, from code and docs only): the login lives in `%USERPROFILE%\.claude\.credentials.json` and runs through the same file logic as Linux, where a running session re-reads it before every request. Details: `docs/research/claude-code-portability.md`; the bigger refactor: `docs/plans/generalize-seat-check.md`.

- [ ] Add a `Windows` branch to the Platform recipes R1 (read) and R2 (write) in `agents/seat-check.md`, and to the store reads in `skills/seats/SKILL.md` (steps 2 and 4).
- [ ] Detect Windows Credential Manager mode (`CLAUDE_CODE_FORCE_WINDOWS_CREDMAN=1` or a server flag): then a file swap does nothing, so refuse or pause with a clear message.
- [ ] Hooks run in Git Bash (PowerShell without it): require Git for Windows + `jq`, or make the hooks simpler.
- [ ] Clipboard: `Get-Clipboard` / `Set-Clipboard`; notification: `systemMessage` only.
- [ ] Check that hooks get `CLAUDE_CODE_EXECPATH` (undocumented).
- [ ] Test on a real Windows machine: hot swap, hooks, onboarding.
