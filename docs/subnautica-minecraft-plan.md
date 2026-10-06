# Subnautica-in-Minecraft: Project Plan

Status: draft for review. Nothing below is built yet.

The goal is a Minecraft experience that feels like Subnautica: a lone survivor
in a deep alien ocean, swimming with weight and momentum, surrounded by
schooling fish and stalking predators, with the world organised by depth. It
is a re-creation in Minecraft's own blocky style, not a port of Subnautica's
assets.

The four work streams from the original pitch are kept, reordered by
dependency, and a fifth (reference extraction) is added because the other
four lean on it.

| # | Stream | Output |
|---|--------|--------|
| 0 | Reference extraction | A spec document of numbers and behaviours taken from the game, no code or assets |
| 1 | Core mod | Swimming, oxygen and depth, creature framework, creatures, flora, surface waves |
| 2 | World generation | Ocean-only world with depth-banded biomes, lifepod spawn, wreck structures |
| 3 | Modpack | Packwiz pack: core mod + curated third-party mods + shaders + configs |
| 4 | Immersion mods | Small companion mods only where no existing mod covers a gap |

---

## 1. Decisions to lock first

These change the shape of everything else, so they come before any code.

### 1.1 Mod loader and Minecraft version

Recommendation: **Fabric**, Java 21, one pinned Minecraft version from the
1.21 line that GeckoLib, Sodium and Iris all support on the day we start.

Why Fabric over NeoForge:

- Faster compile and launch cycle, which matters when iterating on AI.
- The performance and shader mods the pack needs (Sodium, Iris, Lithium) are
  Fabric-first.
- Mixins are first-class, and the swimming overhaul is mostly mixins.

Do not use a multi-loader layer (Architectury) at the start. Add it later only
if a NeoForge build is actually requested.

Pin the version and do not chase Minecraft updates until the MVP milestone is
done. Version churn is the most common reason mod projects stall.

### 1.2 Animation library

Recommendation: **GeckoLib** (AzureLib is the fallback if GeckoLib lags a
Minecraft release). Both consume Blockbench's Bedrock-style geometry and
animation JSON directly, which is what makes the Blockbench pipeline in
section 3 work.

### 1.3 Original names and designs

Subnautica's code, models, textures, sounds and animation data are
copyrighted by Unknown Worlds. Reading them for reference is fine. Shipping
any of it, or a mechanical conversion of it, is not, and it will get the mod
removed from CurseForge or Modrinth.

Recommendation:

- Creature archetypes are kept (small herbivore that is also an oxygen
  source, metal-stealing stalker, ambush sand predator, leviathan).
- Give them original names and Minecraft-styled original designs. If you
  want to keep Subnautica names for a private pack, that is your call, but
  the public build should use its own.
- Market it as "inspired by Subnautica", never as Subnautica.

### 1.4 Scope of the MVP

Subnautica has far more systems than the four named in the pitch. Deciding
what is out of the MVP is what keeps it finishable.

In the MVP: swimming feel, oxygen and depth, schooling fish, two predators,
one flora set, ocean world generation with three depth bands, lifepod spawn,
surface waves via shaders.

Out of the MVP, in later milestones: vehicles (Seamoth-style), base building
beyond vanilla blocks, scanner and blueprint progression, thirst, story,
leviathans.

---

## 2. Stream 0: Reference extraction

The pitch says "decomp of Subnautica". Here is what that realistically gives
us and how to use it safely.

Subnautica is a Unity game. Its logic is in `Assembly-CSharp.dll`, readable
with ILSpy or dnSpy. Its models, animations and audio are in Unity asset
bundles, viewable with AssetRipper and then Blender.

What is worth pulling out, as **numbers and descriptions in a spec file**:

- Player: swim speed, sprint multiplier, acceleration and drag, turn rate,
  oxygen durations by depth, depth thresholds for pressure damage, fin and
  tank modifiers.
- Each creature: swim speed, turn speed, aggression radius, flee radius,
  school size and spacing, preferred depth band, day/night behaviour, damage.
- Flocking parameters: separation, alignment, cohesion weights and radii.
- Biome depth bands and what spawns in each.
- Animation timing: how long a swim cycle is, how fast a fish flicks when
  fleeing, what the idle hover looks like.

