# Jak 2 — Blue Crimson Guard Reskin (`crimson-blue-guard`)

> **Mod Readme / Readme du Mod**
>
> - **Branch / Branche :** `jak2/features/crimson-blueguard/crimson-redguard-behavior`
> - **Type :** `features`
> - **Depends on / Dépend de :** the existing `build-actor` custom-actor pipeline
>   (`goal_src/jak2/lib/project-lib.gp`, `goalc/build_actor/`)
> - [🇬🇧 English Version](#-english-version)
> - [🇫🇷 Version Française](#-version-française)

---

<a name="-english-version"></a>

# 🇬🇧 English Version

## 1. What this is

A blue-recolored Crimson Guard, added as its **own standalone GOAL entity**
(`crimson-blue-guard`) rather than a global texture replacement. While the mod is enabled it
replaces the stock red `crimson-guard` throughout Haven City's ambient traffic — on foot and
riding the guard vehicles — and its behaviour is **strictly inherited**: same states, same alert
reaction after a crime, same arrest and pursuit logic, same stats, same animations, same sounds,
same death (§5). The single gameplay addition is that a guard may carry a grenade launcher (§6).
Everything is gated behind one toggle, `Debug ▸ Mods ▸ crimson-blueguard`, off by default (§7).

The source asset is `custom_assets/jak2/models/custom_levels/crimson-blue-guard.glb` (also copied
to `custom_assets/jak2/models/common/crimson-blue-guard-lod0.glb`, see §4.3): the decompiled native
`crimson-guard` skeleton + all 40 of its animations, re-skinned with a recolored texture set in
Blender, then re-exported.

## 2. The core problem: animation slot indices

`crimson-guard`'s ~4700 lines of AI/state-machine code (`guard.gc`,
`levels/city/traffic/citizen/guard.gc`) reference its animations almost entirely by **numeric
slot index** into its art-group's element array — either through overridable fields
(`anim-walk`, `anim-run`, `anim-get-up-front`, ...) set once in `init-enemy!`, or, in a handful of
methods (`enemy-method-77`, `enemy-method-78`, `set-behavior!`), as **raw literals** baked
directly into the method body (`(-> this draw art-group data 42)` and friends).

The native `crimson-guard-ag` art-group has a fixed layout (see
`decompiler/config/jak2/ntsc_v1/art-group-info.min.json`, key `crimson-guard-ag`):

| Slot | Content |
|---|---|
| 0 | `crimson-guard-lod0-jg` (skinned mesh) |
| 1 | `crimson-guard-lod0-mg` |
| 2 | `crimson-guard-lod2-mg` |
| 3 | `crimson-guard-shadow-mg` |
| 4..43 | 40 animations, in a fixed order (`idle`@4, `walk`@5, `run`@6, ..., `get-up-front`@33, `get-up-back`@34, ...) |

The existing `build-actor` tool (`goalc/build_actor/jak2/build_actor.cpp`) does **not** reproduce
this layout for a standalone custom actor: it always emits a 2-slot header (`jgeo`, one dummy
null slot) before the animations, and it orders animations by their order in the source `.glb`'s
`animations` array — which a normal Blender/glTF export sorts alphabetically. Building the blue
guard "as-is" would have put `crimson-blue-guard-ag`'s `idle` at slot 2 instead of 4, `get-up-back`
at some alphabetically-derived slot instead of 34, etc. — silently playing the *wrong* animation
in every hardcoded-index code path, breaking the "identical behavior" requirement in subtle,
hard-to-notice ways (e.g. only the vehicle-knockout or yellow-eco-hit reactions, which use raw
literals, would be wrong).

## 3. The fix — two additive, opt-in pieces

### 3.1 `build-actor :native-header #t`

`goal_src/jak2/lib/project-lib.gp`'s `build-actor` macro gained a new `&key (native-header #f)`
parameter, threaded through to the `build-actor2` data-compiler tool
(`goalc/make/Tools.cpp::BuildActor2Tool`) and finally to
`jak2::BuildActorParams2::native_anim_header` (`goalc/build_actor/jak2/build_actor.h`). When set,
`run_build_actor` (`goalc/build_actor/jak2/build_actor.cpp`) emits **two extra null placeholder
slots** after the mesh, padding the header from 2 to 4 slots — matching the native layout exactly.
Default is `#f`, so every existing custom actor (`test-actor`, the jetboard, etc.) is completely
unaffected.

```lisp
(build-actor "crimson-blue-guard" :force-run #t :native-header #t)
```

### 3.2 Reordering the source `.glb`'s animation array

A one-off Python script reordered `crimson_blue_guard.glb`'s `animations` JSON array (pure
reordering of array elements — no accessor/bufferView/mesh data touched) to match the 40-name
canonical order from `art-group-info.min.json` above. Combined with the 4-slot native header,
this makes `crimson-blue-guard-ag`'s slot N hold the *same* animation as `crimson-guard-ag`'s slot
N, for every N. If you ever need to rebuild the `.glb` from a fresh Blender export, re-run
`python scripts/modding/reorder_crimson_guard_glb_anims.py <in.glb> <out.glb>` before running
`build-actor`, or your animation indices will drift again.

With both pieces in place, `crimson-blue-guard` needs only to override
`init-enemy!` (`goal_src/jak2/levels/city/traffic/citizen/crimson-blue-guard.gc`) to point at its
own skeleton-group by name — every other inherited method/state from `crimson-guard` keeps
working with the exact same numeric indices, unmodified.

```lisp
(deftype crimson-blue-guard (crimson-guard) ())

(def-art-elt crimson-blue-guard-ag crimson-blue-guard-lod0-jg 0)
(def-art-elt crimson-blue-guard-ag crimson-blue-guard-lod0-mg 1)

(defskelgroup skel-crimson-blue-guard crimson-blue-guard crimson-blue-guard-lod0-jg -1
              ((crimson-blue-guard-lod0-mg (meters 999999)))
              :bounds (static-spherem 0 0 0 5)
              :origin-joint-index 3)

(defmethod init-enemy! ((this crimson-blue-guard))
  ;; identical to crimson-guard's init-enemy!, except the skeleton-group name
  ...)
```

## 4. Getting it into the world

