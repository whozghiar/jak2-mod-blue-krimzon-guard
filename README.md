# Crimson Blue Guard — Jak 2

<p align="center">
  <img src="https://img.shields.io/badge/OpenGOAL-Mod-blue.svg" alt="OpenGOAL Mod">
  <img src="https://img.shields.io/badge/Game-Jak%202-orange.svg" alt="Target Game">
  <img src="https://img.shields.io/badge/AI--assisted-Modding-purple.svg" alt="AI Assisted">
</p>

<p align="center">
  <a href="#-english-version"><b>🇬🇧 English Version</b></a> &nbsp;•&nbsp; <a href="#-version-française"><b>🇫🇷 Version Française</b></a>
</p>

---

# 🇬🇧 English Version

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

### Branch family

This branch is the reskin, and nothing else. The two gameplay variants built on top of the same
entity live on their own branches:

| Branch | Contents |
|---|---|
| **`jak2/features/crimson-blueguard/crimson-redguard-behavior`** *(this one)* | blue guards replacing red guards, identical behaviour, optional grenade launcher |
| `jak2/features/crimson-blueguard/peaceful` | **City Peaceful** — neutral blue patrol squads |
| `jak2/features/crimson-blueguard/city-insurrection` | **City Insurrection** — three-front territorial civil war |

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

The mod adds no new controls. Everything happens through one debug-menu entry:

1. Open the debug menu in-game.
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

## ✅ Compliance Checklist

- [x] **Native non-regression:** with the mod compiled but its toggle OFF, the game plays identically to stock.
- [x] **Debug ▸ Mods toggle:** registered via `(mods-menu-register "crimson-blueguard" ...)`. See [`docs/modding/tools/mods_debug_menu.md`](docs/modding/tools/mods_debug_menu.md).
- [x] **No direct `default-menu*.gc` edits.**
- [x] **Symbols prefixed** with the mod slug (`*mod-crimson-blueguard-enable*`, `mod-crimson-blueguard-*`).
- [ ] **Verified Lisp instructions** used by this mod are present in `docs/modding/jak2_lisp_instructions.md` (to land on `master-dev` via `task modding-land-doc`).
- [x] **In-code comments** on every new/overridden type, method, state, macro.
- [ ] **Cold-boot verification** (`task boot-game`) run after the last `deftype` change.

## 📝 Modding Changes Log

| Area | Change |
|---|---|
| New entity | `crimson-blue-guard` (`levels/city/traffic/citizen/crimson-blue-guard.gc`): blue skeleton-group, hand-reproduced purple death dissolution, optional grenade launcher. Everything else inherited from `crimson-guard`. |
| Ambient spawning | `traffic-manager.gc::traffic-object-spawn` picks the blue subtype for both city guard pools while the mod is on. Plus a `spawn-crimson-blue-guard-debug` REPL helper. |
| Vehicle guards | `vehicle-rider.gc` binds the blue rider skeleton-group for `crimson-guard-rider` while the mod is on. |
| Mod switch | `*mod-crimson-blueguard-enable*` defined in `engine/ai/traffic-h.gc`; the `Debug ▸ Mods` entry in `pc/debug/crimson-blueguard-menu.gc`. |
| Build tooling | `:native-header #t` for `build-actor` (`project-lib.gp`, `Tools.cpp`, `build_actor.{h,cpp}`); a `build-sbk` macro; a generic `register-custom-art-group` hook in `joint.gc` / `level.gc`. |
| Assets & DGOs | the two `.glb` sources; `crimson-blue-guard.o` in `cwi.gd`, `crimson-blueguard-menu.o` in `game.gd`, `crimson-blue-guard-ag.go` in the 10 level DGOs that already carry `crimson-guard-ag.go`. |
| Removed from this branch | City Peaceful / City Insurrection and the `*mod-city-*-hook*` extension layer, now only on their own branches. All stock engine files they touched (`guard.gc`, `citizen.gc`, `traffic-engine.gc`, `default-menu-pc.gc`) are back to their `master-dev` state. |

## 📖 Technical Documentation

For the complete technical breakdown, architecture, and developer notes, refer to:

- 📄 [`docs/modding/current_mod/blue_guard_reskin_readme.md`](docs/modding/current_mod/blue_guard_reskin_readme.md)

---

# 🇫🇷 Version Française

## 📖 Présentation du Mod

Fait passer la Garde Crimson de Haven City au bleu.

Le mod ajoute un garde crimson recoloré en bleu comme entité à part entière — un nouveau type
GOAL, `crimson-blue-guard`, sous-type du `crimson-guard` d'origine. Tant que le mod est activé, il
**remplace** les gardes rouges partout dans le trafic ambiant de la ville : à pied, et à bord des
guard-bikes et des hellcats. Désactivez-le et la ville redevient 100 % rouge.

