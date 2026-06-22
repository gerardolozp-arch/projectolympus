# Project Olympus — Master Technical Context
## WC 2026 Live Prediction Dashboard — Complete Developer Handoff
### ⚠️ READ THIS BEFORE TOUCHING ANYTHING ⚠️

---

## LIVE SITE & REPO
- **Site:** https://h4me5.github.io/project-olympus-Vfinal
- **Repo:** https://github.com/H4ME5/project-olympus-Vfinal
- **API:** API-Football Pro — secret stored as `APIFOOTBALL_KEY` in GitHub repo secrets (never hardcode or expose)
- **League ID:** 1, Season: 2026, Host: `v3.football.api-sports.io`

---

## ARCHITECTURE

```
olympus_colab_fixed.py  (run once manually in Google Colab)
        ↓
olympus_v2p_results.json  ← FROZEN pre-tournament base. Never auto-updated.
        ↓
update_results.py  ← GitHub Actions cron every 2 min
        ↓
olympus_live.json  ← Live output (~209KB). Read by dashboard.
        ↓
index.html  ← Static GitHub Pages site. Fetches olympus_live.json dynamically.
        ↓
content_generator.py  ← Runs after updater, writes content_ideas.md
```

---

## FILE MAP

| File | Purpose | Safe to edit? |
|------|---------|--------------|
| `update_results.py` | Live updater — API fetch, 10k Monte Carlo sims, writes olympus_live.json | Yes, carefully |
| `index.html` | Entire frontend — all JS rendering logic | UI changes only |
| `olympus_v2p_results.json` | Frozen pre-tournament predictions. Source of truth for baseline. | Only after re-running Colab |
| `olympus_colab_fixed.py` | Generates olympus_v2p_results.json. Run in Colab only. | Yes, carefully |
| `olympus_live.json` | Auto-generated every 2 min. Never manually edit. | Never |
| `content_generator.py` | Auto-writes content_ideas.md after each cron run | Yes |
| `cache_players.json` | Player stats cache, 60min TTL | Never |
| `cache_events.json` | Goal events cache, permanent per finished match | Never |
| `cache_lineups.json` | Lineups cache, permanent per finished match | Never |
| `.github/workflows/` | Cron workflow | Read below before touching |

---

## MODEL ARCHITECTURE

### 4 Pillars (frozen pre-tournament — stored in olympus_v2p_results.json and var D in index.html)
| Pillar | Weight | Role |
|--------|--------|------|
| P1 — Macro Strength | 28% | Primary defensive driver (FIFA ranking, ELO, market value, WC history) |
| P2 — Player Quality | 34% | Primary attacking driver (xG, xA, key passes, FBref stats) |
| P3 — Coaching | 13% | Tactical efficiency boost both sides |
| P4 — Auxiliary | 25% | Defensive solidity + cohesion-driven attack |

### 6 Modifiers (frozen, capped +6/-8 total)
Age curve ±4.0pt, Squad depth ±3.0pt, 6-month form ±4.0pt, Fragility ±2.5pt, Penalty record ±2.0pt, Travel burden ±1.5pt

### Match Engine (Dixon-Coles Poisson)
```python
BASE_GOALS = 1.35
EXP = 1.15

def get_lambdas(home, away, teams):
    h_att = ((P2*0.50+P1*0.28+P3*0.12+P4*0.10)/100)**EXP
    a_def = ((P1*0.50+P2*0.22+P4*0.18+P3*0.10)/100)**EXP
    lh = BASE_GOALS * h_att / max(a_def, 0.1)
    la = BASE_GOALS * a_att / max(h_def, 0.1)
```

### Bayesian Score Update (live, runs after each match)
```python
LEARN = 0.08
nudge = max(-8.0, min(6.0, att_delta + def_delta))
team['score'] += nudge
team['form_nudge'] = nudge
```
Only updates composite `score`. Does NOT update P1-P4 pillars.

### League Quality Multiplier (applied to P2 player stats)
Big 5 = 1.00, Brasileirão = 0.82, MLS = 0.78, Saudi PL = 0.68, Uzbek PL = 0.52

---

## GROUPS
```
A: MEX, ZAF, KOR, CZE   B: CAN, BIH, QAT, SUI
C: BRA, SCO, HAI, MAR   D: USA, PAR, AUS, TUR
E: GER, CIV, ECU, CUW   F: NED, SWE, JPN, TUN
G: BEL, NZL, EGY, IRN   H: ESP, URU, KSA, CPV
I: FRA, SEN, NOR, IRQ   J: ARG, ALG, AUT, JOR
K: POR, COL, COD, UZB   L: ENG, GHA, PAN, CRO
```

