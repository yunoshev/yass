---
name: seats
description: Set up and run yass, which keeps Claude Code on whichever subscription seat (account or setup-token key) still has room, by rules the owner states in plain words. Use for onboarding ("set up seats", "add an account/key"); for limits and usage questions in any language: how much is used or left across all seats, burn rate, runway, a limits summary ("limits", "how much is left", "usage summary", /yass:seats limits); for the seats' status, switching seats by hand, pausing, or changing the switching rules.
---

# yass — Yet Another Seat Switcher

All Claude Code sessions on this Mac share one login in the macOS Keychain. yass keeps several seats (logins and `claude setup-token` keys) in `~/.yass/` and swaps that login, which moves every session to another seat within ~30 s, with no restart. Who decides is a model: the **seat-check** agent of this plugin measures the seats and applies the owner's rules from `~/.yass/policy.md`, written in the owner's own words. This plugin's hooks start it in the background (`claude -p`) right after a compaction and at most every `check_every_min` minutes while sessions work. When many subagents are running, a switch waits: the main threads get a one-time note to stop starting new subagents, and the switch happens once the running ones have finished (or after 20 minutes); then a second note tells them to resume.

## Rules for you

- **Never let a secret into the conversation.** Don't print, cat, Read or grep `credentials.json`, the Keychain value or a token; use only the commands below, which move secrets file-to-file. Never ask the owner to paste a token into the chat, and tell them not to run `claude setup-token` through `!` here: its output would land in the conversation.
- **Never** `/logout` or `claude auth logout` to change accounts: logging out revokes the login that was saved. `/login` alone is safe.
- Ask one question at a time. Speak the owner's language; write files in English.

`D=~/.yass` below.

## Onboarding

Go step by step; at each step say what happens and wait for the owner.

**1. Folder.**
```bash
D=~/.yass; umask 077; mkdir -p "$D/seats"; [ -f "$D/config.json" ] || echo '{"model":"sonnet","auto":false,"check_every_min":30,"active":null,"last_switch":null}' > "$D/config.json"; ls "$D/seats"; claude auth status
```
(`claude` may be `"$CLAUDE_CODE_EXECPATH"` when it is not on PATH.)

**2. The current login.** `claude auth status` shows the account and organization. A login is the pair `accountUuid` + `organizationUuid`: one person in two organizations has two seats (each organization is its own subscription). Check whether the pair is saved already:
```bash
D=~/.yass; c=$(jq -c '[.oauthAccount.accountUuid, .oauthAccount.organizationUuid]' ~/.claude.json); echo "current $c"; for id in $(ls "$D/seats"); do a=$(jq -c '[.accountUuid, .organizationUuid]' "$D/seats/$id/account.json" 2>/dev/null); echo "$id ${a:-key}$([ "$a" = "$c" ] && echo ' <- saved already')"; done
```
Only a seat marked `saved already` counts. The same `accountUuid` with another `organizationUuid` is a new seat: never refresh the other organization's seat with this login. If not saved, propose a short seat name (lowercase, from the organization, e.g. `acme`), and save it:
```bash
D=~/.yass; umask 077; S="$D/seats/<seat>"; mkdir -p "$S"
security find-generic-password -s "Claude Code-credentials" -w > "$S/credentials.json" && jq '.oauthAccount' ~/.claude.json > "$S/account.json" && jq -e '.claudeAiOauth.refreshToken | length > 0' "$S/credentials.json" >/dev/null && echo saved || echo "NOT a /login login"
jq -n --arg id "<seat>" --arg label "<organization (email)>" --arg plan "<plan, if known>" --slurpfile a "$S/account.json" '($a[0].userRateLimitTier // "") as $t | {id: $id, kind: "login", label: $label, plan: $plan, capacity: (if ($t|test("20x")) then 20 elif ($t|test("5x")) then 5 else 1 end), owner: "me", notes: "", added: (now|todate)}' > "$S/meta.json"
jq --arg t "<seat>" '.active = $t' "$D/config.json" > "$D/c.tmp" && mv "$D/c.tmp" "$D/config.json"
```
If it is saved already as `<seat>`, refresh its credentials instead (a login rotates its tokens):
```bash
D=~/.yass; umask 077; S="$D/seats/<seat>"
security find-generic-password -s "Claude Code-credentials" -w > "$S/credentials.new" && jq -e '.claudeAiOauth.refreshToken | length > 0' "$S/credentials.new" >/dev/null && cp "$S/credentials.json" "$S/credentials.prev.json" && mv "$S/credentials.new" "$S/credentials.json" && echo saved || { rm -f "$S/credentials.new"; echo NOT SAVED; }
```

