# TRPG_ecs

Voice-first, lightweight 2D multiplayer living RPG with a real-time AI Dungeon Master.

- Foundation first: an ECS live world in Rust + flecs_rust, inspired by Caves of Qud (not a clone).
- `ecs-architecture.md` — the v0.4 architecture design: components, tick loop, systems, faction/history simulation, AI DM interface, data-driven content, and the expanded MVP.
- `data/` — data-driven content, JSON canonical, one entity per file. TOML is an optional authoring format sharing the same Rust structs.
  - `data/creatures/` — 16 creatures (12 villagers, 3 scavengers, 1 Mechanist hermit)
  - `data/items/` — 10 items, `data/factions/` — 3 factions, `data/mutations/` — 6 mutations
  - `data/examples/` — blacksmith in JSON and TOML, verified identical

Expanded MVP: 60x45 village map `joppa` + 40x30 ruin map `red_sand_ruin`, 5 in-game days, faction chain reactions, cross-map travel, and a legendary item.