Les gardes bleus ne sont **pas** une nouvelle faction et **pas** une nouvelle IA. Ils se comportent
exactement comme les rouges — ils réagissent à vos crimes, font monter le niveau de recherche, vous
poursuivent, tirent, tentent de vous arrêter et changent d'arme quand l'alerte monte — parce qu'ils
héritent de chacun de ces chemins de code sans modification. Le seul ajout de gameplay est que
certains portent un lance-grenade.

- **Jeu Ciblé :** Jak 2
- **Dépôt :** [`whozghiar/jak2-mod-blue-krimzon-guard`](https://github.com/whozghiar/jak2-mod-blue-krimzon-guard)
- **Slug du Mod :** `crimson-blueguard`

### Famille de branches

Cette branche est le reskin, et rien d'autre. Les deux variantes de gameplay construites sur la
même entité vivent sur leurs propres branches :

| Branche | Contenu |
|---|---|
| **`jak2/features/crimson-blueguard/crimson-redguard-behavior`** *(celle-ci)* | gardes bleus à la place des gardes rouges, comportement identique, lance-grenade optionnel |
| `jak2/features/crimson-blueguard/peaceful` | **City Peaceful** — escouades de patrouille bleues neutres |
| `jak2/features/crimson-blueguard/city-insurrection` | **City Insurrection** — guerre civile territoriale à trois fronts |

## ✨ Fonctionnalités Clés

- **Des gardes bleus à la place des rouges, partout.** Les gardes piétons et les gardes à bord des
  guard-bikes / hellcats sont tous remplacés. C'est une substitution totale, pas un mélange.
- **Exactement le même comportement que le garde rouge.** `crimson-blue-guard` est un vrai
  sous-type GOAL de `crimson-guard` : il hérite de tous ses états, stats, animations et sons —
  réaction au crime, alerte de ville, poursuite, arrestation, changement d'arme quand l'alerte
  monte, coup de crosse, roulades d'esquive, charge au taser. Aucune surcharge ne touche l'IA.
- **Une mort fidèle.** Effondrement debout ou chute au sol après projection, suivi de l'authentique
  dissolution en particules violettes — reproduite à la main, car le chemin de mort natif du moteur
  plante sur les acteurs personnalisés (voir la doc technique).
- **Bleu sur la minimap aussi.** Les gardes à pied apparaissent en point bleu avec un cône de
  vision bleu, et les véhicules de garde qu'ils pilotent en points bleus, au lieu du rouge que
  le jeu leur donne normalement — la carte se lit donc d'un coup d'œil.
- **Lance-grenade optionnel.** Environ un garde sur trois en porte un et envoie des projectiles
  `vehicle-grenade` en cloche au lieu de tirer des rafales rectilignes, depuis les mêmes portées
  d'engagement qu'un garde fusil.
- **Une seule bascule, propre.** `[L3 + SELECT] ▸ Mods ▸ crimson-blueguard ▸ Enable / Disable`, dans
  le menu Mods unifié (accessible en boot normal sans mode debug). L'activer vide et reremplit le trafic ambiant : le changement est visible
  en une ou deux secondes, dans les deux sens.
- **Désactivé par défaut, aucune régression.** Le mod est livré désactivé. Bascule désactivée, le jeu se joue exactement comme
  d'origine.

## 🚀 Guide Pas à Pas pour Lancer le Mod

### 1. Sélectionner le Jeu Actif

```bash
task set-game-jak2
```

### 2. Compilation des Binaires

Requise : l'outil `build-actor` (`goalc/build_actor/jak2/build_actor.cpp`) et le compilateur de
données `goalc` (`goalc/make/Tools.cpp`) ont tous deux reçu le flag optionnel `:native-header`
utilisé pour construire l'art-group de cet acteur.

```bash
task build-release-game
```

### 3. Extraction des Données (Assets)

Requise une fois — la géométrie de rendu et les textures réelles du garde sont cuites dans
`GAME.fr3` par le décompilateur, à partir de
`custom_assets/jak2/models/common/crimson-blue-guard-lod0.glb`. Nécessite un ISO Jak 2 légalement
dumpé.

```bash
task extract
```

Vérifiez dans le log la ligne `Adding custom model crimson-blue-guard-lod0 to common`, et l'absence
d'erreur `merc failed to find texture` pour lui. Cette étape n'est **pas** à refaire après un
simple changement de code GOAL — `(mi)` au REPL suffit. Seul un changement de `.glb` l'exige à
nouveau.

### 4. Lancer le Jeu

```bash
task boot-game
```

*(Ou via le REPL OpenGOAL avec `task repl`, puis `(mi)` et `(r)`.)*

## 🎮 Contrôles & Utilisation en Jeu

Le mod n'ajoute aucun contrôle. Tout passe par une unique entrée du menu debug :

1. Ouvrir le menu debug en jeu.
2. Aller dans `Mods ▸ crimson-blueguard`.
3. Appuyer sur `Enable / Disable`.

Les gardes de la ville sont recyclés immédiatement : en une ou deux secondes, chaque garde dans la
rue — et chaque garde pilotant un véhicule de garde — est bleu. Rappuyez pour revenir au rouge. La
bascule est utilisable n'importe où : hors de Haven City il n'y a pas de trafic ambiant à recycler,
et le prochain chargement de ville reprend l'état dans lequel vous l'avez laissée.

Ensuite, il n'y a plus qu'à jouer. Commettez un crime et les gardes bleus vous tomberont dessus
exactement comme les rouges le faisaient.

Pour tester un acteur isolé sans attendre un spawn de rue, depuis le REPL, démarré dans un niveau
de ville :

```lisp
(spawn-crimson-blue-guard-debug -1)   ;; arme telle que le trafic ambiant la tirerait
(spawn-crimson-blue-guard-debug 0)    ;; force le garde taser
(spawn-crimson-blue-guard-debug 1)    ;; force le garde fusil
```

## 🎥 Encart Vidéo Démonstrative

[![Vidéo de Démonstration](https://img.youtube.com/vi/q3wOEOJBv3A/maxresdefault.jpg)](https://youtu.be/q3wOEOJBv3A)

▶️ **[Visionner la vidéo de démonstration sur YouTube](https://youtu.be/q3wOEOJBv3A)**

## ✅ Checklist de Conformité

- [x] **Non-régression native :** mod compilé mais bascule désactivée, le jeu se joue exactement comme d'origine.
- [x] **Bascule Debug ▸ Mods :** enregistrée via `(mods-menu-register "crimson-blueguard" ...)`. Voir [`docs/modding/tools/mods_debug_menu.md`](docs/modding/tools/mods_debug_menu.md).
- [x] **Aucune édition directe de `default-menu*.gc`.**
- [x] **Symboles préfixés** par le slug du mod (`*mod-crimson-blueguard-enable*`, `mod-crimson-blueguard-*`).
- [ ] **Instructions Lisp vérifiées** utilisées par ce mod présentes dans `docs/modding/jak2_lisp_instructions.md` (à faire atterrir sur `master-dev` via `task modding-land-doc`).
- [x] **Commentaires dans le code** sur chaque type, méthode, état et macro ajouté ou surchargé.
- [ ] **Vérification cold boot** (`task boot-game`) après le dernier changement de `deftype`.

## 📝 Journal des Changements de Modding

| Domaine | Changement |
|---|---|
| Nouvelle entité | `crimson-blue-guard` (`levels/city/traffic/citizen/crimson-blue-guard.gc`) : skeleton-group bleu, dissolution violette à la mort reproduite à la main, lance-grenade optionnel. Tout le reste hérité de `crimson-guard`. |
| Spawn ambiant | `traffic-manager.gc::traffic-object-spawn` choisit le sous-type bleu pour les deux pools de gardes de la ville quand le mod est actif. Plus une aide REPL `spawn-crimson-blue-guard-debug`. |
| Gardes en véhicule | `vehicle-rider.gc` lie le skeleton-group du pilote bleu pour `crimson-guard-rider` quand le mod est actif. |
| Interrupteur du mod | `*mod-crimson-blueguard-enable*` défini dans `engine/ai/traffic-h.gc` ; l'entrée `Debug ▸ Mods` dans `pc/debug/crimson-blueguard-menu.gc`. |
| Outillage de build | `:native-header #t` pour `build-actor` (`project-lib.gp`, `Tools.cpp`, `build_actor.{h,cpp}`) ; une macro `build-sbk` ; un hook générique `register-custom-art-group` dans `joint.gc` / `level.gc`. |
| Assets & DGOs | les deux sources `.glb` ; `crimson-blue-guard.o` dans `cwi.gd`, `crimson-blueguard-menu.o` dans `game.gd`, `crimson-blue-guard-ag.go` dans les 10 DGOs de niveau qui portent déjà `crimson-guard-ag.go`. |
| Retiré de cette branche | City Peaceful / City Insurrection et la couche d'extension `*mod-city-*-hook*`, désormais uniquement sur leurs propres branches. Tous les fichiers moteur d'origine qu'ils touchaient (`guard.gc`, `citizen.gc`, `traffic-engine.gc`, `default-menu-pc.gc`) sont revenus à leur état `master-dev`. |

## 📖 Documentation Technique

Pour l'audit technique approfondi, l'architecture et les détails d'implémentation, consultez :

- 📄 [`docs/modding/current_mod/blue_guard_reskin_readme.md`](docs/modding/current_mod/blue_guard_reskin_readme.md)

---
*(AI-assisted)*