What not to do: copy C# code into Java, or export animation curves and feed
them into Blockbench as-is. Write the spec, then implement from the spec.
For a hobby project you do not need a strict two-person clean room, but the
"spec in between" habit is what keeps the result yours.

You own the game copy and the machine it runs on, so this stream is yours to
run. I can give you a checklist of exactly which classes and fields to look
for once the loader and version are chosen.

---

## 3. Stream 1: Core mod

### 3.1 Swimming overhaul

Vanilla Minecraft swimming is binary: you are either sprint-swimming
horizontally or bobbing. Subnautica swimming is a free 3D body with momentum.

Implementation plan:

- Mixin into the player's water travel to replace the movement vector with a
  look-direction thrust plus velocity, drag and a small buoyancy term.
- Pitch controls ascend and descend. Jump and sneak become fine vertical
  adjustments, not the main way to move.
- Remove the vanilla "dolphin dive" sprint pose in favour of a continuous
  speed boost with a stamina cost.
- Camera: optional slight roll on turning and a short lag, config-gated,
  since some players find it nauseating.
- Items: fins (speed), tanks (max oxygen), a seaglide-style handheld
  propulsion item (fast travel, battery).

Oxygen and depth:

- Replace the vanilla air bar with a mod oxygen value. Max oxygen comes from
  base plus equipped tanks. Oxygen drains faster below configured depth
  thresholds, matching the reference spec.
- Depth is measured from the world's sea level, which stream 2 raises.
- Pressure damage below the current suit rating, so depth is a progression
  gate, as in the original.
- Oxygen-giving flora (brain-coral-style bubble emitters) refill on contact.

### 3.2 Creature framework (the biggest time saver)

Do not write one Java class per fish. Write one generic swimming entity whose
behaviour is composed from a JSON definition, loaded as a datapack resource.

A creature definition holds:

- Movement: speed, turn rate, vertical freedom, depth band, idle hover.
- Social: schooling on or off, school size, spacing, leader-follow or full
  boids.
- Threat response: flee radius, flee speed multiplier, hide-in-flora flag.
- Predation: target tags, aggression radius, bite damage, cooldown, ambush
  (buried until triggered) flag.
- Loot and utility: drops, is-edible, oxygen-on-eat, cookable.
- Presentation: GeckoLib model, texture, animation set, sound set, scale.
- Spawn: biome tags, depth range, weight, group size, time of day.

Behaviour is built from reusable AI goals: `SwimWanderGoal`,
`SchoolGoal`, `FleeGoal`, `HuntGoal`, `AmbushGoal`, `StealItemGoal`,
`SurfaceForAirGoal`. A new fish is a JSON file, a Blockbench model and a
texture. No Java.

Schooling: use leader-plus-followers rather than full all-pairs boids.
Followers steer toward an offset around the leader with separation from
their nearest neighbours only. It looks nearly identical, costs a fraction,
and scales to many schools in view. Cap neighbour queries per tick and reuse
the result for several ticks.

Movement control: write a custom `MoveControl` that steers in full 3D with
smooth yaw and pitch, modelled on vanilla's dolphin and axolotl controls but
with configurable turn rates so a leviathan turns slowly and a small fish
flicks.

### 3.3 Creature roster for the MVP

Archetypes, with Minecraft-style original designs:

| Archetype | Role | Behaviour |
|-----------|------|-----------|
| Small schooling fish (two species) | Food, oxygen source, ambience | Schooling, flee, edible |
| Bladder-style fish | Crafting into a disposable oxygen item | Solo wander, slow |
| Metal-stealing stalker | Early predator in kelp | Hunts player and fish, steals dropped metal items, drops teeth |
| Sand ambusher | Mid-depth threat | Buried idle, bursts out within radius, slow when exposed |
| Bubble-emitting coral | Oxygen refill | Block entity, particle stream |
| Kelp-style tall plant | Biome identity, materials | Multi-block plant with sway |

Leviathans come after the MVP. They need custom hitboxes, grab animations
and sound design, and they are not needed to prove the concept.

### 3.4 Surface waves

This is the riskiest item and should be done last.

Minecraft's water surface is a flat quad per block. Options:

