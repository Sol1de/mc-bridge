# Integration contracts

## jvm-link

`Connection::connect_existing` borrows the JVM; every game access happens inside
`Connection::with_env` scopes; local references never escape a scope; global references and
cached ids belong to the session and are released before `Connection::close`. No parallel attach
mechanism and no JVM destruction.

## mc-mappings

`Dataset::minecraft_v1()` then `Dataset::select(version, launcher, loader)` with the detected
environment; `Selection::class/field/method` give runtime names, descriptors and the static flag.
A `ResolveError` aborts the operation (no fallback); an unsupported environment fails `connect`.
mc-bridge copies no runtime name. For the packet symbols that exist in one Minecraft version only
([version-dependent symbols](symbols.md#version-dependent-symbols)), mc-bridge resolves the set of
the detected version; `ResolveError::AbsentInVersion` on the other set is expected, not an error.

## Bootstrap without circular resolution

The environment is detected before any mapping is used — loader from its own classes
(`net/fabricmc/loader/api/FabricLoader`, `net/minecraftforge/fml/loading/FMLLoader`), launcher
from Lunar's classes (`com/moonsworth/lunar/...`), version from the game's `version.json`
resource — then the profile's `minecraft.client` class must load from the game class loader,
which proves the selection.

## External members

JDK, LWJGL, authlib and fastutil members are called by their own names and are not mc-mappings
symbols: `Object.toString`, `Iterable.iterator`, `Iterator.hasNext/next`, `List.size/get`,
`Map.values`, `UUID` bits, `Enum.name`, `Number.intValue/doubleValue`, `Double.valueOf`, fastutil
`Object2IntMap.Entry` `getKey/getIntValue`, GLFW `glfwGetKey/glfwGetMouseButton`. `GameProfile`
differs by authlib version: `getName()`/`getId()` in 1.21 (authlib 6.0.54), record accessors
`name()`/`id()` in 1.21.11 (authlib 7.0.61).