---

## FIFA BRACKET STRUCTURE — DO NOT CHANGE THESE

### Official R32 Pairings
```python
R32_PAIRS = [
    ("1E","B1"), ("1I","B2"), ("2A","2B"), ("1F","2C"),
    ("2K","2L"), ("1H","2J"), ("1D","B4"), ("1G","B3"),
    ("1C","2F"), ("2E","2I"), ("1A","B5"), ("1L","B6"),
    ("1J","2H"), ("2D","2G"), ("1B","B7"), ("1K","B8"),
]
```

### Annex C (3rd place B-slot group restrictions)
```python
ANNEX_C_SLOTS = [
    "3ABCDF",  # B1 — faces 1E (GER)
    "3CDFGH",  # B2 — faces 1I (FRA)
    "3AEHIJ",  # B3 — faces 1G (BEL)
    "3BEFIJ",  # B4 — faces 1D (USA)
    "3CEFHI",  # B5 — faces 1A (KOR)
    "3EHIJK",  # B6 — faces 1L (ENG)
    "3EFGIJ",  # B7 — faces 1B (SUI)
    "3DEIJL",  # B8 — faces 1K (POR)
]
```

### Best 8 Thirds Selection — FIFA Tiebreakers (in order)
1. Points
2. Goal difference
3. Goals scored
4. Model score (as final tiebreaker in simulation)

---

## ALL BUGS FIXED — DO NOT REINTRODUCE

### Bug 1: Wrong FIFA R32 pairings
**Was:** Group winners facing runners-up instead of B-slot third-placers. e.g. `("1A","2B")` instead of `("1A","B5")`.
**Fix:** Replaced with correct `R32_PAIRS` and `ANNEX_C_SLOTS`. Applied in both `update_results.py` and `olympus_colab_fixed.py`.

### Bug 2: assign_annex_c() tracking used_groups instead of used_codes
**Was:** `used_groups` set — allowed same team to appear in multiple B-slots (Morocco in B1 AND B5).
**Fix:** `used_codes` set — tracks individual team codes, not groups. Multiple teams from the same group CAN occupy different B-slots.

### Bug 3: URU appearing as third-placer in B-slot display
**Was:** B-slot display used raw slot appearance counts. URU (61% chance of 2nd in Group H) dominated B5 counts by occasionally finishing 3rd.
**Fix:** `b_slot_counts[slot][code]` only increments when team genuinely qualified as top-8 third in that sim via FIFA tiebreakers. `resolve_b_slot_display()` deduplicates by both `used_codes` AND `used_groups`.

### Bug 4: CUW appearing in B5 (wrong team)
**Was:** Display picked modal 3rd-place team per eligible group regardless of whether they made top-8 thirds. CUW finishes 3rd most in Group E but with 0pts/terrible GD never qualifies.
**Fix:** Only teams in `b_slot_counts` (having actually passed FIFA tiebreaker selection) are considered.

### Bug 5: Morocco appearing in two R32 slots — Brazil missing entirely
**Was:** `top2_appearances()` had no cross-slot deduplication. Morocco appeared as both 1C and 2C in different sims, both showed in display, squeezing Brazil out.
**Fix:** `used_r32_codes` set in R32 card building loop — once a team is assigned to any R32 slot display position, it cannot appear in another.

### Bug 6: simulate_group() not returning stats
**Was:** Function only returned sorted standings list.
**Fix:** Returns `(standing, stats)` tuple where stats = `{code: {pts, gd, gf}}`. Stats used for proper FIFA tiebreaker ordering of third-place teams.

### Bug 7: Change% showing →+0% for all teams
**Was:** `lranked` sorted from `LIVE.ranked` (frozen pre-tournament), so live win% compared against itself.
**Fix:** `lranked` sourced from `LIVE.live.live_ranked`. Baseline comparison uses `LIVE.ranked` (frozen).

### Bug 8: Git push conflicts (cron collision)
**Was:** `git pull --rebase` caused merge conflicts when two cron runs modified same files simultaneously.
**Fix:** Workflow uses `git reset --soft origin/main` — moves HEAD to match remote while keeping our changes, then recommits. No rebase, no conflicts.

### Bug 9: Duplicate teams/groups in Colab bracket output
**Was:** `bracket["groups"]` list comprehension had no dedup. Same team appeared at positions 3 and 4.
**Fix:** Explicit loop with `used_display` set and fallback logic to pick next available team.

### Bug 10: Wrong R32 pairings in Colab (original)
Same as Bug 1 but specifically in `olympus_colab_fixed.py`. Fixed by replacing `r32p` with correct `R32_PAIRS`.

