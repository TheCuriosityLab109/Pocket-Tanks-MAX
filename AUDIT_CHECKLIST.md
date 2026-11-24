# BARRAGE — Zuko Audit Tracker

**Brief:** BUILD_BRIEF.md (§9 is the contract)
**Builder:** ChatGPT · **Auditor:** Zuko · **Repo:** TheCuriosityLab109/Pocket-Tanks-MAX
**Status:** R3+R4 audited 2026-11-20 — **NOT PRODUCTION READY**, rehearsal only: the owner accepted the full R3 endgame package on 2026-11-20, so check 6 passes; checks 11/12/13 (real devices) remain (reports: AUDIT_R1_REPORT.md, AUDIT_R2_REPORT.md, AUDIT_R3_R4_REPORT.md)

## How the audit runs (same loop as Mafia / Two Rooms / DecryptIT)
1. Code read: server-authoritative logic, no aim data in wrong payloads, numbers match brief §5/§6 exactly.
2. Live probe: drive the deployed game, run every §9 check, record measured numbers.
3. Verdict: PRODUCTION READY / NOT READY with a numbered fix list for the builder. No sign-off until all 13 pass or the owner explicitly accepts a residual risk.

## §9 Acceptance checks — R1 status
- [x] 1. **PASS** — No one-volley wipe: builder sim test 3 direct Berthas = exactly 40 damage, target alive after 2nd identical volley; auditor solver-aimed volley vs spawned team: max 33/tank, team 720→614, no deaths
- [x] 2. **PASS** — Hiding works: auditor in-sim — Bertha detonating on a concrete roof → tank beneath takes 0 (HP 120); same shot with no roof → HP 86. Cover multiplier 0.0/1.0 verified; Mole stops at rock/concrete
- [x] 3. **PASS (as amended)** — Hidden aim: aim goes only to owner + firing squadmates; host and enemies receive none during the aim phase. **R4 amendment (owner ruling 2026-11-20):** the host/TV view receives live turret ANGLES only (`turretAngles`, host-role view, `{id, angle}` — power/weapon structurally absent); enemy/non-firing phones never receive it. Verified in R4 code + room tests.
- [x] 4. **PASS (note)** — Physics: auditor's independent fine-step integration matches the engine within 0.5% (800 vs 804px); Rail Slug vs 12px rock — no tunnelling; replays bit-identical; wind drift at +50 measured Bertha +52 / Single +103 / Splitter +133. Note: the builder's own anchor test checks the formula, not a simulated landing — add a sim anchor test
- [x] 5. **PASS** — Squad rotation: room tests run by auditor — 6v6 order exactly A3/A3/B3/B3 alternating teams, 5v5 = 3/3/2/2, 4v4 = 2/2/2/2; dead/empty-squad skip in code. Gap: no test for disconnect-during-aim
- [x] 6. **PASS (R3, owner-accepted 2026-11-20)** — Match length + OWNER RULING (decisive win inside the limit; HP decision = fail). R3's pacing gate reproduces on the auditor's machine: 12/12 decisive before turn 40 (5 wipes, 7 mercy; 9/12 in the 20–35 band), and the auditor's 6 fresh unpublished seeds also went 6/6 decisive (27–38 turns). The result is produced by the R3 endgame package (Final Barrage + Twinshot from turn 16, graduated mercy, Sudden Death arena at 30) — **the owner accepted the full package on 2026-11-20**, so the condition is lifted and these mechanics are now part of the game's accepted design. Bots are predictor-grade; human confirmation rides on check 13. Note: BUILD_NOTES' R3 results table is stale post-R4 (about half the rows no longer reproduce exactly; the gate itself passes on current code).
- [x] 7. **PASS (R3)** — Terrain & tanks: Napalm pool, Dirt Ball burial + Excavator rescue, Earthquake continuity/concrete cut, fall damage, rock/concrete durability all verified (R1/R2); R3 closed the last sub-items — Roller ground-follow/friction/trench-stop/tank-contact and Bouncer two-ricochet behaviour implemented with passing regression tests (suite 36/36)
- [x] 8. **PASS** — Mid-air collision: auditor probe — symmetric crossing Single Shots detonated together at (640, 356), zero damage to any tank
- [x] 9. **PASS** — Timeout & lock-in: only locked shots fire, unlocked hold fire, full lock fires early; fuel 60px verified. Minor bug: fuel can drain at the map edge without moving (moveTank reports |step| even when clamped)
- [x] 10. **PASS (deviation D3)** — Reconnect: seat-token reclaim in the Durable Object; host drop pauses with remaining time, reconnect resumes. The brief's void-and-re-fire rule is superseded by D2/D3 (resolution is instant server-side; replay is playback only)
- [ ] 11. **PARTIAL** — 12-player load: live Worker health + room creation verified; live browser host+join flow works (room created, player enlisted RED Squad A, host roster synced, no errors). No 12-client live run yet
- [ ] 12. **PARTIAL** — Console errors: live smoke of landing/host/join flows — no blank pages, no error toasts. Match-phase screens not yet exercised live; full sweep owed. Open question: host Remove-player button showed no effect in the automated session (likely the confirm dialog) — retest by hand in the rehearsal
- [ ] 13. **OPEN — owner's rehearsal** — ≥4 real phones + TV host. Also covers: match start + firing controls live, Remove-player retest, check 6 with humans