1. **Shader pack (MVP choice).** Iris plus a wave-capable shader pack
   (Complementary, Bliss, or a trimmed custom pack) gives moving surface
   displacement, foam and underwater fog with zero mod code. Bundle it in
   the modpack with our fog and colour settings.
2. **Custom fluid renderer (later).** Mixin into the fluid renderer to
   displace surface vertices by a wave function. Works without shaders but
   conflicts with Sodium's renderer, so it needs a Sodium-specific path.
3. **Gameplay waves.** Subnautica's waves are mostly visual. A light
   surface current that nudges a floating player is a small addition if
   wanted.

Vanilla underwater fog colour and distance per biome can be set in the
biome JSON today, which gets most of the atmosphere without any rendering
code.

### 3.5 Blockbench pipeline

The pitch wants Subnautica's animation feel on blocky models. Blockbench
cannot import Unity animations and the rigs do not match, so the pipeline
is:

1. View the reference animation in Blender (via AssetRipper export) or a
   recording, and write down timing, amplitude and the bone chain that moves.
2. Build the Blockbench model with a short bone chain matching that
   description (head, body, tail1, tail2, fins).
3. Hand-animate in Blockbench: swim, idle, flee, bite, death. Export the
   Bedrock animation JSON that GeckoLib loads.

Ease-of-development idea: a small Blender script that reads the reference
rig and prints a per-bone summary (rotation range per axis, cycle length)
so step 1 is mechanical. It outputs numbers, not keyframes, which keeps the
line in section 1.3.

A second idea: a Node or Python script that scaffolds a new creature from a
template (JSON definition, empty bbmodel with the standard bone chain,
placeholder texture, lang entry). New fish in one command.

---

## 4. Stream 2: World generation

Recommendation: a **custom dimension via datapack**, not a hand-built map.
A dimension lets us set sea level high (around y=200 of a 384-block world)
so there is roughly 260 blocks of depth to layer biomes in. The overworld
stays untouched, which keeps the pack compatible with other mods.

Biome layout by depth, each a vanilla-format biome JSON with its own fog
colour, water colour, spawn list and features:

| Depth band (blocks below surface) | Biome | Identity |
|----------------------------------|-------|----------|
| 0 to 40 | Shallows | Sand, light, small schooling fish, coral, lifepod spawn |
| 20 to 80 | Kelp forest | Tall kelp-style plants, stalker predator, dim green fog |
| 60 to 140 | Plateaus and mushroom field | Open cliffs, sand ambusher, big flora |
| 120 to 220 | Deep reef | Low light, bioluminescent flora, later leviathans |
| 200 to floor | Trenches and vents | Dark, hot, geyser blocks, endgame materials |

A noise-based crater shape (deep in the middle, shallow rim that drops into
a void-like dead zone) is doable with a custom density function, and it gives
the "edge of the map" dread without an invisible wall.

Hand-placed content goes in as jigsaw structures: the lifepod at spawn, a
crashed-ship-style wreck, scattered debris with loot. Structures are easier
to iterate on in a creative test world with structure blocks than in a
painted map, and they work in every new world.

A hand-built map (WorldPainter or Axiom) remains an option for a showcase
world, but it should sit on top of the datapack generation, not replace it.

---

## 5. Stream 3: Modpack

Tooling: **Packwiz**. It stores the pack as text files in git, pins exact mod
versions and hashes, and exports to both Modrinth and CurseForge formats.

Candidate mods, all to be re-checked for the pinned version at setup time:

Performance and rendering

- Sodium, Lithium, FerriteCore, Entity Culling (many fish on screen).
- Iris plus the chosen wave shader pack.

Ambience

- Sound Physics Remastered (underwater reverb is a large part of the feel).
- AmbientSounds, Presence Footsteps or an equivalent.
- Dynamic lighting for handheld light sources underwater.

Player presence

- First-person model mod so you see your own body and fins.
- Not Enough Animations or similar for swimming poses.

Survival texture

- A thirst mod if thirst makes the cut.
- A simple base-building or furniture mod for interior decoration.

Utilities

- Mod Menu, Cloth Config, a minimap or compass mod (depth readout matters
  more than a map).

Rules for inclusion: every mod must be redistributable, server-compatible
where it has server logic, and must not change ocean generation. Configs for
every mod live in the pack so defaults are curated.