### Bug 11: Player stats missing top scorers (Haaland, Messi)
**Was:** Only fetching 5 pages of `/players` endpoint (100 players). API doesn't sort by goals.
**Fix:** Fetch 15 pages + supplemental `/players/topscorers` endpoint. Merges by player ID to avoid duplicates.

### Bug 12: Scotland showing GB flag
**Was:** `SCO` mapped to `\uD83C\uDDEC\uD83C\uDDE7` (GB) in FLAGS object.
**Status:** Intentionally kept as GB. Scotland has no ISO 3166-1 code so no standard flag emoji exists. The tag-sequence Scottish flag renders differently than all other flags on desktop. Consistent > correct here.

### Bug 13: 3-hour match duration fallback
**Was:** API sometimes keeps a match as "live" long after it ended.
**Fix:** If match shows as live but kickoff was >3 hours ago, force `is_finished = True`.

### Bug 14: League quality multiplier applied uniformly
**Was:** All players got same league discount regardless of actual club.
**Fix:** Per-club lookup system. France's win% nearly doubled after this fix — most impactful single change in the project.

---

## update_results.py — CRITICAL SECTIONS

### run_live_tournament_sims() key data structures
```python
b_slot_counts[slot][code]        # sims where code occupied that B-slot (FIFA tiebreakers applied)
b_slot_team_group[(slot, code)]  # which group that code came from in that B-slot
r32_slot_appearances[i][code]    # sims where code appeared in R32 slot i (home or away)
used_r32_codes                   # prevents same team appearing in multiple R32 display slots
```

### resolve_b_slot_display()
Builds B1-B8 display from actual sim occupancy. Deduplicates by both `used_codes` AND `used_groups`. One team and one source group per B-slot in display.

### R32 card building logic
- B-slots: reads from `resolve_b_slot_display()`
- Non-B slots: picks most frequent team from `r32_slot_appearances[i]` not already in `used_r32_codes`
- R16 onwards: deterministic chain — each card takes winner of left feeder vs winner of right feeder
- `make_card()` uses `h2h_pct()` for head-to-head win probability

### API endpoint usage
```
/fixtures?league=1&season=2026              # All 104 WC fixtures (1 call/run)
/fixtures/lineups?fixture={id}              # Starting XI (cached permanently)
/fixtures/events?fixture={id}&type=Goal     # Goals (cached permanently for finished)
/players?league=1&season=2026&page={n}      # Player stats, 15 pages (60min TTL)
/players/topscorers?league=1&season=2026    # Supplemental top scorers
```

### API Budget (Pro plan: 7,500 calls/day)
~1,560 calls on busy match days. Well within limit.

---

## index.html — CRITICAL SECTIONS

### var D
Hardcoded frozen team metadata. Contains: name, group, conf, score, dhi, P1-P4, modifier, base_score, winner%. Never auto-updated — must be manually updated if pre-tournament data changes (rare, only if re-running Colab).

### var FLAGS
Maps 3-letter codes to flag emoji. All 48 teams present. SCO = GB flag by design (see Bug 12).

### LIVE variable
Populated by fetching olympus_live.json. ALL live data comes from here.

### computePreMatchPred()
Computes pre-match win probabilities client-side using Poisson. Uses `LIVE.teams` (live-adjusted) if available, falls back to `var D`. Returns `{home_win, draw, away_win, exp_home, exp_away}`.

### renderLive() render order
1. Live matches in progress (inplay cards)
2. Updated Win Probabilities table — uses `LIVE.live.live_ranked` for live%, `LIVE.ranked` for baseline
3. Group Standings
4. Recent Results (last 12 with model prediction overlay)
5. Upcoming Fixtures (with pre-match prediction from computePreMatchPred)

### renderLiveBracket()
Reads `LIVE.live.live_bracket`. Bracket pre-computed by update_results.py — frontend just renders it.

---

## GITHUB ACTIONS WORKFLOW

```yaml
name: Live Tournament Update
on:
  schedule:
    - cron: '*/2 * * * *'
  workflow_dispatch:
jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install numpy
      - name: Run live updater
        env:
          APIFOOTBALL_KEY: ${{ secrets.APIFOOTBALL_KEY }}
        run: python update_results.py
      - name: Generate content ideas
        run: python content_generator.py
      - name: Commit results
        run: |
          git config user.name "Olympus Bot"
          git config user.email "bot@projectolympus"
          git add olympus_live.json cache_players.json cache_events.json cache_lineups.json content_ideas.md
          git diff --staged --quiet && exit 0
          git commit -m "Live update $(date -u '+%H:%M:%S UTC')"
          for i in 1 2 3 4 5; do
            git fetch origin main
            git reset --soft origin/main
            git add olympus_live.json cache_players.json cache_events.json cache_lineups.json content_ideas.md
            git commit -m "Live update $(date -u '+%H:%M:%S UTC')" --allow-empty
            git push origin main && break
            sleep $((i * 5))
          done
```

