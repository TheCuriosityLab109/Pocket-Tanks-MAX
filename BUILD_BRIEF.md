# Build Brief — BARRAGE
### A Pocket Tanks-style artillery game for AD Arcade — up to 6 players per side, squads firing together

**For:** ChatGPT (builder) — **Auditor:** Zuko (will verify everything in §9 before sign-off)
**Repo:** Aditya creates the GitHub repo himself (suggested name: `Barrage`); you build into it.
**Cloudflare:** Aditya does the Cloudflare login himself at deploy time.
**Status:** Design LOCKED by Aditya on 2026-11-19 (this version supersedes all earlier drafts — in particular, there is **no** all-10-fire-at-once mode, **no** attack/defend choice, **no** Flak, **no** Shield Arc, **no** Fire Delay, and **no** Spotter role; if any earlier draft mentions them, it is stale).

---

## 0. Prime directive — production-ready on first pass, and nobody dies in one attack

This game will be played at a live party on up to 12 real phones plus a TV, over home/venue Wi-Fi. There is no second chance to fix it mid-party. Therefore:

- **Real devices first.** Browser only, no installs, no app store. Must work on iOS Safari and Android Chrome. Join via QR code or short room code.
- **Reconnect survival.** A phone that refreshes, loses signal, or locks mid-turn must reclaim its exact tank, team, squad, HP and weapon inventory with zero host intervention (seat token, same pattern as Mafia / DecryptIT).
- **Loud failures.** Never a silent dead-end. Every error surfaces a plain-language message plus what to do ("Connection lost — tap to rejoin", "Waiting for host…").
- **Zero console errors** in normal play. Treat any console error as a bug.
- **Host recovery tools.** The host can fix anything without touching a player's phone: pause/resume, skip a stuck squad turn, remove a player, rebalance teams, restart the volley, end the game, rematch.
- **12-player load** (the game's cap) must be in your acceptance run, not an afterthought.
- **Auditable code.** Clear server-authoritative logic, no cleverness. Zuko will read it and probe it live, including inspecting WebSocket payloads.
- **The anti-wipe rule (Aditya's core correction, in his words):** "it needs to be more interesting, shots can be blocked mid air, its physics has to be accurate like crazy, there must be places to hide so it takes more attacks to destroy the other side completely. Otherwise it'd be one attack and everyone dies." A build that lets a team wipe — or nearly wipe — the other side in one turn has **failed**, however good it looks. §3 (cover), §4 (physics) and §5 (damage numbers) exist to enforce this.

---

## 1. What it is

Pocket Tanks, kept simple, but with multiple players on each side.

- **Host screen (TV/laptop):** a side-view destructible battlefield. Red team tanks on the left, Blue team tanks on the right, terrain, bunkers and tunnels between and around them. The host does **not** play — they run the show.
- **Phones:** every player controls exactly **one tank**: aim (angle + power), optionally move a short distance, pick a weapon, lock in.
- **Teams attack one after the other** — exactly like Pocket Tanks — **but not one player at a time.** On a team's turn, a whole **squad** of that team aims and fires **simultaneously** (see §2). That is the entire multiplayer twist. Do not add other twists to v1.
- **Player count:** 2–12 players, two teams of 1–6. Teams must be equal size or differ by at most 1 (lobby auto-balances; host can swap players before start).
- **Skin:** AD Arcade noir-and-gold, consistent with the hub. No new branding exercise — the work is the physics.

---

## 2. Turn structure — implement exactly

### 2.1 Squads
Each team is split into two squads, **A** and **B**, assigned in the lobby (join order, shown on every phone and on the host screen). Squad sizes by team size:

| Players per side | Squad A | Squad B |
|---|---|---|
| 6 | 3 | 3 |
| 5 | 3 | 2 |
| 4 | 2 | 2 |
| 3 | 2 | 1 |
| 2 | 1 | 1 |
| 1 | 1 | — (solo player fires every team turn) |

### 2.2 Turn order
Teams alternate turns. Within a team, squads alternate turns. Red wins a coin flip (host can override) and takes the first turn.

Example, 6v6:
`Red A (3 fire) → Blue A (3 fire) → Red B (3 fire) → Blue B (3 fire) → Red A → …`

Example, 5v5:
`Red A (3) → Blue A (3) → Red B (2) → Blue B (2) → Red A → …`

### 2.3 One squad turn, step by step
1. **Banner:** host screen and all phones show whose turn it is, e.g. "RED — SQUAD A TO FIRE", plus the current wind (see §4.4). Wind is drawn once per squad turn and is the same for every shot in that turn.
2. **Aim phase (45 seconds, host-configurable 30/45/60/off, default 45):** only the firing squad's phones show controls. Each firing player may, in any order: **move** (§2.4), set **angle** (0–180°) and **power** (0–100), pick **one weapon** (§6), then tap **LOCK IN**. A locked player may unlock and change until the phase ends or everyone is locked.
3. **Early fire:** when every alive firing-squad player has locked in, the phase ends immediately — do not make the room wait for the timer.
4. **Timeout:** a firing player who has not locked in when the timer expires **holds fire** that turn (no auto-fire, no random shot). Their tank simply does not shoot.
5. **Fire:** all locked shots fire **at the same instant**. The host runs the physics sim (§4); all screens watch the same volley play out, including mid-air collisions between the squad's own shells (§4.6), terrain destruction, tanks falling, and damage.
6. **Resolve:** damage, deaths and terrain changes are applied (§5). Dead tanks are removed from the field (wreck marker stays). Win check (§2.5). Then the turn passes to the next squad of the other team.
7. **Everyone else waits — visibly.** Non-firing phones show a spectate screen: whose turn it is, your tank's HP, and "You're up after Blue B" (or equivalent). No aim controls, no inputs that affect the game. Waiting players watch the TV.

### 2.4 Movement
- On its firing turn only, before locking in, a tank may move left/right along the terrain surface using its **fuel: 60 px of horizontal travel per turn**.
- Slope limit **55°**: a tank cannot climb a steeper slope (it stops at the base). It cannot pass through another tank, a concrete wall, or leave the map. Falling into a hole does not cost fuel; getting out does.
- Movement is resolved and shown on the host screen in real time as the player drives (this is not hidden information — everyone can see where tanks *are*; only *aim* is hidden, §2.6).

### 2.5 Winning
- A team wins the moment the other team has **no alive tanks** — checked after every explosion, including mid-volley (remaining projectiles in that volley still finish their flight for the show, but no further turns are taken).
- **Turn limit:** if **40 squad turns (20 per team)** complete with both teams alive, the team with the **higher total remaining HP** wins. Tie on HP → the team with more alive tanks wins. Still tied → sudden death: squads keep alternating, and the first turn that ends with an HP lead decides it.
- **Series:** track matches won per team in the session; host sets best-of (default 3); banner the series winner. Rematch keeps teams/squads unless the host reshuffles.

### 2.6 Hidden aim — hard rule
- The opposing team must **never** see a firing player's angle, power, chosen weapon, or any trajectory preview — not on their phones, not on the host screen, and **not in any WebSocket payload** sent to them (see §8.3). Aim exists only on the firing player's own phone and server-side until the moment of firing.
- Teammates **in the same firing squad** can see each other's angle/power/weapon on their phones (numbers and weapon name only, no trajectory line) so they can coordinate ("all focus their #3!"). Teammates in the *other* squad see the spectate screen like everyone else.
- There is **no trajectory preview for anyone, ever** — Pocket Tanks style. Skill is judging angle/power from your last shot. After a volley, the host screen may show a fading tracer of the shots that were just fired (that information is already public).

### 2.7 Deliberate v1 omissions (do not build these)
- **No attack/defend choice.** Every firing player attacks. There are no intercept, shield, flak, or guard actions. Blocking happens passively through cover and trajectory (§3, §4.6) — never through a defensive action.
- **No cross-team simultaneous fire.** Teams always alternate; opposing shells are never in the air at the same time.
- **No shop / economy / XP between matches.** Weapon inventory is dealt at match start (§6).
- **No Spotter role, no supply drops, no pings.** A dead player spectates (§7).
- **No fire-delay / fuse timers / staggered firing.** Locked shots fire together at the same instant.
- **No AI tanks in v1.** Empty slots simply mean a smaller team (see squad table). Do not fill with bots.
- **No focus-fire damage bonus.** Coordination pays off through the damage numbers themselves, not a multiplier.

---

## 3. Terrain, cover, and maps — the hiding places

### 3.1 Terrain is a 2D material grid, NOT a heightmap
This is a hard technical requirement, and it is what makes hiding possible. A heightmap (one ground height per x) cannot represent a bunker roof, a cave, a tunnel, or an overhang — a tank under any of those must be a legal, reachable, protected position. Therefore:

- The battlefield is a logical **1920 × 1080** grid. Every cell has a material: `EMPTY`, `DIRT`, `ROCK`, or `CONCRETE`.
- **DIRT** — destroyed by 1 explosion hit on that cell. Falling/settling is NOT simulated for dirt (destroyed dirt is gone; added dirt stays where placed) — keep it simple and deterministic.
- **ROCK** — takes **3** explosion hits on that cell before it is destroyed (track per-cell hit points).
- **CONCRETE** — **indestructible.** Never damaged, never carved. Bunkers, tunnel supports and map borders use it. (The Mole stops at concrete, §6.)
- Explosions carve a circular crater in DIRT (and damage ROCK cells in radius) per §4.5. Utility weapons can add DIRT (Dirt Ball) or remove it (Excavator), §6.
- Rendering is the host/phone Canvas drawing this grid (phones only need the host's rendered state during a volley — phones never run terrain physics).

### 3.2 Cover must actually protect — occlusion rule
- When an explosion resolves, **before** carving its crater, sample the straight lines from the explosion centre to **5 points** on each nearby tank (centre + 4 corners of its hitbox). Count the fraction of sampled points whose line passes through any non-EMPTY cell.
- Damage multiplier from cover: **0% blocked → ×1.0 · partially blocked → scaled linearly · 100% blocked → ×0** (a fully covered tank takes **zero** damage from that explosion, however big). This one rule is why a bunker works, why a lobbed shell that arcs *over* the bunker roof and lands behind it is dangerous, and why a flat shot into the bunker wall is wasted. Do not replace it with a flat "in cover" flag.
- Friendly fire is **ON at 50%** damage (including self-damage), then the same cover and cap rules apply.

### 3.3 Maps — ship all three, all symmetric
Each map defines: the material grid, and **6 ordered spawn slots per side** (S1–S6). For a team of N, use slots S1–SN. Spawn order is balanced so small games still get a mix of covered and open slots. Every spawn slot must satisfy: the tank starts on solid ground, and at least the slots marked *covered* have material above or in front of them (toward centre) at spawn.

1. **The Forts** — the hiding map. Per side: a CONCRETE bunker (roof + back wall, open front facing centre) covering S2 and S5; a ROCK mesa in front of S3; an open trench (dug into DIRT, front wall toward centre) holding S4; S1 and S6 on open ridge. Centre: a broad DIRT hill.
2. **Canyon Cross** — per side: spawns on a canyon floor behind ROCK pillars (S1, S3, S4 covered by pillars/roof ledges), with a high ROCK plateau in the centre that blocks flat shots — high arcs or tunnelers needed. S2, S5, S6 on the canyon rim (open, better angles, exposed).
3. **The Hollow** — a big DIRT hill fills the centre and both bases, with pre-dug EMPTY tunnels inside it (CONCRETE support blocks so tunnel roofs are DIRT that *can* be collapsed by explosions from above). Per side, S3 and S4 spawn **inside tunnels** (they fire out of the tunnel mouth — their legal firing angles are naturally limited by the tunnel geometry and the physics, not by a special rule); the other slots are on the surface behind DIRT ridges.

Map pick: host chooses, or lobby majority vote — host breaks ties. Same map for a whole best-of series unless the host changes it between matches.

---

## 4. Physics — "accurate like crazy" is a spec, not a vibe

The host screen runs the one and only simulation. It must be **deterministic**: the same map, tank positions, wind, and locked shots must produce the identical volley result on replay. Log the volley seed + all locked inputs per turn so Zuko can replay any volley in audit.

### 4.1 Integrator
- Fixed timestep **120 Hz** (dt = 1/120 s), semi-implicit Euler. No variable timestep, no `requestAnimationFrame`-dependent physics. Rendering interpolates; physics never does.
- At 120 Hz a max-speed shell (§4.2) moves ≤ ~9 px per step, so it cannot tunnel through a tank (44 px wide), a bunker wall, or a thin ROCK pillar. Collision is checked **every step**, for every live projectile, against tanks, terrain cells, map borders, and other live projectiles (§4.6).
- Logical units: 1 unit = 1 px on the 1920 × 1080 field. **Gravity g = 500 px/s²**, downward.

### 4.2 Launch
- Angle 0–180° (0 = flat right, 90 = straight up, 180 = flat left), set in 1° steps (drag + fine ± buttons on the phone).
- Power 0–100 maps linearly to muzzle velocity **v = 250 + 7.5 × power** px/s (power 100 → 1000 px/s). Sanity anchor for your own testing: on flat ground, no wind, no drag, a 45° shot at power 60 (v = 700) must land **980 px** from the muzzle (±2%).
- A projectile spawns at the tank's muzzle (turret tip, 26 px from tank centre along the aim direction) — never inside the firing tank or inside terrain the tank is under. If the muzzle point is inside terrain (e.g. firing from a tunnel at a bad angle), the shot immediately collides with that terrain when fired — that is correct behaviour, not a bug: tunnel tanks must pick angles that clear the tunnel mouth.

### 4.3 Flight — gravity, wind, drag
Every projectile, every step, in this order: apply wind acceleration, apply drag, apply gravity, integrate, collide.
- **Drag:** acceleration = `−c · |v| · v`, with per-weapon drag coefficient `c` from the §6 table (heavy shells ~0.00015, standard ~0.00035, light ~0.0008). Drag is what makes light and heavy shells fly genuinely differently — do not fake it per-weapon with range multipliers.
- **Wind:** one value per squad turn (§4.4), applied as horizontal acceleration = `wind × (c / 0.00035)` — i.e. wind bites in proportion to how draggy the shell is. A Big Bertha barely drifts; a Splitter submunition drifts a lot. This must fall out of the formula, not be special-cased per weapon beyond its `c`.

### 4.4 Wind
- Drawn at the start of each squad turn: uniform integer **−50 … +50** (displayed on host + firing phones as an arrow + number, labelled e.g. "Wind ◀ 32"). Same value for every shot that turn. Re-drawn every turn — holding a perfect aim from last turn must not stay perfect.

### 4.5 Impact, explosions, terrain destruction
- A projectile resolves on: hitting a tank, hitting a non-EMPTY terrain cell, leaving the map sides/top (shot is lost, no explosion), or hitting the map floor border.
- **Resolution order within a volley is detonation-time order.** Projectiles fly together in real simulated time; when one detonates, its full effect (occlusion → damage → carve → tank settling, §4.7) is applied before simulation continues, so an early shell can blow away the cover (or the ground) that a later shell was counting on. This ordering is part of the game — implement it, don't batch all damage at the end.
- **Explosion effect, in order:** (1) compute cover occlusion per tank from the *pre-carve* terrain (§3.2); (2) compute damage (§5) and accumulate it (cap applied at end of volley, §5); (3) carve the crater: DIRT cells within blast radius are removed, ROCK cells within radius lose 1 hit point (removed at 0), CONCRETE is untouched; (4) settle all tanks (§4.7).
- Blast radius and crater radius are the same number per weapon (§6 table) unless the table says otherwise.

### 4.6 Mid-air collisions and blocking
Blocking is **physics, not a mode**. Two cases, both required:
- **Projectile vs projectile (same volley):** if two live projectiles from the firing squad come within (r1 + r2 + 4) px of each other at any step, **both detonate at the midpoint** as an airburst (normal explosion rules from that point — usually high above the ground, so usually harmless). Squadmates who aim crossing shots *will* waste them. This is intended; it is why squadmates can see each other's numbers (§2.6).
- **Projectile vs cover:** a shell whose trajectory coincides with a hill, bunker roof, mesa, pillar or tunnel wall hits that material and detonates there (§4.5) — whether it ever reaches the tank hiding behind/under it depends entirely on the weapon's trajectory type (§6): a lobbed shell can arc over a wall, a Rail Slug cannot; a Mole travels under dirt but stops at rock/concrete; a Roller is stopped by the first wall or trench lip it meets. **No weapon ignores terrain except where §6 explicitly says so (Earthquake).**

### 4.7 Tanks and terrain
- Tank hitbox **44 × 24 px**, HP **120** (§5). A tank rests on terrain: sample the terrain surface under its left, centre and right; the tank's body tilts to that surface angle (clamp visual tilt to ±40°).
- When terrain under a tank is removed, the tank **falls straight down** under gravity until it lands. Fall damage: **0 for drops ≤ 40 px, then 1 HP per 4 px** beyond that, max **20**, counted inside the per-volley damage cap (§5). A tank that falls into a pit stays in the pit.
- A tank whose hitbox ends up **fully buried** under added dirt (Dirt Ball, §6) cannot move, but **can still fire** (its muzzle is placed at the dirt surface directly above it for that shot — the one exception to §4.2's muzzle rule, and the shot must still clear surrounding terrain normally from there). It can be dug out by an Excavator or by explosions.
- Tanks never overlap terrain, never float, never clip through walls during movement or falling. If a settle would leave a tank overlapping terrain (e.g. dirt added on top of it), push it up to the nearest surface above — never sideways through a wall.

---

## 5. Damage and balance numbers — the anti-wipe maths

Implement these numbers exactly; Zuko will verify them from sim state, not from the UI.

- Tank HP: **120**.
- Per-weapon max damage (perfect direct hit, no cover): per §6 table, **never above 30**.
- **Per-tank, per-volley damage cap: 40 HP** total from all sources in one squad turn (explosions + fall damage + napalm). Enforced in the sim when the volley resolves. Consequence, and acceptance test §9.1: **no tank can die in fewer than 3 enemy volleys**, even if an entire squad focus-fires it with perfect direct hits.
- Damage falloff with distance from explosion centre: **×1.0 at 0 px → ×0 at blast radius**, linear, then cover multiplier (§3.2), then friendly-fire multiplier if applicable (§3.2).
- Expected match length: a full match on The Forts, both sides playing to win, should last roughly **20–35 squad turns in total (10–18 per team)** before a wipe or the §2.5 turn limit settles it. If your playtest matches end in ~6 turns, your damage is too high or your cover is too weak — fix the numbers, not the test.
- No healing in v1. No armour, no shields, no damage types beyond §6.

---

## 6. Weapons — v1 set of 12, implement exactly

Trajectory type is a physics behaviour, not a skin. Damage = max damage on a perfect direct hit before falloff/cover/cap (§5). Blast = blast/crater radius in px. `c` = drag coefficient (§4.3).

| # | Weapon | Trajectory type | Dmg | Blast | c | Behaviour |
|---|---|---|---|---|---|---|
| 1 | **Single Shot** | Lobbed | 25 | 55 | 0.00035 | The standard shell. **Unlimited use.** |
| 2 | **Big Bertha** | Lobbed, heavy | 30 | 75 | 0.00015 | Big crater, barely drifts in wind. |
| 3 | **Triple Burst** | Lobbed ×3 | 12 each | 40 each | 0.00035 | Three shells at aim −2°, aim, aim +2°, same power. |
| 4 | **Splitter** | Lobbed, splits | 8 each | 30 each | 0.0008 | At its apex, splits into 5 submunitions spread ±6° around the current velocity direction. Submunitions are light (wind matters). |
| 5 | **Roller** | Ground | 28 | 50 | — | On first terrain contact it does **not** explode: it rolls along the surface (gravity + friction, decel 120 px/s² on flat, accelerates downhill) for up to 4 s. Explodes on hitting a tank, a wall it cannot climb (slope > 55° while rolling), or when it stops. Stopped cold by trench walls and bunker fronts. |
| 6 | **Bouncer** | Lobbed, bouncy | 24 | 50 | 0.00035 | Bounces on terrain twice (restitution 0.55, keeps its spin/direction), explodes on the **third** terrain contact, or immediately on hitting a tank. |
| 7 | **Mole** | Tunneler | 26 | 50 | 0.00035 in air | Flies normally until first terrain contact, then burrows in a straight line (its velocity direction at entry, no gravity/drag) through **DIRT only**, up to **180 px**, then explodes underground. Stops and explodes immediately on ROCK or CONCRETE. The counter to surface cover — and useless against a concrete bunker. |
| 8 | **Rail Slug** | Direct | 30 | 25 | 0.0001 | Muzzle velocity ×1.6 (power mapping §4.2 scaled). Tiny crater, huge single-target damage. Any cover in its flat line eats it. |
| 9 | **Napalm** | Lobbed, pool | 18 | 20 carve | 0.0005 | On impact, creates a burning pool on the surface: extends up to 70 px each way **along the surface, flowing downhill** (a pool segment only exists where there is surface; it stops at concrete edges and drops into craters). Any tank touching the pool takes its damage once, at volley resolution. Pool lasts only that volley in v1 (no persistent fire). |
| 10 | **Dirt Ball** | Lobbed, utility | 0 | +60 add | 0.0005 | On impact, **adds** DIRT in a 60 px radius disc centred on the impact point (only into EMPTY cells; never replaces ROCK/CONCRETE, never spawns inside a tank — push buried tanks up per §4.7). Builds cover, fills craters, buries enemies. |
| 11 | **Excavator** | Lobbed, utility | 0 | 80 remove | 0.0005 | On impact, removes DIRT in an 80 px radius (ROCK loses 1 hit point in radius, CONCRETE untouched) and deals **no damage**. Digs out buried teammates, strips enemy cover, can drop an enemy tank into a pit (fall damage §4.7 still applies). |
| 12 | **Earthquake** | Ground shockwave | 20 | — | 0.00035 in air | Flies lobbed; on terrain impact, instead of a crater, a shockwave travels **along continuous terrain surface** up to 200 px in both directions from impact. Every tank resting on that same continuous surface within range takes flat 20 damage (cover multiplier does **not** apply — the ground itself carries it — but the per-volley cap §5 does). The shockwave stops where the surface is broken (a crater gap, a trench cut through, or a CONCRETE segment in the surface). No crater. |

**Inventory (dealt at match start, per player):** Single Shot ∞ · Dirt Ball ×2 · Excavator ×2 · plus **5 other weapons drawn at random, ×2 uses each**. Draws are **mirrored by slot**: Red S1–S6 and Blue S1–S6 with the same slot number get the **same** 5 drawn weapons, so neither team gets a luckier arsenal. A player with no damaging weapons left still always has Single Shot. Inventory is per match, refilled on rematch. Your phone shows your remaining inventory; enemies never see it (§2.6 applies to inventory too — it is aim information).

---

## 7. Screens & UX

### 7.1 Lobby (host screen + phones)
- Host creates a room → TV shows **QR code + 6-character room code**, game title, and slots filling live (name + team + squad, e.g. "Priya — RED B").
- Players join on phones with a display name, are auto-balanced across teams (join order alternates), and are assigned squads per §2.1. Host can tap a player to swap team/squad, or remove them.
- Host picks the map (§3.3) and turn timer (§2.3), sees "12/12 ready", and taps **START MATCH**. Minimum to start: 1v1.
- Noir-and-gold AD Arcade skin on every screen.

### 7.2 Host TV screen (the screen the room watches)
- Full battlefield (the 1920×1080 field scaled to fit), all tanks with name + HP bar + team colour, squad badges, wind arrow + number, turn banner ("RED — SQUAD A"), aim-phase countdown, and a clear firing moment when the volley launches.
- During the aim phase the host screen shows **positions and movement only — never aim angles, power, weapons, or trajectory lines for either team** (§2.6). Locked-in status per firing player is shown as a ✓ (that is public: the room can see who is ready).
- Host console (host only, behind a tap/hover, not blocking the field): pause/resume, skip current squad turn, force-fire with currently locked shots, remove player, end match, rematch, reshuffle teams.

### 7.3 Firing player's phone
- Your tank card (name, HP, weapon inventory), big **angle control** (drag dial + numeric, 1° steps, ± fine buttons), **power slider** (0–100 + numeric), **move ◀ ▶** buttons (fuel gauge 60 px), **weapon picker** (your remaining inventory), and a big **LOCK IN / UNLOCK** button.
- Squad panel: your firing squadmates' names with their angle/power/weapon **once they set them** (§2.6) and their locked ✓ status.
- Wind for this turn, turn timer, and whose turn follows yours.
- Controls must be usable one-handed, readable at arm's length, and must not scroll during aiming. Test on a real small phone, not a desktop window.

### 7.4 Waiting player's phone (non-firing squad + team whose turn it isn't + dead players)
- Spectate screen: "BLUE — SQUAD A is firing", your tank's HP and inventory, and when you fire next. **No game-affecting inputs.**
- Dead players stay on this screen for the rest of the match (they watch the TV; there is no other role in v1, §2.7). They are first-class for rematch — no rejoin needed.

### 7.5 Game over
- Host screen: winning team banner, final HP table, match MVP (most damage dealt — tracked server-side), series score, **REMATCH** and **NEW TEAMS** buttons. Phones mirror the result and their personal stats (shots fired, hits, damage dealt, tanks destroyed).

---

## 8. Architecture

### 8.1 Stack (the proven AD Arcade stack — do not reinvent it)
- **Frontend:** static site, GitHub Pages (GitHub Actions deploy, same pattern as DecryptIT / Two Rooms). Vanilla JS + Canvas is preferred — the sim must be easy to audit; no framework build step unless you can justify it in the README.
- **Relay / authority:** **Cloudflare Worker + one Durable Object per room**, in the same repo (`/frontend`, `/worker`). The Durable Object owns room state (players, teams, squads, turn order, inventories, HP, wind draws, lock-ins). The **host screen runs the physics sim** (§4) because it is a rendering-rate visual simulation; it receives the locked inputs from the Durable Object, simulates, and broadcasts the volley event stream (projectile paths are broadcast as the sim's resolved event log so every screen shows the identical volley — phones never simulate independently).
- **Determinism check:** the host also publishes the volley input log + seed with the result; Zuko's audit replays a volley from that log and expects an identical outcome (§9.4).
- **No accounts, no database, no stored personal data.** Room state lives only in the Durable Object for the session. No app-store submission, no payments (§10).

### 8.2 Connection rules
- Seat token per player (stored on the phone), room code + token reclaim the exact seat on reconnect, mid-turn included (§0).
- If the host screen disconnects, the Durable Object **pauses the turn timer** and all phones show "Host disconnected — game paused"; on host reconnect the match resumes from the last completed turn. A volley in flight when the host drops is **voided and re-fired** from the same lock-ins (they are stored server-side) — never half-applied.

### 8.3 Anti-leak rule (hidden aim, §2.6 — enforced here)
- Locked and un-locked aim data (angle, power, weapon, move destination is exempt — movement is public §2.4) is stored in the Durable Object and sent **only** to: the owning player, and their current firing squadmates. It is sent to the host screen **only at the moment of firing**, as part of the volley inputs.
- The host TV client must not receive aim data early and merely "not draw it" — it must not be in its payloads at all. Zuko will inspect WebSocket frames received by an enemy phone and by the host during the aim phase (§9.3).

---

## 9. Acceptance checklist — all must pass before it's called done

Zuko will run these against the live build. Report actual measured numbers in the README for 1, 2, 5 and 6.

1. **No one-volley wipe / damage cap:** scripted test — a full squad focus-fires one exposed enemy tank with perfect direct Big Bertha hits. It takes **≤ 40 damage** in that volley and is still alive after the **second** identical volley. Cap verified from sim state.
2. **Hiding works:** a tank fully behind/under CONCRETE takes **0 damage** from a Big Bertha exploding on the other side of the cover; the same tank takes full (uncapped-by-cover) damage from a lobbed shot that arcs over the cover and lands beside it; a Mole fired into the dirt above a tunnel tank damages it, and the same Mole fired at a concrete bunker does not. All four demonstrated.
3. **Hidden aim:** during an aim phase, inspect every WebSocket payload received by (a) an enemy phone, (b) a non-firing teammate phone, (c) the host screen: none contains the firing players' angle, power, or weapon. After firing, the volley is public.
4. **Physics accuracy:** (a) the §4.2 anchor shot lands at 980 px ±2%; (b) a max-power Rail Slug does **not** tunnel through a 12 px ROCK wall; (c) replaying a logged volley reproduces the identical result (same impact points, same damage) twice; (d) in strong wind (+50), a Splitter submunition drifts visibly more than a Big Bertha on the same aim — measured drift numbers reported.
5. **Squad rotation:** simulated/lived games at 6v6, 5v5 and 4v4 follow §2.2 exactly (3/3, 3/2, 2/2), including after deaths (a squad with 1 alive member fires alone; a squad with 0 alive members is skipped) and after a disconnect/reconnect.
6. **Match length:** a full 6v6 match on The Forts, both sides playing to win, lasts **20–35 squad turns total** and does not end by one team being wiped in the opening turns. Report the turn count and final HP table.
7. **Terrain & tanks:** an explosion under a tank drops it with correct fall damage (§4.7); a Dirt Ball buries a tank (it cannot move, can still fire per §4.7); an Excavator frees it; a Roller is stopped by a trench wall; Napalm pools downhill and does not cross concrete; an Earthquake damages a tank on the same continuous surface and does **not** damage one separated by a concrete segment.
8. **Mid-air collision:** two squadmates firing crossing Single Shots collide mid-air and both airburst with no tank damage (shots aimed to cross well above ground).
9. **Timeout & lock-in:** an unlocked firing player holds fire at timeout; a fully locked squad fires immediately without waiting for the timer; movement respects the 60 px fuel and 55° slope limit.
10. **Reconnect survival:** a phone that drops mid-aim rejoins as the same tank/team/squad with the same HP and inventory; a host drop pauses timers, and the in-flight volley is voided and re-fired from stored lock-ins, not half-applied.
11. **12-player load:** a full 6v6 game runs a complete match with all phones connected simultaneously over Wi-Fi with no dropped lock-ins and no visible lag in the volley playback.
12. **Zero console errors** through all of the above, on iOS Safari and desktop Chrome.
13. **Real-device check:** the whole flow is driven on real phones (at least 4 phones + TV host) — the desktop sandbox cannot prove the aiming feel, the relay under real Wi-Fi, or the simultaneous volley playback.

---

## 10. Limits — do not build past these
- **Personal party game only.** No charging, no paywall, no app-store submission.
- **Two steps are Aditya's, not the builder's:** he creates the GitHub repo, and he does the Cloudflare login at deploy time.
- **AD Arcade hub:** do not edit the live hub (`hub.html` in TheCuriosityLab109/AD-Arcade) in this build. A hub card for BARRAGE is a separate step **after** Zuko's audit passes, the way DecryptIT was added only after its audit.
- **Do not touch his other live games** (Mafia, Puppet Panic, Two Rooms, DecryptIT, WTF Quiz) — this is a new repo, a new game.
- Nothing is deployed or published until Aditya says go. This brief is the deliverable; the build follows his repo.

## 11. Deliverables
- GitHub repo: `/frontend` (static site: host TV screen + phone controllers + lobby), `/worker` (Cloudflare Worker + Durable Object), `README.md` (deploy steps: GitHub Pages + `wrangler deploy`; how to run the §9 checklist; the measured numbers from §9.1, 9.2, 9.5, 9.6).
- The volley replay/log format documented in the README (needed for §9.4 audit).

Build it like Aditya is going to play it with 11 friends on his TV — because he will, and the first squad volley must not end the game.
