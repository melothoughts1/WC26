# Randall's World Cup 2026 — Radial Bracket

A self-updating circular knockout bracket for the 2026 FIFA World Cup, with live scores.

- **Live site:** https://melothoughts1.github.io/WC26/
- **Repo:** https://github.com/melothoughts1/WC26
- **The whole site is one file:** `index.html` — no build step, no framework, no backend.

Built 2026-07-01/02 with Claude Code. Design is a clone of
[shadymccoy.github.io/WC26](https://shadymccoy.github.io/WC26/) (radial bracket concept
by Emilio Sansolini), rewired with a real-time data layer.

---

## How it works

The 32 knockout teams sit on the outer ring. Each match node is placed at the mean
angle of its two feeders, so connectors converge on the trophy at the center. As
matches finish, winner flags pop one ring inward. Hover (or tap) any flag for that
match's score or schedule; during a match the tooltip shows the **actual live score
and match clock**.

### Data sources (the important part)

| Role | Source | Poll | Notes |
|---|---|---|---|
| **Primary** | ESPN unofficial API — `site.api.espn.com/apis/site/v2/sports/soccer/fifa.world/scoreboard?dates=20260628-20260719` | 60 s | Real-time. CORS `*`, no key. Live clocks, explicit `winner` booleans, `shootoutScore` for penalties. |
| **Backup / cross-check** | `https://worldcup26.ir/get/games` | ~5 min (every 5th tick) + whenever ESPN fails | CORS `*`, no auth. Its game `id`s **are** FIFA match numbers (73–104 = knockout). |
| ~~Dropped~~ | openfootball worldcup.json | — | Original source; only regenerated a few times a day, so results lagged hours. Removed 2026-07-02. |

**Merge logic** (`mergeFeeds` in index.html): ESPN wins. Backup fills in when ESPN
is down or hasn't posted a final yet. If both report a *final* with **different
winners**, ESPN is trusted and a `⚠ sources disagree: match N` warning appears in
the footer status line.

**Slot mapping:** bracket positions are matched to ESPN by a hardcoded
`FIFA match num → ESPN event id` table (`ESPN_EVENT` const), verified 2026-07-01 by
pairing official kickoff datetimes. This is immune to team-name spelling drift.
Match 103 (third place) is intentionally excluded from the radial tree.

**Team names:** canonical names key the `ISO` flag map (flagcdn slugs). The `ALIAS`
map normalizes source spellings ("Congo DR" / "Democratic Republic of the Congo" →
"DR Congo", "United States" → "USA", "Bosnia-Herzegovina" → "Bosnia & Herzegovina",
"Cabo Verde" → "Cape Verde", etc.). ESPN placeholder names like "Round of 32 12
Winner" and "TBD" are treated as null.

### Gotchas we already hit (don't re-learn these)

1. **worldcup26.ir returns the *string* `"null"`** (and sometimes `""`) in empty
   fields like `home_penalty_score`. Naive `+value` gives `NaN`, and since
   `NaN !== NaN`, winner comparison went haywire (five phantom "sources disagree"
   warnings). Fix: `numOrNull()` strict parser — use it for anything numeric from
   that API.
2. **ESPN and openfootball disagreed on match 79's kickoff time** (1 h apart) —
   which is why mapping is by stable event id, not datetime or names.
3. **worldcup26.ir scorer names are garbled transliterations** ("Khvliv Ansisv" =
   Julio Enciso). Fine as a score cross-check; never use it for display text.
4. ESPN's API is **unofficial and undocumented** — it could change without notice.
   If it breaks, the site degrades gracefully to worldcup26.ir data (status line
   shows "ESPN down · backup data").

## Design decisions (per Randall)

- Clean near-white background `#f7f8fa` (replaced the original warm beige).
- Teams that **advance**: green ring (`--adv: #2f9e4f`) on both their outer flag
  and their advanced flag.
- Teams **eliminated**: grayscale fade + **black** outline.
- **No footer credits line**; only the status dot + "updated HH:MM".
- **No click-to-predict picks feature.** It was built (dashed-gold ghost flags,
  localStorage picks, ✓/✗ scoring) and then **removed at Randall's request** —
  it lives in git history (commit `99c070c` has it; `592c287` removed it) if
  ever wanted again.
- Champion (when decided) shows as a large gold-ringed flag over the trophy.

## Working on this from a new machine

```
git clone https://github.com/melothoughts1/WC26.git
cd WC26
# edit index.html, then just open it in a browser to test — it runs from file://
git add index.html && git commit -m "..." && git push
```

GitHub Pages auto-deploys `main` (root) about a minute after every push — no
action needed. Settings → Pages is already configured.

To sanity-check rendering without a browser open:
`msedge --headless --screenshot=out.png --window-size=1100,1200 --virtual-time-budget=10000 "file:///path/to/index.html"`

Note: there's also a duplicate copy of the page at
`d:\SteamLibrary\steamapps\common\Grand Theft Auto V Enhanced\randalls-world-cup-2026.html`
on the desktop PC (where this project started). The repo is the source of truth.

## Ideas discussed but not built yet

1. **Goalscorers + venue in tooltips** — ESPN's `details` array has scoring plays;
   venue is on the competition object. Best effort-to-payoff.
2. **Golden Boot leaderboard** — aggregate scorers across all matches.
3. **Group-stage tables on outer-flag hover** — "how did this team get here".
4. **Third-place match** as a small element outside the circle (match 103,
   ESPN event id likely 760516).
5. **Countdown to next match** under the title.

## Key code landmarks in index.html

- `ESPN_EVENT` — FIFA match num → ESPN event id map
- `CHILDREN` / `PARENT` — the bracket tree (match 104 = final = ROOT)
- `fetchESPN()` / `fetchBackup()` — both normalize to the same record shape
  (`{team1, team2, state: pre|in|post, clock, s1, s2, p1, p2, aet, winnerIdx, kick, src}`)
- `mergeFeeds()` — primary/backup merge + conflict detection
- `buildModel()` — propagates winners inward through the tree
- `render()` — SVG connectors/dots/trophy + absolutely-positioned circular flag imgs
- Geometry: polar layout, `RADIUS` per ring, angle = mean of children's angles
