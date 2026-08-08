---
title: "How I Set Up Godot (and a Claude Code Project) on a Beat-Up ThinkPad"
date: 2026-08-08
draft: false
tags: ["gamedev", "godot", "claude-code", "indie-dev"]
---

I haven't written a single line of gameplay code yet. No player sprite, no jump, nothing you could screenshot and call a game. What I do have is a Godot project that opens correctly, a plan I trust enough to hand to an AI coding assistant, and a decent case for why that matters more than it sounds like it should. This is the planning-and-setup post, not the "look what I built" post — that one comes later.

The project: a 2D metroidvania, solo, built in my spare time on a ThinkPad T480 that's a few years past its prime.

## Why Godot, why GDScript

I'm new to Godot, so this wasn't an exhaustive comparison of every engine out there — it came down to two things. First, as a solo dev I don't have a team to absorb licensing costs or a steep learning curve, and Godot is free, open, and has a solid reputation for 2D specifically. Second, and this mattered more than I expected going in: I'm planning to lean on Claude Code for a lot of the actual implementation, and Godot's project files are just text. Scenes, scripts, the project config — all of it is readable and diffable, which means an AI assistant can actually reason about it instead of poking at an opaque binary format.

That same logic pushed me toward GDScript over C#. I'm a beginner in Godot either way, and GDScript's syntax is a smaller lift than C#. It also has better support in the AI-tooling ecosystem around Godot right now — most of the assistant integrations people have built are GDScript-first, with C# support lagging behind. No reason to fight that as a beginner.

## The decisions I made before touching a single scene

This is the part I almost skipped, and I'm glad I didn't. A few architecture calls, settled before any actual code existed:

- **Renderer: Compatibility, not Forward+.** Forward+ is Godot's fancy default, built for advanced 3D lighting I'm never going to use in a 2D game. My T480 runs on integrated graphics, not a discrete GPU — fine for everyday use, but there's no reason to make it work harder than it needs to. Compatibility mode is the deliberately un-fancy renderer, and it's the right call for exactly this situation: 2D game, older hardware.
- **Rooms as separate scenes, not one giant map.** I could build the whole world as one continuous scene, but I went with the classic approach instead: each room is its own scene file, connected by transition triggers. Easier for me to reason about one room at a time, and — not a small thing — easier to hand "build room 3" to an AI assistant as a self-contained task.
- **One singleton to rule the save state.** Every ability I unlock, every boss I beat, every item I pick up needs to live somewhere that isn't tangled into a specific room or script. I set up a single autoload — I'm calling it `Progress` — as the one place that state lives. Every locked door or gated path checks a flag on `Progress` rather than hardcoding logic per room.

None of this is exotic. It's the boring, standard way to structure a metroidvania. But writing it down *before* generating any code meant I had actual answers when decisions came up mid-build, instead of improvising and regretting it three rooms later.

## Turning the plan into a file an AI assistant can actually use

All of the above went into a `CLAUDE.md` file — the file Claude Code reads automatically before it touches a project. The point isn't documentation for future-me so much as: I only have to explain "here's how this project is structured" once, instead of re-explaining it in every prompt.

The build order I landed on, straight from that file:

```markdown
- [ ] Player controller (movement + jump feel, in one bare test room)
- [ ] First ability (e.g. double jump) + the Progress flag + one gated obstacle
- [ ] Room transition system connecting two rooms
- [ ] First enemy with basic health/damage loop
- [ ] Save/load
- [ ] (only after the above) expand content: more rooms, abilities, enemies
- [ ] Minimap, polish, juice
```

That ordering matters more than it looks like it should. It's a vertical slice — the whole game in miniature, working end to end, before I let myself (or Claude Code) generate any more content. It's tempting to build ten rooms first because rooms feel like "real progress." I'm trying to resist that.

The actual project scaffold that came out of this planning session:

```text
scenes/
  rooms/      one .tscn per room/level chunk
  player/     player scene + sub-scenes
  enemies/    one .tscn per enemy type
  ui/         HUD, menus, minimap
scripts/
  player/
  enemies/
  systems/    shared systems (save/load, room transitions, etc.)
autoload/     singletons registered in Project Settings > Autoload
assets/       sprites, audio, tilesets
addons/
```

And the `Progress` singleton itself — currently just flags and a couple of helper methods, nothing fancy yet, but it's the thing every future system will read from:

```gdscript
extends Node
## Autoload singleton (registered in project.godot as "Progress").

# --- Ability flags ---
var has_double_jump: bool = false
var has_dash: bool = false
var has_wall_climb: bool = false

# --- World state ---
var visited_rooms: Array[String] = []
var defeated_bosses: Array[String] = []
var inventory: Dictionary = {}


func unlock_ability(ability_name: String) -> void:
	if ability_name in self:
		set(ability_name, true)


func mark_room_visited(room_id: String) -> void:
	if room_id not in visited_rooms:
		visited_rooms.append(room_id)
```

And `project.godot` got its renderer pinned up front, so I never accidentally build something that only looks right on better hardware than mine:

```ini
[rendering]

renderer/rendering_method="gl_compatibility"
renderer/rendering_method.mobile="gl_compatibility"
```

## Actually getting Godot running on the T480

Here's the part where I skipped the thorough approach. I had a whole walkthrough for installing the native Linux build and checking graphics drivers properly — and I skipped all of it. I just installed Godot through Steam, because it was simpler. No regrets there.

I did check the part that actually mattered, though: I dropped the project scaffold into the Steam-installed Godot, imported it, and confirmed the Compatibility renderer setting and the `Progress` autoload were both registered correctly. Plan-on-paper matching what's actually sitting in the editor — that's the bar I cared about, not which install method got me there.

## Where I stopped

I have not run `git init` on this project yet. That's deliberate, not an oversight — right now the repo is a folder structure and one singleton stub, and that didn't feel like a "commit" yet. I'll start version control once there's an actual first system working, so the first commit means something.

I already had Claude Code installed and working on this machine, so there was no setup step to document there — just a matter of pointing it at this project folder.

## What's next

The very next thing is handing Claude Code its first real task: a bare-bones player controller — movement, jump, gravity, nothing else — in an empty test room, with instructions to read `CLAUDE.md` first and propose its approach before writing anything. That's the next post, once there's something to actually show.
