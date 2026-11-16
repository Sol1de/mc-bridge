# mc-bridge

Minecraft façade for native code running inside the game's JVM (Windows x64, Java 21), built on
[jvm-link](https://github.com/Sol1de/jvm-link) and [mc-mappings](https://github.com/Sol1de/mc-mappings).

**Status:** API catalog revised before implementation; implementation not started.
mc-mappings `data_version` 2 provides every symbol the catalog names.

## Specification

The API catalog lives in [spec/](spec/), in reading order:

1. [overview.md](spec/overview.md) — what mc-bridge is, coverage matrix, conventions, out of scope

`core/` — rules shared by every operation:

2. [core/types.md](spec/core/types.md) — owned types shared by several domains, snapshot consistency
3. [core/errors.md](spec/core/errors.md) — `BridgeError` and the out-of-world rule
4. [core/threading.md](spec/core/threading.md) — game thread, tick/frame hooks and their subscriptions

`api/` — operations, one file per domain:

5. [api/session.md](spec/api/session.md) — connect, disconnect, environment, account, game thread
6. [api/player.md](spec/api/player.md) — local player reads and writes
7. [api/world.md](spec/api/world.md) — world, entities, crosshair target, blocks, raycast, tab list
8. [api/inventory.md](spec/api/inventory.md) — inventory, open menu, slot clicks, screen kind, drop
9. [api/actions.md](spec/api/actions.md) — combat, item use and block breaking
10. [api/input.md](spec/api/input.md) — logical and physical key state
11. [api/client-state.md](spec/api/client-state.md) — camera, effects, attributes, cooldowns, tick scale, options, chat
12. [api/packets.md](spec/api/packets.md) — the three packet hooks and held packets

`integration/` — how mc-bridge uses its dependencies:

13. [integration/contracts.md](spec/integration/contracts.md) — jvm-link and mc-mappings contracts, bootstrap, external members
14. [integration/symbols.md](spec/integration/symbols.md) — appendix of every mc-mappings symbol the catalog uses
