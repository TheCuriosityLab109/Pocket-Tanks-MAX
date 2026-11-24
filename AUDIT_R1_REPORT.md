# BARRAGE / Pocket Tanks MAX — Audit R1 Report

**Build audited:** commit `4f809cb` "BARRAGE R1: first playable multiplayer artillery build" on `main`, TheCuriosityLab109/Pocket-Tanks-MAX
**Live deploy:** Pages https://thecuriositylab109.github.io/Pocket-Tanks-MAX/ · Worker https://pocket-tanks-max.adityasuresh2110.workers.dev
**Auditor:** Zuko · **Date:** 2026-11-19 · **Contract:** BUILD_BRIEF.md §9 (13 checks)

## VERDICT: NOT PRODUCTION READY — strong foundation, core contract holds

The builder's own verdict agrees ("implementation foundation… not yet audit-passed"). The heart of the brief — squad rotation, hidden aim, damage cap, cover, deterministic physics — is **verified working**, including live. What blocks sign-off is match pacing evidence, a few weapon/terrain approximations, spawn distribution vs §3.3, and the real-device checks that no sandbox can do.

## What Zuko verified directly (not from builder claims)
- Repo test suite run locally on Node 24: **14/14 pass**, syntax clean (`shared/engine.js`, `worker/index.js`, `frontend/main.js`).
- Live HTTP checks: Worker `/health` returns BARRAGE r1; `POST /api/rooms` creates a real room (code + host token); Pages serves the game with `config.js` containing the correct Worker URL as clean JS (the DecryptIT literal-`\n` bug did **not** recur) and `shared/engine.js` present.
- Independent physics replication (Python, dt=1/24000): engine's Single Shot (45°, power 60) lands at 800px from muzzle; independent integration of the same drag/gravity equations gives 804px — **integrator matches spec within 0.5%**.
- Auditor probes (Node, builder's engine): mid-air collision, concrete-roof cover, fall damage, wind drift, solver-aimed focus volley — numbers in the table below.

## §9 checklist — auditor status
| # | Check | Status | Evidence |
|---|---|---|---|
| 1 | No one-volley wipe / cap 40 | **PASS** | Builder sim test: 3 direct Berthas → exactly 40, target at 80 HP, 40 HP after identical 2nd volley. Auditor solver-aimed Bertha volley vs spawned team: max 33/tank, team 720→614, no deaths. Cap enforced in `applyDamage`. |
| 2 | Hiding works | **PASS** | Auditor in-sim: Bertha detonating on a concrete roof at (610,521) → tank beneath takes **0** (HP 120); identical shot with no roof → HP 86. `coverMultiplier` 0.0 fully covered / 1.0 open (builder test). Mole stops at rock/concrete (code + builder rock tests). |
| 3 | Hidden aim | **PASS** | Code read of `worker/index.js update()`: aim is sent only to the owner and current firing squadmates; host receives no aim during aim phase; enemies/non-firing teammates get public state only. Builder room test asserts the same over WS frames and passes. |
| 4 | Physics accuracy | **PASS (note)** | (a) Builder's anchor test checks the `ballistic()` formula only — but auditor's independent replication (above) proves the integrator itself. (b) Rail Slug vs 12px rock: no tunnelling (builder in-sim test, impact x≈301–312). (c) Determinism: bit-identical replays (builder test). (d) Wind drift at +50, auditor-measured: Bertha **+52px**, Single **+103px**, Splitter **+133px** — light drifts more than heavy, as specified. |
| 5 | Squad rotation | **PASS** | Builder room tests (run by auditor): 6v6 sequence exactly Red A3 → Blue A3 → Red B3 → Blue B3 → Red A3 → Blue A3; 5v5 = 3,3,2,2; 4v4 = 2,2,2,2. Dead/empty-squad skip is in `beginTurn`. Disconnect-during-aim path exists in code; not covered by a test — minor gap, not a fail. |
| 6 | Match length 20–35 turns | **NOT VERIFIED — AT RISK** | Builder did not measure it (admitted in BUILD_NOTES). Auditor bot match (6v6, Single Shot only, no movement, simple power-learning, random wind): hit the **40-turn limit** with only 1 kill (final HP totals Red 309 / Blue 457). Bots are weaker than humans and humans get Berthas etc., but the signal says pacing is more likely too slow than too fast. Needs a human-played match measurement before sign-off. **Owner ruling (2026-11-19, after this report): damage must be decisive — a team must clearly WIN inside the limit (wipe or overwhelming margin); a match settled by turn-limit HP totals fails this check.** |
| 7 | Terrain & tanks | **PARTIAL** | PASS parts: ROCK takes exactly 3 hits, CONCRETE immune (builder in-sim); fall damage auditor-verified exact — 60px drop → 5 HP = (60−40)/4. FAIL/approx parts: **Napalm has no pool at all** — it damages tanks near the impact point if they're near their own surface; no downhill flow, no pooling, no concrete-edge behaviour (§6/§9.7 not implemented as specified). **Dirt Ball burial is weak**: carve excludes the tank's own cells and `settle()` pushes a covered tank back up, so a tank can't really be buried/dug out the way §9.7 requires; Excavator interactions therefore only partially exist. Roller/Bouncer use flat-surface approximations (builder-admitted). Earthquake surface-continuity is implemented in code but untested. |
| 8 | Mid-air collision | **PASS** | Auditor probe: symmetric crossing Single Shots detonated together at (640, 356) at t=0.917s, **zero damage** to any tank — exactly the §9.8 scenario. |
| 9 | Timeout & lock-in | **PASS** | Code + room test: only locked shots fire; unlocked players hold fire at timeout (alarm → `fire()` uses locked only); full lock fires immediately (lock handler); movement fuel 60px and slope logic in `moveTank` (builder test covers fuel). Minor bug: `moveTank` reports `moved = |step|` even when the map edge clamps actual travel, so fuel can drain without moving at the border. |
| 10 | Reconnect survival | **PASS (with deviation D3)** | Seat-token reclaim implemented (same tank/squad/HP/inventory — state lives in the Durable Object); new joins blocked after lobby; host disconnect pauses the aim timer with remaining time stored, host reconnect resumes (room test covers pause/resume). The brief's "in-flight volley is voided and re-fired" does not apply under deviation D2 — resolution is instant server-side and the replay is playback only, so there is no half-applied state to void. Coherent, but it is a change from the brief text. |
| 11 | 12-player load | **PARTIAL** | Live room creation + Pages/Worker health verified; engine handles a 12-tank volley in tests; live host+join flow works (addendum). No 12-client live run yet. |
| 12 | Zero console errors | **PARTIAL** | Live browser smoke (addendum): landing, host, join and roster flows ran with no blank pages and no error toasts. Match-phase screens not yet exercised live; full console sweep still owed. Syntax checks clean. |
| 13 | Real-device check | **OPEN — owner's rehearsal** | ≥4 real phones + TV, on Aditya. Cannot be substituted by any simulation. |

## Deviations from the brief (builder-documented, owner's discretion — recorded, not silently accepted)
- **D1. Resolution 1280×720 with 4px material cells** (brief: 1920×1080, 1px grid). Consequence: terrain/physics granularity is 4px; all brief distances were effectively rescaled by the builder's weapon table (radii/damage kept at brief values in the smaller field, so blasts cover relatively more of the map). Playable, but it is not the brief's field.
- **D2. Physics authority moved from the TV host to the Durable Object** (brief §8.1 said host runs the sim). Auditor's view: this is *better* for integrity and reconnects, and the frontend replays the server's trace — but it changes §8.1 and makes D3 necessary.
- **D3. Host-disconnect volley semantics** changed accordingly (see check 10).
- **D4. Weapon approximations** (builder-admitted): Roller, Bouncer, Mole, Napalm, Earthquake are first approximations. Napalm and Dirt-Ball burial fall short of §6/§9.7 as detailed in check 7. Also: **Splitter spread is ±12°** in code (brief: ±6° around velocity direction) — undocumented.
- **D5. Spawn distribution does not follow §3.3** (undocumented): all maps spawn each team in a **tight line, 56px apart** (`spawn()` uses slot index only). The brief specified distributed covered slots (bunker S2/S5, mesa S3, trench S4, open S1/S6 on The Forts, and map-specific covered positions on the others). On The Forts some line positions land inside the fort structures by accident; on Canyon/Hollow the described covered spawns aren't realised either. Measured consequence is acceptable for wipe-risk (check 1) but splash is amplified (one Bertha can touch ~3 tanks) and the "places to hide" design intent is only half-delivered at spawn.

## Undocumented minor findings
- **M1.** Turn-limit tiebreak: brief says HP tie → more alive tanks → sudden death. Code skips the alive-count step (HP tie simply plays on). Harmless, off-spec.
- **M2.** Kill/MVP attribution: every death in a volley is credited to the *first* shooter who dealt any damage that volley (`fire()` deaths loop). Cosmetic stat error; MVP table can be wrong.
- **M3.** Projectiles pass through *friendly* tanks in flight (direct-hit test only checks enemy tanks). Explosion damage to friendlies works correctly at 50%.
- **M4.** Damage falloff reaches ×0 at radius+12px rather than at radius (targets are checked out to radius+20). Negligible in practice.
- **M5.** Docs inconsistency: README/BUILD_NOTES refer to "PR #2" and branch `build/r1-playable-foundation`; the build actually landed as a direct commit on `main` (4f809cb) and no PR is open. Harmless, confusing for the next round.
- **M6.** Durable Object storage: the whole room (including the 57,600-cell grid as a typed array) is `storage.put` on nearly every message. Works in tests with mocks; worth watching under real 12-player load (check 11) for latency/quota surprises.

## Fix list for the builder (AUDIT R1)
1. **Match pacing (check 6):** play/measure a full 6v6 match and report turns + final HP; if matches run to the limit with most HP intact (as the auditor's bot proxy suggests), tune within the brief's numbers (e.g. weapon mix/uses) — do not raise the 40 cap or per-hit max. **Owner ruling (2026-11-19): the target is a decisive win — one team wiped or overwhelmingly ahead inside the limit. A turn-limit HP decision is a failed match, not a result.**
2. **Napalm (check 7):** implement the §6 pool — surface-following, downhill flow, stops at concrete edges — or bring a proposed simplification to the owner as a brief amendment.
3. **Dirt Ball / burial (check 7):** make burial real (tank covered → cannot move, can fire per §4.7) and Excavator rescue work end-to-end, with a test.
4. **Spawns (D5):** implement per-map spawn slots per §3.3, or propose the line-spawn as a brief amendment to the owner with the splash numbers from check 1.
5. **Splitter spread (D4):** ±6° per brief, or amend the brief.
6. **Minor:** M1 tiebreak step, M2 kill attribution, M3 friendly direct hits, move-at-edge fuel drain (check 9).
7. **Evidence gaps:** add tests for disconnect-during-aim (check 5), Earthquake continuity (check 7), and a *simulated* anchor landing (check 4a) so the suite proves what §9 asks, not just the formula.

## Addendum — live browser smoke test (2026-11-19, real browser session against the live deploy)
- **Landing page: PASS.** Renders fully — headline, HOST A BATTLE / JOIN WITH CODE, live battlefield preview canvas (not blank).
- **Host create room: PASS.** Room `3NEN63` created; host screen showed QR code, join link, Map/Timer/Series settings, roster; status reached CONNECTED; START correctly disabled ("Requires two balanced teams").
- **Player join: PASS.** Join link prefilled the code; callsign AUDITBOT joined and landed on the enlisted screen: **RED · SQUAD A, Slot 1, 120/120 HP**, CONNECTED.
- **Host roster sync: PASS.** AUDITBOT appeared on the host screen as RED A (1/12) and the host canvas updated to show the tank.
- **Match start / firing controls: NOT TESTED live.** A second distinct player could not be created from another tab in the same browser profile — the game persists seat identity in localStorage per profile, so every tab restored the same AUDITBOT seat. That is correct behaviour for real phones (one device = one player) but it means the live match flow still needs two real devices — i.e. check 13's rehearsal.
- **No blank pages, no error toasts** at any step. This upgrades check 12 to **partially verified** (landing/host/join flows only) and check 11 to **partially verified** (live room + join over the real Worker; 12-client load still untested).
- **Open question:** the host's Remove-player (×) button showed no observable effect when clicked in the automated session — most likely the `confirm()` dialog wasn't confirmed by the automation, but this must be retested by hand in the rehearsal (host recovery is a §0 requirement).
