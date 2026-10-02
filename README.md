# Blue Krimzon Guard — Jak 2

<p align="center">
  <img src="https://img.shields.io/badge/OpenGOAL-Mod-blue.svg" alt="OpenGOAL Mod">
  <img src="https://img.shields.io/badge/Game-Jak%202-orange.svg" alt="Target Game">
  <img src="https://img.shields.io/badge/AI--assisted-Modding-purple.svg" alt="AI Assisted">
</p>

---

> [!NOTE]
> This mod moved from the `jak2/features/blue-krimzon-guard` branch of [whozghiar/jak-project](https://github.com/whozghiar/jak-project) to this repository. Earlier releases stay installable from the launcher catalog.

## 📖 Overview

Turns Haven City's Crimson Guard blue.

The mod adds a blue-recolored Crimson Guard as its own standalone entity — a new GOAL type,
`crimson-blue-guard`, that is a subtype of the stock `crimson-guard`. While the mod is enabled it
**replaces** the red guards everywhere in the city's ambient traffic: on foot, and riding the
guard bikes and hellcats. Turn it off and the city goes back to 100% red.

The blue guards are **not** a new faction and **not** a new AI. They behave exactly like the red
ones — they react to your crimes, raise the wanted level, chase you, shoot, try to arrest you and
change weapons as the alert escalates — because they inherit every one of those code paths
untouched. The only gameplay addition is that some of them carry a grenade launcher.

- **Target Game:** Jak 2
- **Repository:** [`whozghiar/jak2-mod-blue-krimzon-guard`](https://github.com/whozghiar/jak2-mod-blue-krimzon-guard)
- **Mod Slug:** `crimson-blueguard`

### Mod family

This repository is the reskin, and nothing else. The two gameplay variants built on top of the same
entity live in their own repositories:

| Repository | Contents |
|---|---|
| **`whozghiar/jak2-mod-blue-krimzon-guard`** *(this one)* | blue guards replacing red guards, identical behaviour, optional grenade launcher |
| [`whozghiar/jak2-mod-peaceful-haven-city`](https://github.com/whozghiar/jak2-mod-peaceful-haven-city) | **City Peaceful** — neutral blue patrol squads |
| [`whozghiar/jak2-mod-haven-city-rebellion`](https://github.com/whozghiar/jak2-mod-haven-city-rebellion) | **City Insurrection** — three-front territorial civil war |

## ✨ Key Features

- **Blue guards in place of red ones, everywhere.** Pedestrian guards and the guards riding the
  guard-bike / hellcat are all swapped. It is a full substitution, not a mix.
- **Exactly the same behaviour as the red guard.** `crimson-blue-guard` is a real GOAL subtype of
  `crimson-guard`, so it inherits every state, stat, animation and sound: crime reaction, city
  alert, pursuit, arrest, weapon change on alert escalation, rifle-butt melee, evasive rolls,
  taser charge. No override touches the AI.
- **Faithful death.** Standing collapse or ground knockdown, followed by the authentic purple
  particle dissolution — hand-reproduced, because the engine's native death path crashes on custom
  actors (see the technical doc).
- **Blue on the minimap too.** Guards on foot show up as a blue blip with a blue view cone,
  and the guard vehicles they fly as blue dots, instead of the red ones the game normally
  gives them — so you can read the city map at a glance.
- **Optional grenade launcher.** Roughly one guard in three carries one and lobs `vehicle-grenade`
  projectiles on a ballistic arc instead of firing straight bolts, from the same engagement
  ranges as a rifle guard.
- **One clean toggle.** `[L3 + SELECT] ▸ Mods ▸ crimson-blueguard ▸ Enable / Disable`, in the unified Mods
  menu (operable in retail boot without debug mode). Flipping it flushes and refills ambient traffic, so the swap is visible within a second or
  two, both ways.
- **Off by default, no regression.** The mod ships disabled. With the toggle off, the game plays exactly like stock.

## 🚀 Step-by-Step Guide to Run the Mod

### 1. Select the Active Game

```bash
task set-game-jak2
```

### 2. Binary Compilation

Required: the `build-actor` tool (`goalc/build_actor/jak2/build_actor.cpp`) and the `goalc`
data-compiler (`goalc/make/Tools.cpp`) both gained the opt-in `:native-header` flag used to build
this actor's art-group.

```bash
task build-release-game
```

### 3. Asset Extraction

Required once — the guard's actual drawable geometry and textures are baked into `GAME.fr3` by the
decompiler, from `custom_assets/jak2/models/common/crimson-blue-guard-lod0.glb`. Needs a
legally-dumped Jak 2 ISO.

```bash
task extract
```

Check the log for `Adding custom model crimson-blue-guard-lod0 to common`, and for no
`merc failed to find texture` error on it. This step does **not** need repeating after a pure
GOAL-code change — `(mi)` in the REPL is enough. Only a `.glb` change requires it again.

### 4. Launch the Game

```bash
task boot-game
```

*(Or launch via the OpenGOAL REPL with `task repl`, then `(mi)` and `(r)`.)*

## 🎮 Controls & Gameplay Usage

The mod adds no new controls. Everything happens through one Mods-menu entry:

1. Press `L3 + SELECT` in-game to open the Mods menu (works in a normal launcher boot, no debug mode needed).
2. Go to `Mods ▸ crimson-blueguard`.
3. Press `Enable / Disable`.

The city guards are recycled immediately, so within a second or two every guard on the street —
and every guard riding a guard vehicle — is blue. Press it again to go back to red. The toggle is
safe to use anywhere: outside Haven City there is no ambient traffic to recycle, and the next city
load picks up whichever state you left it in.

From there, just play. Commit a crime and the blue guards will come after you exactly like the red
ones did.

For testing a single actor without waiting for a street spawn, from the REPL while booted into any
city level:

```lisp
(spawn-crimson-blue-guard-debug -1)   ;; weapon as ambient traffic would roll it
(spawn-crimson-blue-guard-debug 0)    ;; force the taser guard
(spawn-crimson-blue-guard-debug 1)    ;; force the rifle guard
```

## 🎥 Demonstration Video

[![Demonstration Video](https://img.youtube.com/vi/q3wOEOJBv3A/maxresdefault.jpg)](https://youtu.be/q3wOEOJBv3A)

▶️ **[Watch the demonstration video on YouTube](https://youtu.be/q3wOEOJBv3A)**

## 📖 Technical Documentation

For the complete technical breakdown, architecture, and developer notes, refer to:

- 📄 [`docs/modding/current_mod/blue_guard_reskin_readme.md`](docs/modding/current_mod/blue_guard_reskin_readme.md)

---
*(AI-assisted)*