---

## 6. Stream 4: Immersion companion mods

Only build what the pack cannot source. Likely candidates once the MVP is
running:

- Depth and oxygen HUD in the Subnautica style (small, could live in the
  core mod).
- Scanner tool and knowledge entries (replaces the vanilla recipe book for
  mod items).
- Surface-weather effects over the ocean (rain, lightning on the surface
  only).
- Custom fluid renderer for waves without shaders (section 3.4, option 2).

---

## 7. Milestones

| Milestone | Contents | Done when |
|-----------|----------|-----------|
| M0 Foundation | Repo layout, Fabric template, GeckoLib dependency, CI build, flat-ocean dev world preset | `gradle build` passes in CI and the mod loads |
| M1 Swimming | Movement mixins, oxygen system, depth damage, fins and tanks | Playable swim loop that feels different from vanilla |
| M2 Creatures | Generic entity, JSON loader, goal set, two schooling fish and the stalker | Schools form, flee, and the stalker hunts |
| M3 World | Custom dimension, three biomes, lifepod structure, spawn rules | New world drops you in a lifepod over the shallows |
| M4 Pack | Packwiz pack, shaders and wave settings, configs, first playtest build | A friend can install the pack from one file |
| M5 Depth | Sand ambusher, bubble coral, kelp, deep reef biome, sounds | Progression by depth works end to end |
| Later | Leviathans, vehicles, scanner, base building, custom wave renderer | Scoped one at a time after M5 |

---

## 8. What you need to provide

Decisions

1. Confirm Fabric and the Minecraft version, or state a different preference.
2. Mod id and display name (original, not "Subnautica").
3. Public release or private pack, and a licence for the code (MIT or LGPL
   are usual for Minecraft mods).
4. Which MVP items from section 1.4 you want to add or cut.

Reference material (from your own copy of the game, as a spec, per section 2)

5. Player swim and oxygen numbers.
6. Creature stat table for the MVP roster.
7. Flocking parameters.
8. Notes or short recordings of the animations you want to match.

Art

9. Who is doing models and textures. I can generate placeholder Blockbench
   geometry and flat textures programmatically so the code is never blocked
   on art, but final textures and hand animation need a person in Blockbench.
10. A palette and style reference (Minecraft vanilla, or a resource pack
    style to match).

Infrastructure

11. Whether this repository is the home for the mod. It currently holds an
    unrelated clicker game in `index.html`. Either a new repository or a
    `mod/` and `modpack/` layout alongside it both work.

---

## 9. Ideas for refinement and faster development

- **Data-driven everything.** Creatures, flora, biome fog and spawn rules all
  in JSON. Most balancing happens without a rebuild.
- **Scaffold scripts.** One command creates a new creature with all its files
  from a template.
- **Dev world preset.** A flat, shallow, well-lit ocean with every creature
  spawnable from a debug item. Testing AI in generated worlds wastes time.
- **GameTest coverage for AI.** Fabric's GameTest framework can spawn a
  school and assert it stays together, or spawn a predator and assert it
  reaches a target. Catches AI regressions in CI.
- **Learn from open sources, not from Subnautica's code.** Vanilla
  `Dolphin`, `Squid` and `Axolotl` movement, and permissively licensed mods
  with good swimming AI, show the Minecraft-idiomatic way to do what the
  spec describes.
- **Shaders first for waves.** Ship the look in M4 with a shader pack and
  only write a renderer if players without shaders ask.
- **Leader-follower schooling** instead of all-pairs boids, as above.
- **Treat depth as the progression axis** and design every creature, item
  and biome around which depth it unlocks. This is the thing that makes
  Subnautica feel like Subnautica, more than any single fish.
- **Pin versions and hashes** with Packwiz from day one so playtest builds
  are reproducible.
- **Silence is content.** Subnautica's dread is mostly audio. Budget time for
  sound design per biome and per creature early, not as polish.

---

## 10. What I can do in this environment

Java 21, Gradle and network access to the Fabric and Modrinth servers are
available here, so I can carry out M0 (scaffold, CI, dev world preset) and the
code side of M1 to M3 directly, including placeholder models and textures. I
cannot run the game client with a display, so visual checks of animations and
waves are yours; GameTests cover what can be verified headlessly.