- **Code residency:** `crimson-blue-guard.gc` compiles to `crimson-blue-guard.o`, added next to
  `guard.o` in `goal_src/jak2/dgos/cwi.gd` (the always-resident common DGO that already carries
  `crimson-guard`'s own code).
- **Art residency:** `crimson-blue-guard-ag.go` was added next to every existing
  `crimson-guard-ag.go` entry (append-only, nothing removed) in the 10 level DGOs that carry it:
  `cas.gd`, `dg1.gd`, `fdb.gd`, `fea.gd`, `fob.gd`, `fra.gd`, `lwidea.gd`, `lwideb.gd`, `lwidec.gd`,
  `pae.gd`. This guarantees the blue variant's assets are loaded everywhere the stock guard's are,
  so it can never be picked for a spawn without its art being resident.
- **Ambient traffic spawning:** `traffic-manager.gc::traffic-object-spawn` is the single place
  where the traffic simulation turns a `(traffic-type crimson-guard-1)` /
  `(traffic-type crimson-guard-0)` pick into a concrete process, via
  `(citizen-spawn arg0 crimson-guard arg1)`. Both call sites now read the mod's master flag:

  ```lisp
  (((traffic-type crimson-guard-1))
   (set! v0-0 (citizen-spawn arg0 (if *mod-crimson-blueguard-enable* crimson-blue-guard crimson-guard) arg1))
   )
  ```

  This is the **only** touch point in the whole traffic simulation. The `traffic-type` enum, the
  `guard-type-info-array` weighting table, the want-counts and everything else about how / when /
  where a guard slot gets picked are completely untouched -- `crimson-blue-guard` is just an
  alternate concrete type for an existing spawn decision, so all traffic-engine bookkeeping
  (nav mesh, alert state, population counts) behaves identically whichever variant lands in the
  process slot. It is a full substitution, not a mix: with the mod on there are no red guards left
  in ambient traffic, with it off there are no blue ones.
- **Guards riding the guard vehicles:** the guard-bike and hellcat riders are `crimson-guard-rider`
  (`levels/city/traffic/vehicle/vehicle-rider.gc`), a `vehicle-rider` that is *not* a
  `crimson-guard` at all -- it only borrows the guard's skeleton-group. So the swap there is one
  string: `vehicle-rider-method-32` binds `"skel-crimson-blue-guard-rider"` instead of
  `"skel-crimson-guard-rider"` while the flag is on. Both skeleton-groups live in CWI and share
  the same 38-bone skeleton and animation slot numbering, so `riding-anim` (35 / 36) and every
  other numeric slot stay valid either way. A rider knocked off its vehicle respawns through
  `traffic-object-spawn` as a `(traffic-type crimson-guard-1)`, so it lands on its feet as the
  matching faction for free.
- `(declare-type crimson-blue-guard crimson-guard)` was added near the top of `traffic-manager.gc`
  so the reference above compiles independent of file ordering (same idiom as `crimson-guard`'s
  own forward declaration in `traffic-engine.gc`).

### 4.3 A second, easy-to-miss piece: the actual drawable geometry ("Circuit 2")

`build-actor` (Circuit 1, §3) only produces the skeleton/animations art-group. The actual triangles
+ textures the PC renderer draws (Circuit 2) come from a completely separate system: the
decompiler bakes them into `.fr3` files, looked up **by name** at runtime. See
`docs/modding/jak2_lisp_instructions.md` for the full mechanism.
`build-actor`'s own merc-ctrl output is a placeholder (`generate_dummy_merc_ctrl` in
`build_actor.cpp` literally reuses a hardcoded dummy mesh) — without Circuit 2, the guard spawns,
moves and makes sound normally, but is **invisible**.

The fix: a second copy of the same `.glb`, renamed to match the placeholder merc-ctrl's own name
(`<art-group-name>-lod0`, here `crimson-blue-guard-lod0.glb`), dropped in
`custom_assets/jak2/models/common/`. The decompiler's `add_custom_model_to_level`
(`decompiler/level_extractor/extract_merc.cpp`) auto-scans that folder at `task extract` time — no
config needed — and bakes the model + all its textures into `GAME.fr3` (`common` → always
resident, regardless of level). This is a one-time step (or after any `.glb` model change); it
does **not** need to be repeated after ordinary `(mi)` GOAL-code iteration.

## 5. Behaviour: identical to the red guard, on purpose

This is the design rule of the branch, and the thing to protect when editing
`crimson-blue-guard.gc`:

> `crimson-blue-guard` is a plain subtype of `crimson-guard` that **must behave exactly like the
> stock red guard**. Same states, same alert reaction after a crime, same arrest and pursuit
> logic, same stats, same animations, same sounds, same death. The mod is a reskin, not an AI mod.

Concretely, the type overrides only four things, and each one is either cosmetic or a hard
technical requirement of a `build-actor` custom actor:

| Override | Why it exists |
| --- | --- |
| `init-enemy!` | binds `skel-crimson-blue-guard` instead of `skel-crimson-guard` -- the blue mesh, i.e. the whole point of the mod. Every stat still comes from `*crimson-guard-nav-enemy-info*`. |
| `citizen-init!` | registers the blue `blue-guard-frustum` minimap icon (§5.2), then calls the parent and rolls the grenade launcher (§6). `guard-type`, `hit-points`, collide-spec and alert reaction are all left to the parent. |
| `die` + `enemy-method-78` | replicate the native purple dissolution by hand (§5.1). |
| `crimson-guard-method-214` | the optional grenade launcher (§6). |

There is **no** `general-event-handler` override, no state override, no targeting-method override.
That is deliberate: any such override would, by definition, be a behaviour difference from the red
guard. A blue guard reacts to a crime, joins a city alert, arrests Jak, changes weapon on alert
escalation and pursues exactly like a red one, because it *is* one -- it inherits every one of
those code paths untouched.

### 5.1 The one unavoidable deviation: death

A custom actor built by `build-actor` carries dummy merc-ctrl geometry (`generate_dummy_merc_ctrl`
in `build_actor.cpp` literally reuses a hardcoded dummy mesh). Letting the stock death path set
`death-timer` therefore hands the actor to the Generic Merc C++ routines, which read uninitialized
fragment memory and crash -- with no GOAL error to catch.

The `die` state reproduces the native look by hand instead (same approach as the
`jak2/features/yakow_killable` branch):

```lisp
(defbehavior crimson-blue-guard-dissolve-sequence crimson-blue-guard ()
  (sound-play "enemy-fizz")
  (let ((node-cnt (-> self node-list length)) ...)
    (dotimes (frame 60)
      ;; hide the body once the mist has taken over
      (when (= frame 5)
        (logior! (-> self draw status) (draw-control-status no-draw)))
      ;; 8 purple death sparks per frame, spread over the 38 bones with jitter
      (dotimes (j 8)
        (let ((joint-idx (rand-vu-int-count node-cnt)))
          (vector<-cspace! pos (-> self node-list data joint-idx))
          (+! (-> pos x) (rnd-float-range self -819.2 819.2))
          ...
          (merc-death-spawn 73 pos zero-vec)))
      (suspend)))
  ;; mandatory teardown, same as every native enemy death
  (send-event self 'death-end)
  (while (-> self child) (suspend))
  (cleanup-for-death self)
  (none))
```

`knocked-fatal?` -- set from `enemy-method-78` when the killing blow was a knockdown -- makes the
`die` state skip the standing-collapse animation, so a knocked-down guard dissolves lying on the
ground exactly like the native crimson-guard.

### 5.2 The other deliberate deviation: the minimap icon

Purely cosmetic, and the natural companion to the blue mesh: a blue guard should read blue on the
minimap and on the bigmap too.

The minimap never asks an actor what color it is. Every blip and every view cone is tinted from
`(-> connection class color)` — the `minimap-class-node` the icon was registered with — in both
`draw-frustum-1` and the icon draw path of `minimap.gc`. So the whole change is one new class:

```lisp
;; minimap-h.gc -- new class id at the end of the minimap-class enum
(blue-guard-frustum 71)

;; minimap.gc -- *minimap-class-list* grown 71 -> 72; a clone of `guard-frustum` (32),
;; same icon-xy, same scale, same `frustum` flag, only the tint differs
(new 'static 'minimap-class-node
  :default-position (new 'static 'vector :w 1.0)
  :flags (minimap-flag frustum)
  :name "blue-guard-frustum"
  :icon-xy (new 'static 'vector2ub :data (new 'static 'array uint8 2 #x0 #x1))
  :scale 1.0
  :color (new 'static 'rgba :r 0 :g #x40 :b #xff :a #x80)
  )
```

The subtle part is **when** the icon is claimed. `crimson-guard`'s `citizen-init!` ends with:

```lisp
(if (not (-> this minimap))
    (set! (-> this minimap) (add-icon! *minimap* this (the-as uint 32) (the-as int #f) (the-as vector #t) 0))
    )
```

— it only adds the red icon while the slot is still `#f`. So the override registers the blue icon
**before** delegating to the parent, and the parent's `if` then falls through untouched. No stock
code had to be edited. `die` `:enter` fades the icon out, mirroring what the stock `inactive` state
already does for red guards.

Guard **vehicles** get the same treatment. `vehicle-guard`'s `vehicle-method-128` registered a
hardcoded class 14 (`guard`, the plain red dot with no view cone), so a hellcat or guard-bike
flown by a blue pilot still read red. A second new class, `blue-guard` (72), retints `guard` the
same way, and the call site picks between the two on the mod flag:

```lisp
(add-icon! *minimap*
           this
           (the-as uint (if *mod-crimson-blueguard-enable* (minimap-class blue-guard) (minimap-class guard)))
           (the-as int #f)
           (the-as vector #t)
           0
           )
```

