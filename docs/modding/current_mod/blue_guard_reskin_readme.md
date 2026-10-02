# Jak 2 — Blue Crimson Guard Reskin (`crimson-blue-guard`)

> **Mod Readme**
>
> - **Branch:** `jak2/features/crimson-blueguard/crimson-redguard-behavior`
> - **Type:** `features`
> - **Depends on:** the existing `build-actor` custom-actor pipeline
>   (`goal_src/jak2/lib/project-lib.gp`, `goalc/build_actor/`)

---

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
*(AI-assisted)*