## Deviations on record (builder-documented, owner's discretion — see AUDIT_R1_REPORT.md)
- D1: 1280×720 field, 4px cells (brief: 1920×1080, 1px)
- D2: physics authority in the Durable Object, not the TV host (auditor's view: better for integrity/reconnects)
- D3: host-disconnect volley semantics follow from D2
- D4: weapon approximations (Roller/Bouncer/Mole/Napalm/Quake); Splitter spread is ±12° in code vs ±6° in the brief (undocumented)
- D5: spawns are a tight 56px-spaced line on every map, not §3.3's distributed covered slots (undocumented)

## Watch-list (from past game audits)
- Deploy workflow writing secrets/config incorrectly (DecryptIT's literal `\n` config.js bug) — **R1: checked, did NOT recur** (live config.js is clean, correct Worker URL).
- Cache staleness on GitHub Pages after redeploys — hard-refresh / version-check the live build before auditing it.
- Any "works in sandbox" claim for real phones / venue Wi-Fi — unproven until check 13.

## Rounds
- **R1 (2026-11-19)** — Build commit `4f809cb`. Auditor verdict: **NOT PRODUCTION READY**. 8 PASS (checks 1, 2, 3, 4, 5, 8, 9, 10), 3 PARTIAL (7, 11, 12), check 6 not verified / at risk, check 13 open. Fix list: AUDIT_R1_REPORT.md, "Fix list for the builder" (7 items).
- **R2 (2026-11-19)** — Build commit `8d8d1b3e`. Auditor verdict: **NOT PRODUCTION READY** — close. Suite 22/22 re-run green; Napalm, burial/rescue, spawns (§3.3), Splitter ±6°, tiebreak, kill attribution, edge-fuel all verified closed; live deploy confirmed updated. Check 6 still fails on bot evidence (mercy rule N1 awaits owner sign-off; endgame tuning needed); check 7 down to Roller/Bouncer only; 11/12 partial, 13 open. New: N2 Red starts every match in a series (first-mover edge — recommend alternating). Report: AUDIT_R2_REPORT.md.
- **R3+R4 (2026-11-20)** — Builds `d4bdb202` (R3) + `24301422` (R4). Auditor verdict: **NOT PRODUCTION READY — one owner decision + the rehearsal away.** Suite 36/36. Issue #6 fully verified: vanishing-shell bug FIXED (auditor probes: 80°/100, 60°/100, 70°/80 all land after leaving the frame, trace to y=−272); live TV turret angles implemented exactly as ruled (host-only, angle-only); camera follow shipped + live (health "r4"). Issue #4: alternating starter done; Roller/Bouncer done (check 7 PASS); pacing gate green 18/18 across builder + fresh seeds → check 6 CONDITIONAL PASS pending owner acceptance of the R3 endgame package (Final Barrage + Twinshot @16, graduated mercy, Sudden Death arena @30). Checks 11/12 partial, 13 open. Report: AUDIT_R3_R4_REPORT.md.