`guard-bike` and `hellcat` are the only two `vehicle-guard` subtypes, so that one call site covers
every guard vehicle in the city. The icon is registered once, at spawn — which is fine here
because the toggle's `kill-all` + `spawn-all` flush (§7.2) rebuilds the vehicle pools anyway.

## 6. The optional grenade launcher

`guard-type` is left entirely to the traffic engine, exactly as for a red guard (`0` = taser,
`1` = rifle). On top of that, a guard has a 1-in-3 chance of carrying a grenade launcher:

```lisp
(defmethod citizen-init! ((this crimson-blue-guard))
  ((method-of-type crimson-guard citizen-init!) this)
  (set! (-> this knocked-fatal?) #f)
  (set! (-> this grenade-launcher?) (zero? (rand-vu-int-count 3)))
  (set! (-> this grenade-last-time) 0)
  (none))
```

`grenade-launcher?` changes **only what `crimson-guard-method-214` spawns** -- a `vehicle-grenade`
on a ballistic arc (`traj3d-calc-initial-velocity-using-tilt`, fired from joint 14 `"blast"`)
instead of a straight `guard-shot`. Every range, state transition, animation and cooldown stays
that of a stock rifle guard, so a grenade guard is simply a red rifle guard with a different
projectile.

**Why the rate limit.** The stock `gun-shoot` state hardcodes a four-shot burst
(`(let ((gp-0 3)) (until #f ...))` in `guard.gc`), which for a launcher would mean four grenades
per engagement. `grenade-last-time` gates the projectile to one per 2 seconds; suppressed shots
fire nothing at all while the shoot animation still plays, which reads as the launcher reloading:

```lisp
(cond
  ((and (-> this grenade-launcher?) (time-elapsed? (-> this grenade-last-time) (seconds 2)))
   (set-time! (-> this grenade-last-time))
   ... spawn-projectile vehicle-grenade ...)
  ((-> this grenade-launcher?) 0)   ;; on cooldown: animation only
  (else ((method-of-type crimson-guard crimson-guard-method-214) this)))
```

`vehicle-grenade` and its `eco-canister` art-group both live in `GAME` (`guard-projectile.o`,
`eco-canister-ag.go`), so they are resident everywhere and need no DGO change.

**Deliberately not used: `guard-type` 2.** The engine does define a third pedestrian guard type,
`ped-grenade` (`traffic-engine.gc`, `guard-settings-array` index 2), but the stock `hostile` state
only dispatches on types `0` and `1` -- a `guard-type` 2 pedestrian never enters `gun-shoot` and
just stands there. Driving the grenade off a flag on top of `guard-type` 1 keeps every inherited
state transition bit-identical to the red guard's, which is exactly the constraint from §5.

## 7. The one switch: `Debug > Mods > crimson-blueguard`

The mod has a single toggle and it lives in the **unified Mods tab from `master-dev`**
(`goal_src/jak2/pc/debug/mods-menu.gc`, see
[`../tools/mods_debug_menu.md`](../tools/mods_debug_menu.md)). This branch therefore never edits
`default-menu.gc` or `default-menu-pc.gc`, and cannot collide with another mod branch's toggles.

```text
Debug > Mods > crimson-blueguard > Enable / Disable
```

`goal_src/jak2/pc/debug/crimson-blueguard-menu.gc` is `(declare-file (debug))` and is registered in
`game.gd` immediately after `mods-menu.o` (the registry must be defined before the file that calls
it). It holds nothing but the pick-func and the builder:

```lisp
(defun mod-crimson-blueguard-enable-pick ((arg0 symbol) (arg1 debug-menu-msg))
  (case arg1
    (((debug-menu-msg press))
     (set! (-> arg0 value) (not (-> arg0 value)))
     (when *traffic-manager*
       (send-event *traffic-manager* 'kill-all)
       (send-event *traffic-manager* 'spawn-all))))
  (-> arg0 value))

(mods-menu-register "crimson-blueguard" mod-crimson-blueguard-build-menu)
```

### 7.1 Where the flag lives, and why not here

`*mod-crimson-blueguard-enable*` is **defined in `engine/ai/traffic-h.gc`**, not in the menu file,
for two reasons -- both of them bugs you would otherwise hit:

1. The menu file is `(declare-file (debug))`, so it is stripped from release builds. Defining the
   flag there would leave `traffic-object-spawn` reading a symbol that was never initialised.
   Putting it in an always-loaded engine header means the symbol exists and reads `#f` at boot, so
   the mod ships **OFF by default** as the project's non-regression rule requires.
2. A `(define ...)` in a *level* DGO file is re-run every time that level loads. Defining the flag
   in `crimson-blue-guard.gc` (CWI) would silently reset it to `#f` on the next city load,
   switching the mod off mid-session.

### 7.2 Why the toggle needs `kill-all` + `spawn-all`, not `deactivate-by-type`

The traffic engine allocates each pool's processes **once** and then recycles them: `spawn-all`
creates a process through `traffic-object-spawn` only while
`(+ active-count inactive-count) < want-count`, after which `activate-from-params` just pulls an
existing handle off the inactive list. A process that already exists therefore keeps its GOAL type
-- red or blue -- forever, and `deactivate-by-type` (which only moves actives back to inactive)
would change nothing visible.

`kill-all` destroys the pooled processes (active *and* inactive, via `kill-all-inactive`) and
`spawn-all` sets `fast-spawn` and immediately re-creates them, at which point
`traffic-object-spawn` re-reads the flag and hands back the other faction. Vehicle riders come
along for free: they are children of the guard vehicles, which the same flush re-creates. Both are
pre-existing stock events, and both use a stack message block -- nothing is allocated on the heap
from the menu.

`*traffic-manager*` is `#f` outside the city, so the toggle guards on it; the next city load builds
its pools from the new flag value anyway.

## 8. Engine changes made on this branch

Everything below is append-only or a single guarded substitution. No stock behaviour changes while
the flag is off.

| File | Change |
| --- | --- |
| `goal_src/jak2/levels/city/traffic/citizen/crimson-blue-guard.gc` | **new** -- the whole entity (§5, §6). |
| `goal_src/jak2/pc/debug/crimson-blueguard-menu.gc` | **new** -- the `Debug > Mods` toggle (§7). |
| `goal_src/jak2/engine/ai/traffic-h.gc` | `(define *mod-crimson-blueguard-enable* #f)` (§7.1). |
| `goal_src/jak2/engine/ui/minimap-h.gc` | `(blue-guard-frustum 71)` and `(blue-guard 72)` appended to the `minimap-class` enum, plus a `define-extern` for `*minimap-class-list*` (§5.2). |
| `goal_src/jak2/engine/ui/minimap.gc` | `*minimap-class-list*` grown 71 -> 73 with `blue-guard-frustum` (clone of `guard-frustum` 32) and `blue-guard` (clone of `guard` 14), both blue-tinted. Dormant while the flag is off (§5.2). |
| `goal_src/jak2/levels/city/traffic/vehicle/vehicle-guard.gc` | `vehicle-method-128` picks `blue-guard` (72) over `guard` (14) while the flag is on (§5.2). |
| `goal_src/jak2/levels/city/traffic/traffic-manager.gc` | forward declarations, the guarded faction swap in `traffic-object-spawn`, and the `spawn-crimson-blue-guard-debug` REPL helper (§9). |
| `goal_src/jak2/levels/city/traffic/vehicle/vehicle-rider.gc` | `crimson-guard-rider` binds the blue rider skeleton-group while the flag is on (§4). |
| `goal_src/jak2/engine/anim/joint.gc`, `engine/level/level.gc` | generic `register-custom-art-group` / `custom-art-group-to-link?` hook so a `build-actor` art-group built with `:master-art-group` gets its animations linked at level login. Reusable infrastructure; **unused by this mod** (the blue guard keeps its animations in its own art-group), kept because it is generic tooling. |
| `goal_src/jak2/lib/project-lib.gp`, `goalc/make/Tools.cpp`, `goalc/build_actor/jak2/build_actor.{h,cpp}` | the `:native-header #t` option (§3.1) and the `build-sbk` macro. |
| `goal_src/jak2/game.gp` | `build-actor` + `goal-src` steps for the blue guard, and the `*file-entry-map*` pre-marks that stop `cgo-file` generating duplicate build steps. |
| `goal_src/jak2/dgos/game.gd` | `crimson-blueguard-menu.o` after `mods-menu.o`. |
| `goal_src/jak2/dgos/cwi.gd` | `crimson-blue-guard.o` after `guard.o`. |
| 10 level DGOs (`cas`, `dg1`, `fdb`, `fea`, `fob`, `fra`, `lwidea`, `lwideb`, `lwidec`, `pae`) | `crimson-blue-guard-ag.go` next to each existing `crimson-guard-ag.go` (§4). |