**Key:** `git reset --soft origin/main` not `git pull --rebase`. Never revert this.

---

## olympus_live.json STRUCTURE

```json
{
  "meta": { "phase": "GROUP_STAGE", "matches_completed": 41, "last_updated": "..." },
  "teams": { "ESP": { "score": 89.26, "form_nudge": -1.74, "played": 2, ... } },
  "groups": { "A": [{"code":"MEX",...}] },
  "ranked": [{"code":"ESP","win":19.0,...}],      ← FROZEN pre-tournament baseline
  "bracket": { "r32":[], "r16":[], ... },          ← FROZEN pre-tournament bracket
  "live": {
    "results": [],           ← finished matches + predictions
    "live_now": [],          ← current live matches
    "live_probs": [],        ← in-play win probabilities
    "remaining": [],         ← upcoming fixtures
    "standings": {},         ← real group standings
    "eliminated": [],
    "lineups": [],
    "tournament_goals": {},  ← player_id → goal count
    "player_stats": [],
    "live_ranked": [],       ← live sim win probabilities (USE THIS for change%)
    "live_bracket": {}       ← live bracket (pre-computed deterministic chain)
  }
}
```

---

## HOW TO REGENERATE BASE JSON (Colab)

1. Open `olympus_colab_fixed.py` in Google Colab
2. Run (~75 seconds for 50k sims)
3. Check output shows `Thirds duplicates: ✅ None`
4. Download `olympus_v2p_results.json` from Colab files panel
5. Replace file in GitHub repo → commit
6. Wait 2 minutes for cron to pick up new base data

---

## API USAGE NOTES

- Player stats cache TTL: 60 minutes (re-fetches hourly)
- Events cache: permanent for finished matches, always re-fetches live
- Lineups cache: permanent for finished matches, re-fetches live
- `NAME_MAP` in update_results.py maps API team names → 3-letter codes
- If a team shows as unmapped in logs: add to `NAME_MAP`

---

## KNOWN ISSUES / FUTURE WORK

### Model
- Group stage tiebreaker: FIFA uses H2H before GD — model uses GD first
- Extra time: draws go straight to penalties at 90min (no ET sim)
- Player-level goal probability not implemented
- Injury tracking: no mechanism to reduce P2 for injured players

### Site
- Google Analytics: needs Measurement ID (G-XXXXXXXXXX) from analytics.google.com
- Branded image generator for social posts (Python Pillow)
- Content tab on dashboard showing content_ideas.md
- GitHub raw CDN has ~5min cache ceiling regardless of fetch headers

### Infrastructure
- Twitter/X or Telegram auto-post after each match
- Instagram auto-post requires Facebook Business account + Graph API approval

---

## ⛔ DO NOT TOUCH LIST

| Thing | Why |
|-------|-----|
| `ANNEX_C_SLOTS` array | Official FIFA Annex C — any change breaks bracket integrity |
| `R32_PAIRS` array | Official FIFA bracket structure |
| `assign_annex_c()` function | Correct greedy assignment with used_codes |
| `simulate_group()` return signature | Must return `(standing, stats)` tuple |
| `b_slot_counts` / `b_slot_team_group` tracking | Fixes URU/CUW/Morocco bugs |
| `used_r32_codes` in R32 card building | Fixes Brazil missing bug |
| `lranked` source in index.html | Must be `LIVE.live.live_ranked` not `LIVE.ranked` |
| Git push strategy in workflow | Must use `git reset --soft` not `git pull --rebase` |
| `olympus_v2p_results.json` | Never auto-modify — frozen baseline |
| API key | Never hardcode — always use `APIFOOTBALL_KEY` secret |

---

## DEPLOYMENT CHECKLIST (new developer)

1. Verify `APIFOOTBALL_KEY` is set in repo Settings → Secrets → Actions
2. Verify `olympus_v2p_results.json` is in repo root
3. Check `index.html` has `GITHUB_USER = 'H4ME5'` and `GITHUB_REPO = 'project-olympus-Vfinal'`
4. Run workflow manually → check logs show fixture count, results, successful push
5. Check dashboard loads (no CORS error, no "Loading..." stuck state)
6. Open browser DevTools console — should be clean (favicon 404 is harmless)

---

*Project Olympus — WC 2026 — Thurston Hamer — June 2026*