Then measure it: start the **seat-check** agent (Agent tool, `subagent_type: yass:seat-check`) with `Measure only.` and report its line.

**3. More logins.** Ask whether the owner has another account to add. If so: "Type `/login` and sign in with the next account. Don't use `/logout`." When they're back, run `claude auth status`; if the account or organization changed, repeat step 2 for it. Repeat until there are no more logins.

**4. Keys.** Ask whether they have long-lived keys (`claude setup-token`, valid a year, possibly from other people's accounts). For each key:
- the owner runs `claude setup-token` **in a separate terminal**, signs in with that account in the browser, copies the printed token and tells you "copied";
- you save it straight from the clipboard as the next free `k<N>` and clear the clipboard. Ask nothing: no name, owner or plan. Say "saved as k<N>"; the owner can copy the next key right away. If they mention whose key it is or its plan, put that in `owner`/`plan`/`notes`.
```bash
D=~/.yass; umask 077; n=$(ls "$D/seats" | sed -n 's/^k\([0-9][0-9]*\)$/\1/p' | sort -n | tail -1); id="k$((${n:-0} + 1))"; S="$D/seats/$id"; mkdir -p "$S"
pbpaste | tr -d '[:space:]' > "$S/token.tmp"; pbcopy < /dev/null
grep -Eq '^sk-ant-oat01-[A-Za-z0-9_-]{20,}$' "$S/token.tmp" && jq -n -c --rawfile t "$S/token.tmp" '{claudeAiOauth: {accessToken: $t, refreshToken: null, expiresAt: ((now + 364*86400)*1000|floor), scopes: ["user:inference"], subscriptionType: null, rateLimitTier: null}}' > "$S/credentials.json"; rm -f "$S/token.tmp"
h=$(jq -r .claudeAiOauth.accessToken "$S/credentials.json" 2>/dev/null | shasum); for o in $(ls "$D/seats"); do [ "$o" != "$id" ] && [ "$(jq -r .claudeAiOauth.accessToken "$D/seats/$o/credentials.json" | shasum)" = "$h" ] && { echo "same key as $o"; rm -f "$S/credentials.json"; }; done
if [ -s "$S/credentials.json" ]; then jq -n --arg id "$id" '{id: $id, kind: "key", label: ("key " + $id), owner: "", plan: "", notes: "", added: (now|todate)}' > "$S/meta.json"; echo "saved $id"; else rmdir "$S"; echo "not saved: the clipboard did not hold a new setup-token"; fi
```

Keep taking keys until the owner says they're done ("done", «готово»). Then measure all seats with the seat-check agent (`Measure all.`): it also shows each key works. (`Seat check. Trigger: manual.` only after step 5.)

**5. The rules, in the owner's words.** This is the heart of it. Show the default rules from `default-policy.md` (next to this file) as a short summary: balance across seats by what each used last period; keys never above 70% of 5h or 7d; on a key's last day before its weekly reset, the 7d cap rises to 90% if its owner isn't using it; logins capped at 90% to stay clear of paid extra usage; switch at cheap moments (after compaction, or when sessions are idle) unless a cap is near. Then ask the owner to tell you how they want their seats used: which to spend first, what to spare, special seats, hours. Write `~/.yass/policy.md` in their words (in English, first person, as they'd say it), starting from the default and changing what they changed. Read it back in a few lines and fix what they correct. Rules for a single seat can also go into that seat's `meta.json` `notes`.

**6. The deciding model.** Ask which model runs the checks: `sonnet` (recommended: reliable and cheap enough), `haiku` (cheapest, may miss nuances), `opus` (most careful, priciest).

Then ask how often to check while sessions work: every 15, 30 (recommended) or 60 minutes. A check also runs after every compaction, the cheapest moment. Each check is a short background run of the chosen model plus one tiny probe, billed to the active seat, so 15 minutes is ~4 runs an hour; when sessions are idle nothing runs (the hook is one shell line that exits in milliseconds). Save both answers and switch automatic checks on:
```bash
D=~/.yass; jq --arg m "<model>" --argjson n <15|30|60> '.model = $m | .check_every_min = $n | .auto = true' "$D/config.json" > "$D/c.tmp" && mv "$D/c.tmp" "$D/config.json"
```