## 9. How to test

The asset half (`build-actor` + the `.fr3` bake) only needs redoing after a `.glb` change; ordinary
GOAL iteration is `(mi)` only.

```bash
task set-game-jak2
task extract          # only after a .glb change -- bakes crimson-blue-guard-lod0 into GAME.fr3
task build-release    # only after a C++ change (build_actor / Tools.cpp)
task repl             # then (mi) for GOAL-only iteration
```

Then, in game:

1. Open the debug menu and go to `Mods > crimson-blueguard`. The row should read
   `Enable / Disable` with **no** checkmark -- the mod is off, the city is full of red guards.
2. Press it. Ambient traffic is flushed and refills within a second or two; every guard on the
   street, and every guard riding a guard-bike or hellcat, is now blue.
3. Commit a crime (punch a civilian, shoot, steal a vehicle). The blue guards must raise the wanted
   level, chase, shoot and try to arrest Jak exactly like the red ones -- that is the acceptance
   test for §5.
4. Kill one on its feet, and kill one by knocking it down. Both must dissolve into purple mist,
   the knocked-down one lying on the ground (§5.1).
5. Watch a rifle guard for a while: roughly one in three lobs arcing grenades instead of firing
   bolts, about one grenade per engagement burst (§6).
6. Press the toggle again. The city must return to 100% red guards, with stock behaviour.

Single-actor inspection from the REPL, bypassing traffic density (boot into any city level first):

```lisp
(spawn-crimson-blue-guard-debug -1)   ;; weapon as ambient traffic would roll it
(spawn-crimson-blue-guard-debug 0)    ;; force the taser guard
(spawn-crimson-blue-guard-debug 1)    ;; force the rifle guard
```

Per the project's cold-boot rule, finish with `task boot-game` before concluding: hot-reload leaves
old type layouts and symbols in simulated PS2 memory, so a `deftype` change (this branch adds
three fields to `crimson-blue-guard`) can appear to work under `(mi)` and be broken on a clean
launch.

## 10. Status

| Piece | State |
| --- | --- |
| Blue mesh + 38-bone skeleton, native animation slot alignment | ✅ |
| Drawable geometry baked into `GAME.fr3` (Circuit 2) | ✅ |
| Behaviour strictly inherited from `crimson-guard` | ✅ |
| Purple death dissolution, standing and knocked-down | ✅ |
| Ambient pedestrian guards swapped | ✅ |
| Guard-vehicle riders swapped | ✅ |
| Blue minimap icon, guards on foot (`blue-guard-frustum`) | ✅ |
| Blue minimap icon, guard vehicles (`blue-guard`) | ✅ |
| Optional grenade launcher (1 in 3, rate-limited) | ✅ |
| `Debug > Mods > crimson-blueguard` toggle, live flush | ✅ |
| OFF by default, including release builds | ✅ |
| Cold-boot verification | ⏳ to be run by the maintainer |

### Out of scope on this branch

City Peaceful and City Insurrection (neutral blue patrol squads, the three-front territorial civil
war, the war-zone district picker and the `*mod-city-*-hook*` extension layer) were removed from
this branch. They live on their own branches:

- `jak2/features/crimson-blueguard/peaceful`
- `jak2/features/crimson-blueguard/city-insurrection`

Nothing on this branch should reference them again: this branch is the reskin and nothing else.


Not verified yet: a cold boot (`task boot-game`) after the last `deftype` change.

## 11. Change log

| Area | Change |
|---|---|
| New entity | `crimson-blue-guard` (`levels/city/traffic/citizen/crimson-blue-guard.gc`): blue skeleton-group, hand-reproduced purple death dissolution, optional grenade launcher. Everything else inherited from `crimson-guard`. |
| Ambient spawning | `traffic-manager.gc::traffic-object-spawn` picks the blue subtype for both city guard pools while the mod is on. Plus a `spawn-crimson-blue-guard-debug` REPL helper. |
| Vehicle guards | `vehicle-rider.gc` binds the blue rider skeleton-group for `crimson-guard-rider` while the mod is on. |
| Mod switch | `*mod-crimson-blueguard-enable*` defined in `engine/ai/traffic-h.gc`; the `Debug ▸ Mods` entry in `pc/debug/crimson-blueguard-menu.gc`. |
| Build tooling | `:native-header #t` for `build-actor` (`project-lib.gp`, `Tools.cpp`, `build_actor.{h,cpp}`); a `build-sbk` macro; a generic `register-custom-art-group` hook in `joint.gc` / `level.gc`. |
| Assets & DGOs | the two `.glb` sources; `crimson-blue-guard.o` in `cwi.gd`, `crimson-blueguard-menu.o` in `game.gd`, `crimson-blue-guard-ag.go` in the 10 level DGOs that already carry `crimson-guard-ag.go`. |
| Removed from this branch | City Peaceful / City Insurrection and the `*mod-city-*-hook*` extension layer, now only on their own branches. All stock engine files they touched (`guard.gc`, `citizen.gc`, `traffic-engine.gc`, `default-menu-pc.gc`) are back to their `master-dev` state. |

---
---

<a name="-version-française"></a>

# 🇫🇷 Version Française

## 1. Ce que c'est

Un garde crimson recoloré en bleu, ajouté comme **entité GOAL à part entière**
(`crimson-blue-guard`) plutôt que comme remplacement de texture global. Tant que le mod est activé,
il remplace le garde rouge classique (`crimson-guard`) dans tout le trafic ambiant de Haven City —
à pied comme à bord des véhicules de garde — et son comportement est **strictement hérité** :
mêmes états, même réaction d'alerte après un crime, même logique d'arrestation et de poursuite,
mêmes stats, mêmes animations, mêmes sons, même mort (§5). Le seul ajout de gameplay est qu'un
garde peut porter un lance-grenade (§6). Tout est conditionné à une unique bascule,
`Debug ▸ Mods ▸ crimson-blueguard`, désactivée par défaut (§7).

L'asset source est `custom_assets/jak2/models/custom_levels/crimson-blue-guard.glb` (aussi copié
vers `custom_assets/jak2/models/common/crimson-blue-guard-lod0.glb`, voir §4.3) : le squelette
natif décompilé de `crimson-guard` + ses 40 animations, reskinné avec un jeu de textures recoloré
dans Blender, puis réexporté.

## 2. Le problème central : les indices de slot d'animation

Les ~4700 lignes d'IA/machine à états de `crimson-guard`
(`goal_src/jak2/levels/city/traffic/citizen/guard.gc`) référencent ses animations presque
entièrement par **indice numérique de slot** dans le tableau d'éléments de son art-group — soit
via des champs surchargeables (`anim-walk`, `anim-run`, `anim-get-up-front`...) positionnés une
fois dans `init-enemy!`, soit, dans une poignée de méthodes (`enemy-method-77`, `enemy-method-78`,
`set-behavior!`), sous forme de **littéraux bruts** codés en dur directement dans le corps de la
méthode (`(-> this draw art-group data 42)` et consorts).

