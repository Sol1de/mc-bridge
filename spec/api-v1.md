# mc-bridge — Minecraft API v1 catalog

Status: **frozen for v1** (ticket B1). Changes after this point are additive and go through a new
catalog revision; an operation is never repurposed.

mc-bridge is a Minecraft façade for native code running inside the game's JVM (Windows x64,
Java 21). It depends only on [jvm-link](https://github.com/Sol1de/jvm-link) (JVM access) and
[mc-mappings](https://github.com/Sol1de/mc-mappings) (runtime names). Its public API exposes
owned Rust values only: no `JNIEnv`, JNI handle, `jobject`, class loader or obfuscated name, and
no network packets (out of v1).

Coverage: every operation below is required on every configuration of the matrix — Minecraft 1.21
and 1.21.11 × standard launcher Vanilla/Fabric/Forge × Lunar, Lunar + Fabric — with no partial
support.

## Owned types and units

| Type | Content | Units and ranges |
| --- | --- | --- |
| `Environment` | `minecraft_version: String`, `launcher: Launcher` (`Standard`, `Lunar`), `loader: Loader` (`None`, `Fabric`, `Forge`) | as selected in mc-mappings |
| `UserInfo` | `name: String`, `uuid: Uuid` (128-bit, owned) | — |
| `Vec3` | `x`, `y`, `z: f64` | blocks (positions), blocks per tick (velocity) |
| `Rotation` | `yaw`, `pitch: f32` | degrees; `pitch` in [-90, 90], `yaw` unbounded |
| `PlayerSnapshot` | `entity_id: i32`, `uuid`, `position: Vec3`, `rotation`, `velocity: Vec3`, `health`, `max_health: f32`, `on_ground: bool` | health in half-hearts as the game stores it |
| `EntitySnapshot` | `entity_id: i32`, `uuid`, `type_id: String` (translation key, e.g. `entity.minecraft.zombie`), `position`, `rotation`, `alive: bool`, `health: Option<f32>` (living entities only) | — |
| `WorldInfo` | `dimension: String` (resource location, e.g. `minecraft:overworld`) | — |
| `ItemStackInfo` | `item_id: String` (translation key, e.g. `item.minecraft.stone`), `count: i32` | count ≥ 1 |
| `InventorySnapshot` | `selected_hotbar_slot: u8`, `slots: Vec<Option<ItemStackInfo>>` | slot 0..=8 hotbar, size = container size |
| `ClickAction` | `Pickup`, `QuickMove`, `Swap`, `Throw` | — |
| `GameTick` | `index: u64` | counted from the subscription, one per client tick (20 per second at full speed) |

Snapshots are consistent for one call (read on the game thread in one operation) and are never
updated afterwards. `entity_id` identifies an entity only within one world generation (see B8).

## Errors and the out-of-world rule

Every operation returns `Result<_, BridgeError>`. `BridgeError` variants: `Disconnected` (session
closed or never opened), `UnsupportedEnvironment` (configuration not in mc-mappings, carries the
detected environment), `Mapping` (mc-mappings `ResolveError`, carries the semantic id),
`Java` (Java exception, converted to class name and message), `InvalidArgument` (with the
argument name), `GameThreadUnavailable` (the game no longer processes operations), `Cancelled`.

Operations that need a world or a local player return `Ok(None)` when there is none (title
screen, loading, disconnected from a server): `Result<Option<T>>`. They never return default
values instead.

## Thread and ticks

- Every read or write of game state runs on the game (render) thread. Operations called from
  another thread are submitted to it and wait for the result; called from the game thread they run
  inline (no deadlock, re-entrant).
- Submission uses `BlockableEventLoop.execute(Runnable)` on the client; `isSameThread()` decides
  inline execution. The native `Runnable` mechanism, bounds, cancellation and shutdown are B5/B6.
- Tick subscriptions observe the end of `Minecraft.tick()` (client tick). The hooking mechanism is
  chosen and proven by B5; the observed method is fixed here.

## Operations

| Operation | Contract | Out of world | Testable criteria | mc-mappings symbols |
| --- | --- | --- | --- | --- |
| `Bridge::connect() -> Session` | Borrows the JVM through jvm-link, detects the environment, selects the mc-mappings profile, checks the profile's `minecraft.client` class loads | n/a (works on the title screen) | succeeds on every matrix configuration; unknown or ambiguous environment → `UnsupportedEnvironment`; partial failure leaves nothing open (B2) | `minecraft.client` |
| `Session::environment()` | Detected environment, no game call | n/a | equals the launcher/loader/version actually running | — |
| `Session::user() -> UserInfo` | Signed-in account of the client | n/a | name and UUID match the launcher account | `minecraft.client.get_instance`, `minecraft.client.get_user`, `minecraft.user`, `minecraft.user.get_name`, `minecraft.user.get_profile_id` |
| `Session::in_world() -> bool` | True when both a client level and a local player exist | `false` | flips on join/leave | `minecraft.client.level`, `minecraft.client.player`, `minecraft.client_level`, `minecraft.local_player` |
| `Session::run_on_game_thread(f)` | Runs `f` on the game thread and returns its result | n/a | `f` observes `is_same_thread`; inline when already on it | `minecraft.blockable_event_loop`, `minecraft.blockable_event_loop.execute`, `minecraft.blockable_event_loop.is_same_thread` |
| `Session::subscribe_ticks(cb) -> Subscription` | `cb(GameTick)` once per client tick until unsubscribed; no callback after unsubscribe returns (B6) | ticks continue on the title screen | 20 ± 1 ticks per second at full speed; none after unsubscribe | `minecraft.client.tick` |
| `Session::player() -> Option<PlayerSnapshot>` | Local player state | `None` | values match the debug screen (F3) | `minecraft.entity`, `minecraft.living_entity`, `minecraft.player`, `minecraft.entity.get_id`, `minecraft.entity.get_uuid`, `minecraft.entity.get_x`, `minecraft.entity.get_y`, `minecraft.entity.get_z`, `minecraft.entity.get_y_rot`, `minecraft.entity.get_x_rot`, `minecraft.entity.get_delta_movement`, `minecraft.vec3`, `minecraft.vec3.x`, `minecraft.vec3.y`, `minecraft.vec3.z`, `minecraft.living_entity.get_health`, `minecraft.living_entity.get_max_health`, `minecraft.entity.on_ground` |
| `Session::set_rotation(Rotation) -> Option<()>` | Sets the local player's yaw and pitch | `None` | non-finite or out-of-range pitch → `InvalidArgument`; camera turns accordingly | `minecraft.entity.set_y_rot`, `minecraft.entity.set_x_rot` |
| `Session::set_velocity(Vec3) -> Option<()>` | Sets the local player's motion for the next tick | `None` | non-finite component → `InvalidArgument`; movement observed next tick | `minecraft.entity.set_delta_movement` (`double, double, double`) |
| `Session::world() -> Option<WorldInfo>` | Current client level | `None` | dimension changes with portals | `minecraft.level`, `minecraft.level.dimension`, `minecraft.resource_key`, `minecraft.resource_key.location`, `minecraft.resource_location` (rendered with `Object.toString`) |
| `Session::entities() -> Option<Vec<EntitySnapshot>>` | Entities the client currently renders, local player included | `None` | count matches F3 `E:` value; removed entities absent | `minecraft.client_level.entities_for_rendering`, `minecraft.entity.get_type`, `minecraft.entity_type`, `minecraft.entity_type.get_description_id`, `minecraft.entity.is_alive`, `minecraft.entity.is_removed`, plus the entity readers of `player()` |
| `Session::inventory() -> Option<InventorySnapshot>` | Local player inventory | `None` | slots match the inventory screen | `minecraft.player.get_inventory`, `minecraft.inventory`, `minecraft.inventory.get_container_size`, `minecraft.inventory.get_item`, `minecraft.inventory.selected`, `minecraft.item_stack`, `minecraft.item_stack.is_empty`, `minecraft.item_stack.get_count`, `minecraft.item_stack.get_item`, `minecraft.item`, `minecraft.item.get_description_id` |
| `Session::select_hotbar_slot(u8) -> Option<()>` | Selects a hotbar slot the way scrolling does (the game sends the change to the server on its next tick) | `None` | slot > 8 → `InvalidArgument`; server-side held item follows | `minecraft.inventory.selected` |
| `Session::click_slot(slot: i32, button: u8, ClickAction) -> Option<()>` | Clicks a slot of the open container through the game's own handler, which keeps client and server in sync | `None` | invalid slot or button → `InvalidArgument`; result visible in the container | `minecraft.client.game_mode`, `minecraft.multi_player_game_mode`, `minecraft.multi_player_game_mode.handle_inventory_mouse_click`, `minecraft.player.container_menu`, `minecraft.abstract_container_menu`, `minecraft.abstract_container_menu.container_id`, `minecraft.click_type`, `minecraft.click_type.pickup`, `minecraft.click_type.quick_move`, `minecraft.click_type.swap`, `minecraft.click_type.throw` |

## Symbols required from mc-mappings

The union of the last column is the mc-mappings v1 catalog (`mc_mappings/spec/catalog.md`,
`data_version` 1): 19 classes, 13 fields, 36 methods. mc-bridge copies no runtime name: it
resolves these identifiers through mc-mappings at connection time and caches the JNI ids per
session and class loader (B4). JDK members (`Object.toString`, `Iterable.iterator`,
`Iterator.hasNext/next`, `UUID.getMost/LeastSignificantBits`) are called by their standard names.

## Integration contracts

- **jvm-link**: `Connection::connect_existing` borrows the JVM; every game access happens inside
  `Connection::with_env` scopes; local references never escape a scope; global references and
  cached ids belong to the session and are released before `Connection::close`. No parallel
  attach mechanism and no JVM destruction (B2, B10).
- **mc-mappings**: `Dataset::minecraft_v1()` then `Dataset::select(version, launcher, loader)` with
  the detected environment; `Selection::class/field/method` give runtime names, descriptors and
  the static flag used for `GetStaticMethodID` versus `GetMethodID`. A `ResolveError` aborts the
  operation (no fallback); an unsupported environment fails `connect`.
- **Bootstrap without circular resolution (B3)**: the environment is detected before any mapping is
  used — loader from its own classes (`net/fabricmc/loader/api/FabricLoader`,
  `net/minecraftforge/fml/loading/FMLLoader`), launcher from Lunar's classes
  (`com/moonsworth/lunar/...`), version from the game's `version.json` resource — then the
  profile's `minecraft.client` class must load from the game class loader, which proves the
  selection.
