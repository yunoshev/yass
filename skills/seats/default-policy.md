# How to use my seats

These are my rules, in my words. Whoever decides on switching applies them to the latest numbers.

## Balance
- Spread the load across all seats as evenly as possible.
- Look at what each seat used in the previous period, not just right now: a seat that ended last week low, or is behind its pace this week, should get more work.
- Spend first the quota that would otherwise be lost: a seat whose 7-day window resets soon with lots of room left goes ahead of one that resets in days.

## Keys (seats of kind `key`)
- Never use more than 70% of a key's 5-hour window.
- Never use more than 70% of a key's 7-day window.
- Exception, the last day: when a key's 7-day window resets within 24 hours and its owner hasn't been using it (its 7-day figure grew by less than 3 points over the last 24 hours while it wasn't our active seat), the 7-day cap rises evenly from 70% to 90% as the reset approaches (about +0.8 points per hour). The 5-hour cap stays at 70%. The last 10% is always left for the owner.
- The point of the reserve: the owner keeps room for their own work. If they leave it unused, we may take it near the end, but never drain a key dry.

## Logins (seats of kind `login`)
- Caps: 90% of the 5-hour window, 90% of the 7-day window. Past 100% the organization pays for extra usage, so leave before that.
- In the last 24 hours before a login's 7-day reset, its 7-day cap may rise evenly to 97%.

## When to switch
- Switching is not free: each account has its own prompt cache, so after a switch every open conversation and every running subagent is re-sent once, uncached (reportedly 10–13% of a 5-hour window in a busy setup). Switch when it is cheap:
  - right after a compaction (trigger `compact`), or when few subagents are running (their contexts total 150k tokens or less), or when no session has been active for 5+ minutes (the cache has gone cold anyway);
  - when a switch is due but many subagents with big contexts are running, plan it: ask the main threads to stop starting new subagents, switch once the running ones have finished, then let them resume. Don't wait longer than 20 minutes.
- Switch at once, whatever the cost, when the current seat is within 3 points of a cap, or will reach a cap within the next hour at its current burn rate.
- Otherwise switch only for a clear gain (another seat would lose a lot of quota at its reset, or the load is badly uneven), at most once an hour.
- Pick the target by: obeys its caps with room to spare (at least 10 points in both windows), then the one these balance rules favor most, then the larger 5-hour headroom.
- If every seat is at its caps, say so and stay.