L'art-group natif `crimson-guard-ag` a une disposition fixe (voir
`decompiler/config/jak2/ntsc_v1/art-group-info.min.json`, clé `crimson-guard-ag`) :

| Slot | Contenu |
|---|---|
| 0 | `crimson-guard-lod0-jg` (mesh skinné) |
| 1 | `crimson-guard-lod0-mg` |
| 2 | `crimson-guard-lod2-mg` |
| 3 | `crimson-guard-shadow-mg` |
| 4..43 | 40 animations, dans un ordre fixe (`idle`@4, `walk`@5, `run`@6, ..., `get-up-front`@33, `get-up-back`@34, ...) |

L'outil `build-actor` existant (`goalc/build_actor/jak2/build_actor.cpp`) ne reproduit **pas**
cette disposition pour un acteur custom autonome : il émet toujours un header à 2 slots (`jgeo`,
un slot vide factice) avant les animations, et ordonne les animations selon leur ordre dans le
tableau `animations` du `.glb` source — qu'un export Blender/glTF normal trie alphabétiquement.
Construire le garde bleu « tel quel » aurait placé `idle` de `crimson-blue-guard-ag` au slot 2 au
lieu de 4, `get-up-back` à un slot dérivé de l'ordre alphabétique au lieu de 34, etc. — jouant
silencieusement la *mauvaise* animation dans chaque chemin de code à indice codé en dur, cassant
l'exigence de « comportement identique » de façon subtile et difficile à remarquer (par exemple
seules les réactions d'éjection de véhicule ou de choc éco-jaune, qui utilisent des littéraux
bruts, seraient fausses).

## 3. Le correctif — deux pièces additives et optionnelles

### 3.1 `build-actor :native-header #t`

La macro `build-actor` de `goal_src/jak2/lib/project-lib.gp` a reçu un nouveau paramètre
`&key (native-header #f)`, propagé jusqu'à l'outil de compilation de données `build-actor2`
(`goalc/make/Tools.cpp::BuildActor2Tool`) puis jusqu'à
`jak2::BuildActorParams2::native_anim_header` (`goalc/build_actor/jak2/build_actor.h`). Quand il
est activé, `run_build_actor` (`goalc/build_actor/jak2/build_actor.cpp`) émet **deux slots
factices supplémentaires** après le mesh, faisant passer le header de 2 à 4 slots — reproduisant
exactement la disposition native. La valeur par défaut est `#f`, donc tous les acteurs custom
existants (`test-actor`, le jetboard, etc.) restent totalement inchangés.

```lisp
(build-actor "crimson-blue-guard" :force-run #t :native-header #t)
```

### 3.2 Réordonner le tableau d'animations du `.glb` source

Un script Python ponctuel a réordonné le tableau JSON `animations` de `crimson_blue_guard.glb`
(simple réordonnancement des éléments du tableau — aucune donnée d'accessor/bufferView/mesh
touchée) pour correspondre à l'ordre canonique des 40 noms issu de
`art-group-info.min.json` ci-dessus. Combiné au header natif à 4 slots, cela fait que le slot N de
`crimson-blue-guard-ag` contient la *même* animation que le slot N de `crimson-guard-ag`, pour
tout N. Si vous devez un jour reconstruire le `.glb` depuis un nouvel export Blender, relancez
`python scripts/modding/reorder_crimson_guard_glb_anims.py <in.glb> <out.glb>` avant `build-actor`,
sinon les indices d'animation dériveront à nouveau.

Avec ces deux pièces en place, `crimson-blue-guard` n'a besoin de surcharger que `init-enemy!`
(`goal_src/jak2/levels/city/traffic/citizen/crimson-blue-guard.gc`) pour pointer vers son propre
skeleton-group par nom — toutes les autres méthodes/états hérités de `crimson-guard` continuent de
fonctionner avec exactement les mêmes indices numériques, sans modification.

## 4. Le faire apparaître dans le jeu

- **Résidence du code :** `crimson-blue-guard.gc` compile vers `crimson-blue-guard.o`, ajouté à
  côté de `guard.o` dans `goal_src/jak2/dgos/cwi.gd` (le DGO commun toujours résident qui contient
  déjà le code de `crimson-guard`).
- **Résidence de l'art :** `crimson-blue-guard-ag.go` a été ajouté à côté de chaque entrée
  `crimson-guard-ag.go` existante (ajout uniquement, rien retiré) dans les 10 DGOs de niveau qui
  le portent : `cas.gd`, `dg1.gd`, `fdb.gd`, `fea.gd`, `fob.gd`, `fra.gd`, `lwidea.gd`,
  `lwideb.gd`, `lwidec.gd`, `pae.gd`. Cela garantit que les assets de la variante bleue sont
  chargés partout où ceux du garde classique le sont, donc elle ne peut jamais être choisie pour
  un spawn sans que son art soit résident.
- **Spawn dans le trafic ambiant :** `traffic-manager.gc::traffic-object-spawn` est l'unique
  endroit où la simulation de trafic transforme un choix `(traffic-type crimson-guard-1)` /
  `(traffic-type crimson-guard-0)` en process concret, via
  `(citizen-spawn arg0 crimson-guard arg1)`. Les deux points d'appel lisent désormais le drapeau
  maître du mod :

  ```lisp
  (((traffic-type crimson-guard-1))
   (set! v0-0 (citizen-spawn arg0 (if *mod-crimson-blueguard-enable* crimson-blue-guard crimson-guard) arg1))
   )
  ```

  C'est le **seul** point de contact dans toute la simulation de trafic. L'enum `traffic-type`, la
  table de pondération `guard-type-info-array`, les `want-count` et tout le reste de la logique de
  qui / quand / où un slot de garde est choisi restent totalement intouchés — `crimson-blue-guard`
  n'est qu'un type concret alternatif pour une décision de spawn déjà existante, donc toute la
  comptabilité du traffic-engine (nav mesh, état d'alerte, comptages de population) se comporte
  identiquement quelle que soit la variante qui atterrit dans le slot de process. C'est une
  substitution totale, pas un mélange : mod activé, il ne reste aucun garde rouge dans le trafic
  ambiant ; mod désactivé, aucun bleu.
- **Gardes à bord des véhicules de garde :** les pilotes du guard-bike et du hellcat sont des
  `crimson-guard-rider` (`levels/city/traffic/vehicle/vehicle-rider.gc`), un `vehicle-rider` qui
  n'est *pas du tout* un `crimson-guard` — il emprunte seulement son skeleton-group. Le swap y
  tient donc en une chaîne de caractères : `vehicle-rider-method-32` lie
  `"skel-crimson-blue-guard-rider"` au lieu de `"skel-crimson-guard-rider"` quand le drapeau est
  actif. Les deux skeleton-groups vivent dans CWI et partagent le même squelette 38 os et la même
  numérotation de slots d'animation, donc `riding-anim` (35 / 36) et tous les autres indices
  numériques restent valides dans les deux cas. Un pilote éjecté de son véhicule réapparaît via
  `traffic-object-spawn` en `(traffic-type crimson-guard-1)`, donc il retombe sur ses pieds dans la
  bonne faction sans code supplémentaire.
- `(declare-type crimson-blue-guard crimson-guard)` a été ajouté en haut de `traffic-manager.gc`
  pour que la référence ci-dessus compile indépendamment de l'ordre des fichiers (même idiome que
  la déclaration anticipée de `crimson-guard` lui-même dans `traffic-engine.gc`).

### 4.3 Une seconde pièce, facile à manquer : la géométrie de rendu réelle (« Circuit 2 »)

`build-actor` (Circuit 1, §3) ne produit que l'art-group squelette/animations. Les triangles +
textures réellement dessinés par le renderer PC (Circuit 2) viennent d'un système totalement
séparé : le décompilateur les cuit dans des fichiers `.fr3`, recherchés **par nom** au runtime.
Voir `docs/modding/jak2_lisp_instructions.md` pour le mécanisme
complet. Le merc-ctrl produit par `build-actor` lui-même est un placeholder
(`generate_dummy_merc_ctrl` dans `build_actor.cpp` réutilise littéralement un mesh factice codé en
dur) — sans le Circuit 2, le garde apparaît, se déplace et fait du bruit normalement, mais est
**invisible**.

