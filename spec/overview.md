# mc-bridge — Minecraft API catalog

Status: **revised before implementation**. Changes after freeze are additive and go through a new
catalog revision; an operation is never repurposed.

mc-bridge is a Minecraft façade for native code running inside the game's JVM (Windows x64,
Java 21). It depends only on [jvm-link](https://github.com/Sol1de/jvm-link) (JVM access) and
[mc-mappings](https://github.com/Sol1de/mc-mappings) (runtime names). Its public API exposes
owned Rust values only: no `JNIEnv`, JNI handle, `jobject`, class loader or obfuscated name.

It is a façade, not a client: it offers reads, vanilla actions and three packet hooks. It contains
no module (no feature logic, no automation, no rendering). Actions are performed through the
game's own client code paths; the server stays authoritative and may reject or correct their
effect (attack reach, `on_ground`, velocity, position). mc-bridge never touches sockets or packet
bytes: a vanilla action that makes the game send a packet is an ordinary operation.

## Coverage

Every operation of this catalog is required on every configuration of the matrix — Minecraft 1.21
and 1.21.11 × standard launcher Vanilla/Fabric/Forge × Lunar, Lunar + Fabric — with no partial
support.

## Conventions

**Symbols** entries name mc-mappings identifiers without the `minecraft.` prefix; the full list,
with Mojang names, is in the [symbol appendix](integration/symbols.md).

## Out of scope

OpenGL hooks, chams, X-ray renderer, animations, client GUI, friends list; any module or feature
logic; sockets, raw packets and packet bytes.