**7. First check.** Start the seat-check agent with `Seat check. Trigger: manual.` (model from step 6) and show the result and the status.

## Everyday requests

- **Status.** Latest reading per seat, the rules in force and recent events (no secrets in these files):
```bash
D=~/.yass; cat "$D/config.json"; for id in $(ls "$D/seats"); do jq -c '{id, kind, label, owner, notes}' "$D/seats/$id/meta.json"; done
jq -s -c 'group_by(.seat) | map(max_by(.t)) | .[] | {seat, h5, d7, h5_reset: (.h5_reset // 0 | strflocaltime("%a %H:%M")), d7_reset: (.d7_reset // 0 | strflocaltime("%a %H:%M")), at: (.t | strflocaltime("%H:%M"))}' "$D/usage.jsonl"; tail -n 10 "$D/journal.md"; tail -n 5 "$D/check.log"
```
- **Limits summary** ("how much is left", "limits", burn rate, «сколько осталось», `/yass:seats limits`). Refresh the current seat with one tiny request, then gather everything:
```bash
D=~/.yass; id=$(jq -r .active "$D/config.json"); env -u CLAUDE_CODE_OAUTH_TOKEN YASS_CHILD=1 "${CLAUDE_CODE_EXECPATH:-claude}" -p ok --model haiku --setting-sources "" --tools "" --strict-mcp-config --no-session-persistence --system-prompt "Reply with exactly: ok" --output-format json | jq -c --arg seat "$id" '([(if type=="array" then .[] else . end) | select(.type=="rate_limit_event") | .rate_limit_info] | first) // empty | {t: (now|floor), seat: $seat, src: "probe", status, overage: .overageStatus, h5: ((.unifiedWindows.five_hour.utilization // 0)*100|round), h5_reset: .unifiedWindows.five_hour.resetsAt, d7: ((.unifiedWindows.seven_day.utilization // 0)*100|round), d7_reset: .unifiedWindows.seven_day.resetsAt}' >> "$D/usage.jsonl"
date '+now %s %a %H:%M'; cat "$D/config.json" "$D/policy.md"; cat "$D/pending.json" 2>/dev/null; for s in $(ls "$D/seats"); do jq -c '{id, kind, label, plan, capacity, notes}' "$D/seats/$s/meta.json"; done
jq -s -c 'group_by(.seat)[] | {seat: .[0].seat, last24h: [.[] | select(.t > (now - 86400)) | [.t, .h5, .h5_reset, .d7, .d7_reset]]}' "$D/usage.jsonl"
```
Then work it out and answer in a few lines, in the owner's language:
  - per seat: 5h and 7d used against the caps its policy gives it (keys and logins differ; mind the last-day release), when each window resets (local time, "in 2 h"), how old the reading is (a window whose reset passed counts as 0%);
  - burn rate of the active seat: 5h-window points per hour over its last 1–2 hours of readings, 7d points per day over the last 24 hours; and when it reaches its cap at that rate;
  - totals across all seats, each seat counting 100% (plans are not weighed: a key's plan is usually unknown), so four seats hold 400%: 7d left = Σ(100 − d7)% of N×100% (e.g. "290% of 400%"), room to the caps = Σ(cap7 − d7)%, and how many hours that is at the current burn; the same for the 5-hour windows right now;
  - the next resets, any planned switch (`pending.json`), the last switch.

  A short table plus 2–3 summary lines. Seats not measured for hours are shown as such; if the owner wants them fresh, start the seat-check agent with `Measure all.` (each probe opens an idle seat's 5-hour window, so say that first).
- **Check now** / **switch to a seat**: start the seat-check agent with `Seat check. Trigger: manual.` or `Switch to <seat>. Reason: <owner's words>.` Use the model from `config.json`.
- **Pause / resume** automatic checks: set `.auto` to `false` / `true` in `config.json` (the hooks read it). **How often** to check while working: `.check_every_min`.
- **Change the rules**: edit `policy.md` with the owner's words, read the change back.
- **Add** a seat: onboarding steps 2–4. **Remove** one (never the active one): `mv "$D/seats/<seat>" "$D/removed-<seat>-$(date +%s)"`.
- **Something broke** (the Keychain holds a login no seat knows, a switch failed): `/login` with any saved account restores a working login; then save it again (step 2).