Le correctif : une seconde copie du même `.glb`, renommée pour correspondre au nom du merc-ctrl
placeholder (`<nom-art-group>-lod0`, ici `crimson-blue-guard-lod0.glb`), déposée dans
`custom_assets/jak2/models/common/`. `add_custom_model_to_level` du décompilateur
(`decompiler/level_extractor/extract_merc.cpp`) scanne automatiquement ce dossier à `task extract`
— aucune config nécessaire — et cuit le modèle + toutes ses textures dans `GAME.fr3` (`common` →
toujours résident, peu importe le niveau). C'est une étape ponctuelle (ou à refaire après tout
changement du `.glb`) ; elle n'est **pas** à refaire après une simple itération de code GOAL
(`(mi)`).

## 5. Comportement : identique au garde rouge, volontairement

C'est la règle de conception de la branche, et ce qu'il faut protéger en éditant
`crimson-blue-guard.gc` :

> `crimson-blue-guard` est un simple sous-type de `crimson-guard` qui **doit se comporter
> exactement comme le garde rouge d'origine**. Mêmes états, même réaction d'alerte après un crime,
> même logique d'arrestation et de poursuite, mêmes stats, mêmes animations, mêmes sons, même mort.
> Ce mod est un reskin, pas un mod d'IA.

Concrètement, le type ne surcharge que quatre choses, et chacune est soit cosmétique, soit une
contrainte technique dure des acteurs personnalisés `build-actor` :

| Surcharge | Pourquoi elle existe |
| --- | --- |
| `init-enemy!` | lie `skel-crimson-blue-guard` au lieu de `skel-crimson-guard` — le mesh bleu, c'est-à-dire tout l'objet du mod. Toutes les stats viennent toujours de `*crimson-guard-nav-enemy-info*`. |
| `citizen-init!` | enregistre l'icône de minimap bleue `blue-guard-frustum` (§5.2), puis appelle le parent et tire le lance-grenade (§6). `guard-type`, `hit-points`, collide-spec et réaction d'alerte sont entièrement laissés au parent. |
| `die` + `enemy-method-78` | reproduisent à la main la dissolution violette native (§5.1). |
| `crimson-guard-method-214` | le lance-grenade optionnel (§6). |

Il n'y a **aucune** surcharge de `general-event-handler`, aucune surcharge d'état, aucune surcharge
de méthode de ciblage. C'est délibéré : une telle surcharge serait, par définition, une différence
de comportement avec le garde rouge. Un garde bleu réagit à un crime, rejoint une alerte de ville,
tente d'arrêter Jak, change d'arme quand l'alerte monte et poursuit exactement comme un rouge,
parce qu'il *en est* un — il hérite de chacun de ces chemins de code sans modification.

### 5.1 La seule déviation inévitable : la mort

Un acteur personnalisé construit par `build-actor` embarque une géométrie merc-ctrl factice
(`generate_dummy_merc_ctrl` dans `build_actor.cpp` réutilise littéralement un mesh factice codé en
dur). Laisser le chemin de mort standard régler `death-timer` confie donc l'acteur aux routines C++
Generic Merc, qui lisent de la mémoire de fragments non initialisée et plantent — sans aucune
erreur GOAL à rattraper.

L'état `die` reproduit donc le rendu natif à la main (même approche que la branche
`jak2/features/yakow_killable`) :

```lisp
(defbehavior crimson-blue-guard-dissolve-sequence crimson-blue-guard ()
  (sound-play "enemy-fizz")
  (let ((node-cnt (-> self node-list length)) ...)
    (dotimes (frame 60)
      ;; masquer le corps une fois que la brume a pris le dessus
      (when (= frame 5)
        (logior! (-> self draw status) (draw-control-status no-draw)))
      ;; 8 étincelles de mort violettes par frame, réparties sur les 38 os avec du jitter
      (dotimes (j 8)
        (let ((joint-idx (rand-vu-int-count node-cnt)))
          (vector<-cspace! pos (-> self node-list data joint-idx))
          (+! (-> pos x) (rnd-float-range self -819.2 819.2))
          ...
          (merc-death-spawn 73 pos zero-vec)))
      (suspend)))
  ;; démontage obligatoire, comme pour toute mort d'ennemi native
  (send-event self 'death-end)
  (while (-> self child) (suspend))
  (cleanup-for-death self)
  (none))
```

`knocked-fatal?` — posé depuis `enemy-method-78` quand le coup fatal était une projection au sol —
fait sauter à l'état `die` l'animation d'effondrement debout, de sorte qu'un garde mis au sol se
dissout couché, exactement comme le crimson-guard natif.

### 5.2 L'autre déviation volontaire : l'icône de minimap

Purement cosmétique, et le complément naturel du maillage bleu : un garde bleu doit aussi se lire
bleu sur la minimap et sur la bigmap.

La minimap ne demande jamais sa couleur à l'acteur. Chaque point et chaque cône de vision est
teinté depuis `(-> connection class color)` — le `minimap-class-node` avec lequel l'icône a été
enregistrée — aussi bien dans `draw-frustum-1` que dans le chemin de dessin des icônes de
`minimap.gc`. Tout le changement tient donc dans une nouvelle classe :

```lisp
;; minimap-h.gc -- nouvel id de classe a la fin de l'enum minimap-class
(blue-guard-frustum 71)

;; minimap.gc -- *minimap-class-list* passe de 71 a 72 ; un clone de `guard-frustum` (32),
;; meme icon-xy, meme scale, meme flag `frustum`, seule la teinte change
(new 'static 'minimap-class-node
  :default-position (new 'static 'vector :w 1.0)
  :flags (minimap-flag frustum)
  :name "blue-guard-frustum"
  :icon-xy (new 'static 'vector2ub :data (new 'static 'array uint8 2 #x0 #x1))
  :scale 1.0
  :color (new 'static 'rgba :r 0 :g #x40 :b #xff :a #x80)
  )
```

Le point subtil est **le moment** où l'icône est réservée. Le `citizen-init!` de `crimson-guard`
se termine par :

```lisp
(if (not (-> this minimap))
    (set! (-> this minimap) (add-icon! *minimap* this (the-as uint 32) (the-as int #f) (the-as vector #t) 0))
    )
```

— il n'ajoute l'icône rouge que tant que le slot vaut encore `#f`. L'override enregistre donc
l'icône bleue **avant** de déléguer au parent, et le `if` du parent tombe alors dans le vide. Aucun
code d'origine n'a eu besoin d'être modifié. Le `:enter` de `die` fait disparaître l'icône, comme
l'état `inactive` d'origine le fait déjà pour les gardes rouges.

Les **véhicules** de garde reçoivent le même traitement. Le `vehicle-method-128` de `vehicle-guard`
enregistrait une classe 14 en dur (`guard`, le simple point rouge sans cône de vision) : un hellcat
ou une guard-bike pilotée par un garde bleu restait donc rouge. Une seconde nouvelle classe,
`blue-guard` (72), reteinte `guard` de la même façon, et le site d'appel choisit entre les deux
selon le drapeau du mod :

```lisp
(add-icon! *minimap*
           this
           (the-as uint (if *mod-crimson-blueguard-enable* (minimap-class blue-guard) (minimap-class guard)))
           (the-as int #f)
           (the-as vector #t)
           0
           )
```

`guard-bike` et `hellcat` sont les deux seuls sous-types de `vehicle-guard` : ce site d'appel unique
couvre donc tous les véhicules de garde de la ville. L'icône n'est enregistrée qu'une fois, au
spawn — sans conséquence ici, puisque le vidage `kill-all` + `spawn-all` de la bascule
(§7.2) reconstruit de toute façon les pools de véhicules.

