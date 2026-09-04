# Fantasy Football War Room — Core Rules

Applies to every agent, every run, every league. Agents in `.claude/agents/` reference this file and must not restate it — just follow it.

## Non-negotiable rules

1. **Verify settings, don't assume.** League settings below are current as of the last update to this file. Treat them as ground truth unless the query concerns something not covered here (mid-season rule change, a setting you're unsure about) — in that case pull it live via the API paths below before analyzing.
2. **Show the settings you used, every time.** Every output that touches scoring or roster construction must state, inline, which league and which specific settings (PPR value, TD points, flex slots) it anchored to. Silent settings = an untraceable error. This is not optional and does not get skipped because "it's already in Core Rules."
3. **Value over name recognition, always.** Anchor every player evaluation to WAR/VOR against actual format-specific replacement level — not raw ADP, generic rankings, or reputation. Show the calculation or at minimum the inputs (projected points, replacement-level baseline, resulting WAR) — don't just assert a number. ADP is cited only to show the gap vs. true value, never as the value itself.
4. **Retry once before falling back, don't hard-fail on the first error.** Render-hosted MCP servers spin down when idle — the first call after idle time is slow and may time out or error even though the service is healthy. On any tool failure: wait, retry once. Only fall back to the raw API path (or flag the failure to the user) if the retry also fails. Never silently answer from memory instead of retrying/falling back.
5. **Cite player_id, not just name.** Any agent referencing a specific player tags it with the Sleeper player_id (or Yahoo player key for Pharma FF Inc) alongside the name. This is what lets Cross-Checker catch mismatches.
6. **Leagues never blend.** Four leagues, fully separate rosters/settings/context. Never carry a number, a player note, or a roster fact from one league into analysis of another.

## League Quick Reference

### Ma Homiez — Sleeper
- `league_id`: 1389709287655235584 · `draft_id`: 1389709287655235585
- 12-team keeper/dynasty-adjacent, superflex, 2× IDP_FLEX, 15-round snake (draft endpoint is authoritative — league metadata's `draft_rounds: 3` is a stale artifact, ignore it), 60s clock
- Half-PPR: WR/TE 1.0 pt/rec (0.5 base + 0.5 bonus), RB 0.5 pt/rec; rushing attempt bonus +0.25; passing TD +6 / INT −4
- IDP: sack +2, QB hit +1, TFL +1, solo tackle +1, multi-sack bonus +2, forced fumble +3
- No taxi, no keepers (rosters start empty), no kicker
- Jon's `roster_id`: 4

### Steven and Friends — Sleeper
- `league_id`: 1386056024351342592
- 10-team redraft, superflex, 13-round snake, Jon drafts from slot 3
- ~1.5 PPR for WR/TE (0.5 base + 0.5 bonus), RB 0.5 PPR
- Roster: QB, 2×RB, WR, FLEX, REC_FLEX, SUPERFLEX, DEF, 5 bench, 1 IR — no mandatory TE slot, no kicker
- Rolling waivers, clear Wednesday; 6-team playoffs from Week 15; trade deadline Week 14
- Jon's `roster_id`: 1 (username `jcabael`)

### 32-team league — Sleeper
- `league_id`: 1397006970975744000 · `draft_id`: 1397006969625137152
- Full PPR (1.0 pt/rec); TE reception bonus +1 pt/rec (not +2); passing TDs +4 (not +6); 2 IDP starting slots
- IDP: TDs/sacks/INTs +6, sacks +6, TFL +2, pass defended +3
- QB usable only in one WR/RB/TE/QB flex — meaningfully depresses QB value vs. standard superflex
- Roster: RB, 2×WR, 2× WR/RB/TE flex, 1× WR/RB/TE/QB flex, 2× IDP, 6 bench (14 total)
- No team DEF, no kicker; FAAB waivers; 8-team playoffs from Week 15; Week 10 trade deadline
- **API access is broken**: `rosters`, `users`, `drafts`, and the bare league endpoint all return permissions errors. Roster analysis in this league currently requires Jon to paste screenshots — flag this explicitly rather than guessing at his roster.

### Pharmaceutical Football Inc. — Yahoo (Jon is commissioner, running this team for a friend — not his own team)
- League ID: 133606 · 12 teams (confirmed — the league page's "Max Teams: 16" is a cap, not the team count)
- Single QB (not superflex), full PPR (1.0 pt/rec)
- Passing TD 4pt / 25 yd per pt / INT −1; rushing TD 6pt / 10 yd per pt; receiving TD 6pt; fumble lost −2; 2pt conversion 2pt
- Kicker and DEF use standard Yahoo default tables (FG by distance, PAT, DEF sacks/INT/fumble/TD/points-allowed tiers)
- Roster: QB, 2WR, 2RB, 2 W/R/T flex, K, DEF, 4 BN, 2 IR (15 total)
- Live Standard snake draft Sun Sep 6 2026, 9pm EDT, 1-min pick clock, 15 rounds, random order revealed 30 min pre-draft — **draft format is Jon's read of "Live Standard Draft," not written-confirmed by Yahoo; re-verify in the draft room before trusting it**
- Trade deadline Nov 28 2026; playoffs 8 teams, weeks 15–17
- **No API access.** Yahoo's Fantasy API requires OAuth application/approval and no MCP connector is set up. Draft prep uses public standard-PPR ADP/rankings (FantasyPros, ESPN, PFF), not league-native pulls. Live draft picks must be manually fed by Jon during the draft — same workaround pattern as Sleeper's live-pick caching gap, but for this league it's the *only* path, not a fallback.

## Data Access — what actually works right now

| Source | Status | Use for |
|---|---|---|
| `sleeper-mahomiez` MCP: `get_league_rosters` | Working | Ma Homiez rosters |
| `sleeper-mahomiez` MCP: `get_league_info` | Errors | — fall back to `GET https://api.sleeper.app/v1/league/1389709287655235584` |
| `sleeper-stevenandfriends` MCP | Erroring on most endpoints as of last check | Retry once (rule 4), then fall back to raw API: `GET https://api.sleeper.app/v1/league/1386056024351342592/...` |
| 32-team league raw API | Blocked (permissions) on rosters/users/drafts/bare endpoint | Ask Jon for a screenshot; state explicitly that you're working from a screenshot, not live data |
| Yahoo (Pharma FF Inc) | No API/connector | Public ADP sources + Jon's manual live-draft input only |
| `/draft/{id}/picks` (any Sleeper league) | Returns empty during/shortly after a live draft (Sleeper-side caching) | Don't trust for live pick tracking — use Jon's manual updates |
| DynastyProcess player ID crosswalk | Reliable | `raw.githubusercontent.com/dynastyprocess/data/master/files/db_playerids.csv`, match on `sleeper_id` |
| Full Sleeper `/players/nfl` dictionary | Too large, returns token-limited slices | Don't pull directly — use the crosswalk instead |
| `docs.sleeper.com` | — | Check before assuming an endpoint shape not listed here |

Jon's Sleeper identity: username `jcabael`, `user_id` `1028403377371234304`. Never assume which `roster_id` is his in a league you haven't checked — resolve via `owner_id` match against the rosters endpoint.

## Standing style preferences

- Direct, no hedging. Don't re-verify or restate a setting you already confirmed earlier in this same run.
- Draft boards / pick discussion: overall pick numbers, never round.pick.
- Trade proposals: structure to favor Jon's side, not just fair value — but flag outright fleeces as unrealistic even if they'd help him.
- Draft boards: cross-positional overall tiers, so a reach is visible instantly on the clock.
- Jon corrects assumptions directly — if he pushes back on a framing, update and move on, don't re-litigate.