## 6. Le lance-grenade optionnel

`guard-type` est entièrement laissé au moteur de trafic, exactement comme pour un garde rouge
(`0` = taser, `1` = fusil). Par-dessus, un garde a une chance sur trois de porter un lance-grenade :

```lisp
(defmethod citizen-init! ((this crimson-blue-guard))
  ((method-of-type crimson-guard citizen-init!) this)
  (set! (-> this knocked-fatal?) #f)
  (set! (-> this grenade-launcher?) (zero? (rand-vu-int-count 3)))
  (set! (-> this grenade-last-time) 0)
  (none))
```

`grenade-launcher?` ne change **que ce que `crimson-guard-method-214` fait apparaître** — une
`vehicle-grenade` sur une trajectoire balistique (`traj3d-calc-initial-velocity-using-tilt`, tirée
depuis le joint 14 `"blast"`) au lieu d'un `guard-shot` rectiligne. Toutes les portées, transitions
d'état, animations et temporisations restent celles d'un garde fusil d'origine : un garde
lance-grenade est simplement un garde fusil rouge avec un projectile différent.

**Pourquoi la limitation de cadence.** L'état `gun-shoot` d'origine code en dur une salve de quatre
tirs (`(let ((gp-0 3)) (until #f ...))` dans `guard.gc`), ce qui pour un lance-grenade voudrait
dire quatre grenades par engagement. `grenade-last-time` limite le projectile à un toutes les
2 secondes ; les tirs supprimés ne lancent rien du tout alors que l'animation de tir joue quand
même, ce qui se lit comme un rechargement du lanceur :

```lisp
(cond
  ((and (-> this grenade-launcher?) (time-elapsed? (-> this grenade-last-time) (seconds 2)))
   (set-time! (-> this grenade-last-time))
   ... spawn-projectile vehicle-grenade ...)
  ((-> this grenade-launcher?) 0)   ;; en recharge : animation uniquement
  (else ((method-of-type crimson-guard crimson-guard-method-214) this)))
```

`vehicle-grenade` et son art-group `eco-canister` vivent tous deux dans `GAME`
(`guard-projectile.o`, `eco-canister-ag.go`) : ils sont résidents partout et ne demandent aucune
modification de DGO.

**Volontairement non utilisé : `guard-type` 2.** Le moteur définit bien un troisième type de garde
piéton, `ped-grenade` (`traffic-engine.gc`, index 2 de `guard-settings-array`), mais l'état
`hostile` d'origine ne dispatche que sur les types `0` et `1` — un piéton en `guard-type` 2 n'entre
jamais dans `gun-shoot` et reste planté. Piloter la grenade par un drapeau posé au-dessus du
`guard-type` 1 garde chaque transition d'état héritée strictement identique à celle du garde rouge,
ce qui est exactement la contrainte du §5.

## 7. L'unique interrupteur : `Debug > Mods > crimson-blueguard`

Le mod n'a qu'une seule bascule, et elle vit dans l'**onglet Mods unifié de `master-dev`**
(`goal_src/jak2/pc/debug/mods-menu.gc`, voir
[`../tools/mods_debug_menu.md`](../tools/mods_debug_menu.md)). Cette branche n'édite donc jamais
`default-menu.gc` ni `default-menu-pc.gc`, et ne peut pas entrer en collision avec les bascules
d'une autre branche de mod.

```text
Debug > Mods > crimson-blueguard > Enable / Disable
```

`goal_src/jak2/pc/debug/crimson-blueguard-menu.gc` est `(declare-file (debug))` et est enregistré
dans `game.gd` juste après `mods-menu.o` (le registre doit être défini avant le fichier qui
l'appelle). Il ne contient rien d'autre que la pick-func et le builder :

```lisp
(defun mod-crimson-blueguard-enable-pick ((arg0 symbol) (arg1 debug-menu-msg))
  (case arg1
    (((debug-menu-msg press))
     (set! (-> arg0 value) (not (-> arg0 value)))
     (when *traffic-manager*
       (send-event *traffic-manager* 'kill-all)
       (send-event *traffic-manager* 'spawn-all))))
  (-> arg0 value))

(mods-menu-register "crimson-blueguard" mod-crimson-blueguard-build-menu)
```

### 7.1 Où vit le drapeau, et pourquoi pas ici

`*mod-crimson-blueguard-enable*` est **défini dans `engine/ai/traffic-h.gc`**, pas dans le fichier
de menu, pour deux raisons — deux bugs que l'on rencontrerait sinon :

1. Le fichier de menu est `(declare-file (debug))` : il est retiré des builds release. Y définir le
   drapeau laisserait `traffic-object-spawn` lire un symbole jamais initialisé. Le placer dans un
   en-tête moteur toujours chargé garantit que le symbole existe et vaut `#f` au boot, donc que le
   mod est livré **DÉSACTIVÉ par défaut**, comme l'exige la règle de non-régression du projet.
2. Un `(define ...)` dans un fichier de DGO *de niveau* est ré-exécuté à chaque chargement de ce
   niveau. Définir le drapeau dans `crimson-blue-guard.gc` (CWI) le remettrait silencieusement à
   `#f` au prochain chargement de ville, désactivant le mod en cours de session.

### 7.2 Pourquoi la bascule a besoin de `kill-all` + `spawn-all`, et pas de `deactivate-by-type`

Le moteur de trafic alloue les process de chaque pool **une seule fois** puis les recycle :
`spawn-all` ne crée un process via `traffic-object-spawn` que tant que
`(+ active-count inactive-count) < want-count`, après quoi `activate-from-params` se contente de
tirer un handle existant de la liste des inactifs. Un process déjà existant garde donc son type
GOAL — rouge ou bleu — pour toujours, et `deactivate-by-type` (qui ne fait que renvoyer les actifs
vers les inactifs) ne changerait rien de visible.

`kill-all` détruit les process du pool (actifs *et* inactifs, via `kill-all-inactive`) et
`spawn-all` positionne `fast-spawn` et les recrée immédiatement : à ce moment
`traffic-object-spawn` relit le drapeau et renvoie l'autre faction. Les pilotes de véhicules
suivent gratuitement : ce sont des enfants des véhicules de garde, que le même vidage recrée. Les
deux sont des événements d'origine préexistants, et tous deux utilisent un message block sur la
pile — rien n'est alloué sur le tas depuis le menu.

`*traffic-manager*` vaut `#f` hors de la ville, donc la bascule le teste ; de toute façon le
prochain chargement de ville construit ses pools à partir de la nouvelle valeur du drapeau.

## 8. Les changements moteur de cette branche

Tout ce qui suit est purement additif, ou une substitution unique sous condition. Aucun
comportement d'origine ne change tant que le drapeau est désactivé.

| Fichier | Changement |
| --- | --- |
| `goal_src/jak2/levels/city/traffic/citizen/crimson-blue-guard.gc` | **nouveau** — toute l'entité (§5, §6). |
| `goal_src/jak2/pc/debug/crimson-blueguard-menu.gc` | **nouveau** — la bascule `Debug > Mods` (§7). |
| `goal_src/jak2/engine/ai/traffic-h.gc` | `(define *mod-crimson-blueguard-enable* #f)` (§7.1). |
| `goal_src/jak2/engine/ui/minimap-h.gc` | `(blue-guard-frustum 71)` et `(blue-guard 72)` ajoutés à la fin de l'enum `minimap-class`, plus un `define-extern` pour `*minimap-class-list*` (§5.2). |
| `goal_src/jak2/engine/ui/minimap.gc` | `*minimap-class-list*` passe de 71 à 73 avec `blue-guard-frustum` (clone de `guard-frustum` 32) et `blue-guard` (clone de `guard` 14), tous deux teintés en bleu. Dormants tant que le drapeau est éteint (§5.2). |
| `goal_src/jak2/levels/city/traffic/vehicle/vehicle-guard.gc` | `vehicle-method-128` choisit `blue-guard` (72) plutôt que `guard` (14) quand le drapeau est actif (§5.2). |
| `goal_src/jak2/levels/city/traffic/traffic-manager.gc` | déclarations anticipées, le swap de faction sous condition dans `traffic-object-spawn`, et l'aide REPL `spawn-crimson-blue-guard-debug` (§9). |
| `goal_src/jak2/levels/city/traffic/vehicle/vehicle-rider.gc` | `crimson-guard-rider` lie le skeleton-group du pilote bleu quand le drapeau est actif (§4). |
| `goal_src/jak2/engine/anim/joint.gc`, `engine/level/level.gc` | hook générique `register-custom-art-group` / `custom-art-group-to-link?` pour qu'un art-group `build-actor` construit avec `:master-art-group` voie ses animations liées au login de niveau. Infrastructure réutilisable ; **non utilisée par ce mod** (le garde bleu garde ses animations dans son propre art-group), conservée car c'est de l'outillage générique. |
| `goal_src/jak2/lib/project-lib.gp`, `goalc/make/Tools.cpp`, `goalc/build_actor/jak2/build_actor.{h,cpp}` | l'option `:native-header #t` (§3.1) et la macro `build-sbk`. |
| `goal_src/jak2/game.gp` | les étapes `build-actor` + `goal-src` du garde bleu, et les pré-marquages `*file-entry-map*` qui empêchent `cgo-file` de générer des étapes de build en double. |
| `goal_src/jak2/dgos/game.gd` | `crimson-blueguard-menu.o` après `mods-menu.o`. |
| `goal_src/jak2/dgos/cwi.gd` | `crimson-blue-guard.o` après `guard.o`. |
| 10 DGOs de niveau (`cas`, `dg1`, `fdb`, `fea`, `fob`, `fra`, `lwidea`, `lwideb`, `lwidec`, `pae`) | `crimson-blue-guard-ag.go` à côté de chaque `crimson-guard-ag.go` existant (§4). |

## 9. Comment tester

La moitié « assets » (`build-actor` + la cuisson `.fr3`) n'a besoin d'être refaite qu'après un
changement de `.glb` ; l'itération GOAL ordinaire se fait uniquement avec `(mi)`.

```bash
task set-game-jak2
task extract          # seulement après un changement de .glb -- cuit crimson-blue-guard-lod0 dans GAME.fr3
task build-release    # seulement après un changement C++ (build_actor / Tools.cpp)
task repl             # puis (mi) pour l'itération purement GOAL
```

Puis, en jeu :

1. Ouvrir le menu debug et aller dans `Mods > crimson-blueguard`. La ligne doit afficher
   `Enable / Disable` **sans** coche — le mod est désactivé, la ville est pleine de gardes rouges.
2. Appuyer dessus. Le trafic ambiant est vidé puis se remplit en une ou deux secondes ; chaque
   garde à pied, et chaque garde pilotant un guard-bike ou un hellcat, est maintenant bleu.
3. Commettre un crime (frapper un civil, tirer, voler un véhicule). Les gardes bleus doivent faire
   monter le niveau de recherche, poursuivre, tirer et tenter d'arrêter Jak exactement comme les
   rouges — c'est le test d'acceptation du §5.
4. En tuer un debout, et en tuer un par projection au sol. Les deux doivent se dissoudre en brume
   violette, celui mis au sol restant couché (§5.1).
5. Observer un garde fusil un moment : environ un sur trois lance des grenades en cloche au lieu de
   tirer des projectiles, à raison d'une grenade par salve d'engagement (§6).
6. Rappuyer sur la bascule. La ville doit redevenir 100 % rouge, avec le comportement d'origine.

Inspection d'un acteur isolé depuis le REPL, en contournant la densité de trafic (démarrer d'abord
dans un niveau de ville) :

```lisp
(spawn-crimson-blue-guard-debug -1)   ;; arme telle que le trafic ambiant la tirerait
(spawn-crimson-blue-guard-debug 0)    ;; force le garde taser
(spawn-crimson-blue-guard-debug 1)    ;; force le garde fusil
```

Conformément à la règle de cold boot du projet, terminer par `task boot-game` avant de conclure :
le hot-reload laisse d'anciens layouts de types et symboles en mémoire PS2 simulée, donc un
changement de `deftype` (cette branche ajoute trois champs à `crimson-blue-guard`) peut sembler
fonctionner sous `(mi)` et être cassé au démarrage propre.

## 10. Statut

| Élément | État |
| --- | --- |
| Mesh bleu + squelette 38 os, alignement natif des slots d'animation | ✅ |
| Géométrie de rendu cuite dans `GAME.fr3` (Circuit 2) | ✅ |
| Comportement strictement hérité de `crimson-guard` | ✅ |
| Dissolution violette à la mort, debout et au sol | ✅ |
| Gardes piétons ambiants remplacés | ✅ |
| Pilotes des véhicules de garde remplacés | ✅ |
| Icône bleue sur minimap, gardes à pied (`blue-guard-frustum`) | ✅ |
| Icône bleue sur minimap, véhicules de garde (`blue-guard`) | ✅ |
| Lance-grenade optionnel (1 sur 3, cadence limitée) | ✅ |
| Bascule `Debug > Mods > crimson-blueguard`, vidage à chaud | ✅ |
| Désactivé par défaut, y compris en build release | ✅ |
| Vérification cold boot | ⏳ à exécuter par le mainteneur |

### Hors périmètre de cette branche

City Peaceful et City Insurrection (escouades de patrouille bleues neutres, guerre civile
territoriale à trois fronts, sélecteur de district de zone de guerre et couche d'extension
`*mod-city-*-hook*`) ont été retirés de cette branche. Ils vivent sur leurs propres branches :

- `jak2/features/crimson-blueguard/peaceful`
- `jak2/features/crimson-blueguard/city-insurrection`

Rien sur cette branche ne doit les référencer à nouveau : cette branche est le reskin, et rien
d'autre.


Pas encore vérifié : un démarrage à froid (`task boot-game`) après le dernier changement de `deftype`.

## 11. Journal des modifications

| Domaine | Changement |
|---|---|
| Nouvelle entité | `crimson-blue-guard` (`levels/city/traffic/citizen/crimson-blue-guard.gc`) : skeleton-group bleu, dissolution violette à la mort reproduite à la main, lance-grenade optionnel. Tout le reste hérité de `crimson-guard`. |
| Spawn ambiant | `traffic-manager.gc::traffic-object-spawn` choisit le sous-type bleu pour les deux pools de gardes de la ville quand le mod est actif. Plus une aide REPL `spawn-crimson-blue-guard-debug`. |
| Gardes en véhicule | `vehicle-rider.gc` lie le skeleton-group du pilote bleu pour `crimson-guard-rider` quand le mod est actif. |
| Interrupteur du mod | `*mod-crimson-blueguard-enable*` défini dans `engine/ai/traffic-h.gc` ; l'entrée `Debug ▸ Mods` dans `pc/debug/crimson-blueguard-menu.gc`. |
| Outillage de build | `:native-header #t` pour `build-actor` (`project-lib.gp`, `Tools.cpp`, `build_actor.{h,cpp}`) ; une macro `build-sbk` ; un hook générique `register-custom-art-group` dans `joint.gc` / `level.gc`. |
| Assets & DGOs | les deux sources `.glb` ; `crimson-blue-guard.o` dans `cwi.gd`, `crimson-blueguard-menu.o` dans `game.gd`, `crimson-blue-guard-ag.go` dans les 10 DGOs de niveau qui portent déjà `crimson-guard-ag.go`. |
| Retiré de cette branche | City Peaceful / City Insurrection et la couche d'extension `*mod-city-*-hook*`, désormais uniquement sur leurs propres branches. Tous les fichiers moteur d'origine qu'ils touchaient (`guard.gc`, `citizen.gc`, `traffic-engine.gc`, `default-menu-pc.gc`) sont revenus à leur état `master-dev`. |

---
*(AI-assisted)*

