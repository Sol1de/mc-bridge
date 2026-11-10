# mc-bridge — Minecraft API v1 catalog

Status: **revised before implementation** (ticket B1). Changes after freeze are additive and go
through a new catalog revision; an operation is never repurposed.

mc-bridge is a Minecraft façade for native code running inside the game's JVM (Windows x64,
Java 21). It depends only on [jvm-link](https://github.com/Sol1de/jvm-link) (JVM access) and
[mc-mappings](https://github.com/Sol1de/mc-mappings) (runtime names). Its public API exposes
owned Rust values only: no `JNIEnv`, JNI handle, `jobject`, class loader or obfuscated name.

It is a façade, not a client: it offers reads, vanilla actions and three packet hooks. It contains
no module (no feature logic, no automation, no rendering). Actions are performed through the
game's own client code paths; the server stays authoritative and may reject or correct their
effect (attack reach, `on_ground`, velocity, position). mc-bridge never touches sockets or packet
bytes: a vanilla action that makes the game send a packet is an ordinary operation.

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
| `Aabb` | `min`, `max: Vec3` | blocks, world coordinates |
| `BlockPos` | `x`, `y`, `z: i32` | block coordinates |
| `Direction` | `Down`, `Up`, `North`, `South`, `West`, `East` | the game's 3D data values 0..=5 |
| `Hand` | `Main`, `Off` | — |
| `GameMode` | `Survival`, `Creative`, `Adventure`, `Spectator` | — |
| `EquipmentInfo` | `main_hand`, `off_hand: Option<ItemStackInfo>`, `armor: [Option<ItemStackInfo>; 4]` | armor feet → legs → chest → head |
| `PlayerSnapshot` | `entity: EntitySnapshot`, `max_health: f32`, `fall_distance: f64`, `food: i32`, `saturation: f32`, `using_item: bool`, `use_ticks_remaining: i32`, `game_mode: GameMode`, `flying`, `may_fly`, `instabuild: bool`, `attack_strength: f32` | food 0..=20; `attack_strength` in [0, 1] (partial tick 0) |
| `EntitySnapshot` | `entity_id: i32`, `uuid`, `type_id: String` (translation key, e.g. `entity.minecraft.zombie`), `position`, `previous_position: Vec3`, `velocity: Vec3`, `rotation`, `aabb: Aabb`, `eye_height: f32`, `tick_count: i32`, `alive`, `on_ground: bool`, `sneaking`, `sprinting`, `swimming`, `gliding`, `invisible`, `glowing: bool`, `custom_name: Option<String>`, `profile_name: Option<String>` (players only), `health: Option<f32>`, `absorption: Option<f32>`, `hurt_time: Option<i32>`, `equipment: Option<EquipmentInfo>` (living entities only), `vehicle_id: Option<i32>` | `previous_position` is the position at the start of the current tick; `hurt_time` counts down from 10 |
| `WorldInfo` | `dimension: String` (resource location, e.g. `minecraft:overworld`) | — |
| `ItemStackInfo` | `item_id: String` (translation key, e.g. `item.minecraft.stone`), `count: i32`, `damage`, `max_damage: i32`, `enchantments: Vec<(String, i32)>` (id such as `minecraft:sharpness`, level) | count ≥ 1; `max_damage` 0 for items without durability |
| `InventorySnapshot` | `selected_hotbar_slot: u8`, `main: [Option<ItemStackInfo>; 36]`, `armor: [Option<ItemStackInfo>; 4]`, `offhand: Option<ItemStackInfo>` | `main` 0..=8 is the hotbar; armor feet → head; independent of any open screen |
| `MenuSnapshot` | `container_id: i32`, `slots: Vec<Option<ItemStackInfo>>`, `carried: Option<ItemStackInfo>` | indices are the open menu's (`player.containerMenu`, the player inventory menu when no container is open); they are **not** `inventory()` indices |
| `ClickAction` | `Pickup`, `QuickMove`, `Swap`, `Throw` | — |
| `Hit` | `Miss`, `Block { pos: BlockPos, face: Direction, location: Vec3, inside: bool }`, `Entity { entity_id: i32, location: Vec3 }` | — |
| `BlockSample` | `block_id: String` (translation key, e.g. `block.minecraft.stone`), `air`, `replaceable`, `collision_empty`, `fluid: bool` | `fluid` true when the block holds a non-empty fluid |
| `Camera` | `position: Vec3`, `rotation: Rotation`, `fov_degrees: f32`, `third_person: bool` | values of the last rendered frame |
| `Frame` | `partial_tick: f32` | in [0, 1) |
| `GameTick` | `index: u64` | counted from the subscription |
| `Key` | `Attack`, `Use`, `Forward`, `Back`, `Left`, `Right`, `Jump`, `Sneak`, `Sprint`, `Drop`, `SwapOffhand`, `Hotbar(u8)` | `Hotbar` 0..=8; each maps to the player's key binding |
| `ScreenKind` | `None`, `Chat`, `Inventory` (survival or creative inventory), `Container` (any other container screen), `Other` | — |
| `Effect` | `id: String` (e.g. `minecraft:speed`), `amplifier: i32`, `duration_ticks: i32` | amplifier 0 = level I; duration −1 = infinite |
| `Attribute` | `id: String` (e.g. `minecraft:movement_speed`), `base`, `value: f64` | `value` includes modifiers |
| `TabEntry` | `name: String`, `uuid`, `latency_ms: i32`, `game_mode: Option<GameMode>`, `display_name: Option<String>` | — |
| `GameOptions` | `sensitivity: f64`, `fov_degrees: i32`, `gamma: f64` | sensitivity 0..=1, FOV 30..=110, gamma as set |
| `PacketKind` | `EntityMotion`, `EntityPosition`, `MovePlayer` | — |
| `PacketEvent` | `EntityMotion { entity_id, velocity: Vec3 }` (velocity mutable), `EntityPosition { entity_id, position: Vec3, rotation: Rotation, on_ground: bool, teleport: bool }` (read-only), `MovePlayer { position: Option<Vec3>, rotation: Option<Rotation>, on_ground: bool }` (mutable; `None` when the packet does not carry it) | — |
| `PacketDecision` | `Pass`, `Drop`, `Hold` | — |

Snapshots are consistent for one call (read on the game thread in one operation) and are never
updated afterwards. `entity_id` identifies an entity only within one world generation (see B8).

## Errors and the out-of-world rule

Every operation returns `Result<_, BridgeError>`. `BridgeError` variants: `Disconnected` (session
closed or never opened), `UnsupportedEnvironment` (configuration not in mc-mappings, carries the
detected environment), `Mapping` (mc-mappings `ResolveError`, carries the semantic id),
`Java` (Java exception, converted to class name and message), `InvalidArgument` (with the
argument name and reason), `GameThreadUnavailable` (the game no longer processes operations),
`Cancelled`.

Operations that need a world or a local player return `Ok(None)` when there is none (title
screen, loading, disconnected from a server): `Result<Option<T>>`. They never return default
values instead. Arguments are validated before any game call: non-finite numbers, out-of-range
slots, hotbar indices, radii, scales or text are `InvalidArgument`.

## Thread, ticks and frames

- Every read or write of game state runs on the game (render) thread. Operations called from
  another thread are submitted to it and wait for the result; called from the game thread they run
  inline (no deadlock, re-entrant).
- Submission uses `BlockableEventLoop.execute(Runnable)` on the client; `isSameThread()` decides
  inline execution. The native `Runnable` mechanism, bounds, cancellation and shutdown are B5/B6.
- The hooks below are fixed here; the hooking mechanism is chosen and proven by B5.

| Hook | Point | Order |
| --- | --- | --- |
| `subscribe_pre_tick(cb)` | head of `LocalPlayer.sendPosition()` | after the player's movement of the tick, **before** the movement packet; not while riding (the game sends vehicle packets instead) |
| `subscribe_ticks(cb)` | end of `Minecraft.tick()` | **after** the movement packet; also on the title screen |
| `subscribe_frames(cb)` | return of `GameRenderer.pick(float)` | once per rendered frame, after the crosshair target is updated; `Frame.partial_tick` is `pick`'s argument |

Callbacks run on the game thread; no callback runs after its unsubscribe returns (B6).

## Operations

Symbol columns name mc-mappings identifiers without the `minecraft.` prefix; the full list, with
Mojang names, is in the [appendix](#appendix-mc-mappings-symbols).

### Session

| Operation | Contract | Out of world | Testable criteria | Symbols |
| --- | --- | --- | --- | --- |
| `Bridge::connect() -> Session` | Borrows the JVM through jvm-link, detects the environment, selects the mc-mappings profile, checks the profile's `minecraft.client` class loads | n/a | succeeds on every matrix configuration; unknown or ambiguous environment → `UnsupportedEnvironment`; partial failure leaves nothing open (B2) | `client` |
| `Session::disconnect()` | Releases held packets, ends every subscription, frees session references, then closes the jvm-link connection; later calls return `Disconnected` | n/a | idempotent; the game keeps running | — |
| `Session::environment()` | Detected environment, no game call | n/a | equals the running launcher/loader/version | — |
| `Session::user() -> UserInfo` | Signed-in account | n/a | name and UUID match the launcher account | `client.get_instance`, `client.get_user`, `user.*` |
| `Session::in_world() -> bool` | A client level and a local player exist | `false` | flips on join/leave | `client.level`, `client.player` |
| `Session::run_on_game_thread(f)` | Runs `f` on the game thread and returns its result | n/a | inline when already on it | `blockable_event_loop.*` |
| `Session::subscribe_pre_tick(cb)` / `subscribe_ticks(cb)` / `subscribe_frames(cb)` | See [hooks](#thread-ticks-and-frames); return a `Subscription` | ticks continue on the title screen; pre-tick and frames need a player / a world | pre-tick runs before the movement packet, tick after; 20 ± 1 ticks per second at scale 1 | `local_player.send_position`, `client.tick`, `client.game_renderer`, `game_renderer.pick` |

### Local player

| Operation | Contract | Out of world | Testable criteria | Symbols |
| --- | --- | --- | --- | --- |
| `player() -> Option<PlayerSnapshot>` | Local player state | `None` | values match the debug screen (F3); `attack_strength` in [0, 1] | entity readers (below), `player.get_food_data`, `food_data.*`, `living_entity.is_using_item`, `living_entity.get_use_item_remaining_ticks`, `multi_player_game_mode.get_player_mode`, `game_type.*`, `player.get_abilities`, `abilities.*`, `player.get_attack_strength_scale`, `entity.fall_distance` (versioned type) |
| `set_rotation(Rotation) -> Option<()>` | Writes `yRot`, `xRot`, `yRotO` and `xRotO` (no interpolation from the previous rotation) | `None` | non-finite or pitch outside [-90, 90] → `InvalidArgument`; camera turns at once | `entity.y_rot`, `entity.x_rot`, `entity.y_rot_o`, `entity.x_rot_o` |
| `turn(dyaw, dpitch) -> Option<()>` | Feeds the delta to `MouseHandler.turnPlayer` as mouse movement (current sensitivity and mouse inversion apply, as for a real mouse); pitch clamps at ±90 | `None` | non-finite → `InvalidArgument`; resulting rotation differs from the request only by the game's mouse quantisation | `client.mouse_handler`, `mouse_handler.accumulated_dx`, `mouse_handler.accumulated_dy`, `mouse_handler.turn_player` |
| `set_velocity(Vec3) -> Option<()>` | Sets the local player's motion for its next movement | `None` | non-finite → `InvalidArgument`; movement observed next tick. Called from the tick hook it does **not** undo a knockback already integrated in this tick's movement (use the `EntityMotion` packet hook for that) | `entity.set_delta_movement` |
| `set_position(Vec3) -> Option<()>` | Moves the local player (client side) | `None` | non-finite → `InvalidArgument`; the server may correct it | `entity.set_pos` |
| `set_on_ground(bool) -> Option<()>` | Sets the client on-ground flag | `None` | sent with the next movement packet; the server may reject it | `entity.set_on_ground` |
| `set_sprinting(bool) -> Option<()>` | Starts or stops sprinting through the player's own state change (the game sends the sprint command) | `None` | sprint state and server view follow | `local_player.set_sprinting` |
| `set_sneaking(bool) -> Option<()>` | Holds or releases the logical `Sneak` key, so the game's input produces sneaking and its packets | `None` | sneaking follows within one tick | key symbols (below) |
| `respawn() -> Option<()>` | Requests a respawn | `None` | player not dead → `InvalidArgument` | `local_player.respawn` |

### World and entities

| Operation | Contract | Out of world | Testable criteria | Symbols |
| --- | --- | --- | --- | --- |
| `world() -> Option<WorldInfo>` | Current client level | `None` | dimension changes with portals | `level.dimension`, `resource_key.location`, `resource_location` (`Object.toString`) |
| `entities() -> Option<Vec<EntitySnapshot>>` | **Every entity the client tracks**, in or out of the view frustum, local player included. `ClientLevel.entitiesForRendering()` returns `getEntities().getAll()` (checked by disassembly on 1.21.11): it is not the rendered count of F3's `E:` | `None` | an entity behind the camera is present; removed entities absent | `client_level.entities_for_rendering`, entity readers: `entity.*`, `living_entity.*`, `aabb.*`, `entity_type.get_description_id`, `component.get_string`, `player.get_game_profile`, `equipment_slot.*`, `living_entity.get_item_by_slot` |
| `hit() -> Option<Hit>` | Current crosshair target (`Minecraft.hitResult`) | `None` | follows the crosshair | `client.hit_result`, `hit_result.*`, `hit_result_type.*`, `block_hit_result.*`, `entity_hit_result.get_entity`, `block_pos.*`, `direction.*` |
| `block(BlockPos) -> Option<BlockSample>` | One block state | `None` | air and solid blocks are distinct; unloaded positions read as the game reports them | `level.get_block_state`, `block_state.*`, `block.get_description_id`, `fluid_state.is_empty`, `voxel_shape.is_empty`, `block_pos.new` |
| `blocks(center, radius: u8) -> Option<Vec<(BlockPos, BlockSample)>>` | Cube of side `2 × radius + 1`, x then z then y order | `None` | `radius > 32` → `InvalidArgument` | as `block` |
| `raycast(from, to: Vec3, blocks: bool, fluids: bool) -> Option<Hit>` | Block ray from the game's clip (`blocks`: collision shapes when false, outline shapes when true; `fluids`: stop on any fluid) | `None` | non-finite → `InvalidArgument`; matches `hit()` when cast along the view | `level.clip`, `clip_context.new`, `clip_context_block.*`, `clip_context_fluid.*`, `vec3.new` |
| `tab() -> Option<Vec<TabEntry>>` | Tab list entries | `None` | matches the tab list | `client.get_connection`, `client_packet_listener.get_online_players`, `player_info.*` |

### Inventory and menus

| Operation | Contract | Out of world | Testable criteria | Symbols |
| --- | --- | --- | --- | --- |
| `inventory() -> Option<InventorySnapshot>` | `main[36]` (`Inventory.getItem(0..36)`), armor and offhand from the player's equipment slots; independent of the open screen | `None` | armor and offhand read with a chest closed and open alike | `player.get_inventory`, `inventory.get_item`, `inventory.selected`, `living_entity.get_item_by_slot`, `equipment_slot.*`, item readers: `item_stack.*`, `item.get_description_id`, `item_enchantments.entry_set`, `holder.get_registered_name` |
| `select_hotbar_slot(u8) -> Option<()>` | Selects a hotbar slot like scrolling (the game sends the change on its next tick) | `None` | slot > 8 → `InvalidArgument` | `inventory.selected` |
| `menu() -> Option<MenuSnapshot>` | `player.containerMenu`: its slots in menu order and the carried stack | `None` | indices match `click_slot` | `player.container_menu`, `abstract_container_menu.*`, `slot.get_item`, item readers |
| `click_slot(slot: i32, button: u8, ClickAction) -> Option<()>` | Clicks slot `slot` **of `player.containerMenu`** through the game's handler (client and server stay in sync); menu indices, not `inventory()` ones | `None` | slot outside the menu or invalid button → `InvalidArgument`; result visible in `menu()` | `client.game_mode`, `multi_player_game_mode.handle_inventory_mouse_click`, `abstract_container_menu.container_id`, `click_type.*` |
| `screen() -> ScreenKind` | Kind of the open screen | works on any screen: the title screen is `Other`, `None` only when no screen is open | chat, inventory, chest distinguished | `client.screen`, `chat_screen`, `inventory_screen`, `creative_mode_inventory_screen`, `abstract_container_screen` |
| `drop(entire_stack: bool) -> Option<()>` | Drops the selected hotbar stack (or one item) like the drop key | `None` | count decreases accordingly | `local_player.drop` |

### Combat, use and blocks

All go through `MultiPlayerGameMode`, as the game does for mouse input.

| Operation | Contract | Out of world | Testable criteria | Symbols |
| --- | --- | --- | --- | --- |
| `attack(entity_id) -> Option<()>` | Attacks the entity, then swings the main hand (vanilla order) | `None` | unknown id → `InvalidArgument`; the swing is visible | `client_level.get_entity`, `multi_player_game_mode.attack`, `living_entity.swing`, `interaction_hand.*` |
| `use_item(Hand) -> Option<()>` | Uses the item in hand | `None` | e.g. eating starts | `multi_player_game_mode.use_item`, `interaction_result` |
| `use_on(Hand, Hit) -> Option<()>` | Uses the item on a block hit | `None` | non-`Block` hit → `InvalidArgument` | `multi_player_game_mode.use_item_on`, `block_hit_result.new` |
| `release_use() -> Option<()>` | Releases the item being used | `None` | bow fires | `multi_player_game_mode.release_using_item` |
| `interact(entity_id, Hand) -> Option<()>` | Interacts with an entity | `None` | unknown id → `InvalidArgument` | `multi_player_game_mode.interact` |
| `start_destroy(BlockPos, Direction)` / `continue_destroy(BlockPos, Direction)` / `stop_destroy()` `-> Option<()>` | Block breaking as with a held attack key | `None` | progress reported by `destroy_progress` | `multi_player_game_mode.start_destroy_block`, `continue_destroy_block`, `stop_destroy_block` |
| `swing(Hand) -> Option<()>` | Arm swing animation and packet | `None` | swing visible | `living_entity.swing` |

### Input

| Operation | Contract | Out of world | Testable criteria | Symbols |
| --- | --- | --- | --- | --- |
| `input() -> Vec<(Key, bool)>` | Logical state of every `Key` binding (`KeyMapping.isDown`) | works on any screen | reflects `set_key` | `client.options`, `options.key_*`, `key_mapping.is_down` |
| `physical_key(Key) -> bool` | Physical state of the bound key or mouse button, read from GLFW with the game window (`glfwGetKey` / `glfwGetMouseButton`, external) | works on any screen | stays true while held even after `set_key(Attack, false)` | `key_mapping.key`, `input_constants_key.*`, `input_constants_type.mouse`, `client.get_window`, `window.get_window` |
| `set_key(Key, bool)` | Sets the logical state until changed or until the next physical event on that binding | — | the game acts on it next tick | `key_mapping.set_down` |
| `click_key(Key)` | Counts one press, consumed by the game like a real press | — | one action per click | `key_mapping.click_count` |

### Camera, timing and options

| Operation | Contract | Out of world | Testable criteria | Symbols |
| --- | --- | --- | --- | --- |
| `camera() -> Option<Camera>` | Camera of the last rendered frame: position, rotation, effective FOV (`GameRenderer.getFov`), third person (`Camera.isDetached`) | `None` | follows F5 and FOV effects | `game_renderer.get_main_camera`, `camera.*`, `game_renderer.get_fov` (versioned type) |
| `effects() -> Option<Vec<Effect>>` | Active effects of the local player | `None` | match the inventory screen | `living_entity.get_active_effects`, `mob_effect_instance.*`, `holder.get_registered_name` |
| `attributes() -> Option<Vec<Attribute>>` / `set_attribute(id, base: f64) -> Option<()>` | Attribute instances of the local player; the setter writes the base value (client side) | `None` | unknown id or non-finite → `InvalidArgument`; the server may recompute | `living_entity.get_attributes`, `attribute_map.attributes`, `attribute_instance.*` |
| `item_cooldown() -> Option<f32>` | Cooldown fraction of the main-hand item | `None` | in [0, 1]; 0 without cooldown | `player.get_cooldowns`, `item_cooldowns.get_cooldown_percent` (versioned parameter), `living_entity.get_main_hand_item` |
| `destroy_progress() -> Option<f32>` | Current block-breaking progress | `None` | 0 when not breaking | `multi_player_game_mode.destroy_progress`, `multi_player_game_mode.is_destroying` |
| `right_click_delay()` / `set_right_click_delay(i32)` | Game's use cooldown in ticks | works on the title screen | negative → `InvalidArgument` | `client.right_click_delay` |
| `miss_time()` / `set_miss_time(i32)` | Game's attack miss cooldown in ticks | as above | negative → `InvalidArgument` | `client.miss_time` |
| `tick_scale() -> f32` / `set_tick_scale(f32)` | Client tick speed relative to vanilla (`1.0` = vanilla). The setter writes the client timer's `msPerTick` **and** the client level's tick rate, since the timer uses the larger of the two (`Math.max`, checked by disassembly); re-applied when the level or the server's ticking state changes | scale kept on the title screen | `<= 0` or non-finite → `InvalidArgument`; `set_tick_scale(2.0)` doubles ticks per second | `client.timer`, `delta_tracker_timer.ms_per_tick`, `client_level.tick_rate_manager`, `tick_rate_manager.*` |
| `options() -> GameOptions` / `set_gamma(f64)` | Sensitivity, FOV and gamma; `set_gamma` writes the option's value without range check or saving | — | non-finite or negative → `InvalidArgument` | `options.sensitivity`, `options.fov`, `options.gamma`, `option_instance.get`, `option_instance.value` |
| `send_chat(String) -> Option<()>` | Sends a chat message, or a command when it starts with `/` | `None` | empty, over 256 characters or containing control characters → `InvalidArgument` | `client_packet_listener.send_chat`, `client_packet_listener.send_command` |

## Packets

`subscribe_packets(kinds: &[PacketKind], cb) -> Subscription`; `cb(&mut PacketEvent) ->
PacketDecision`. Only the three kinds below exist; there is no generic packet access and no bytes.

| Kind | Direction and point | Decision |
| --- | --- | --- |
| `EntityMotion` | incoming `ClientboundSetEntityMotionPacket`, at `ClientPacketListener.handleSetEntityMotion` on the game thread | `velocity` mutable: a changed velocity replaces the packet by `new(entity_id, velocity)` before the game handles it |
| `EntityPosition` | incoming teleport and relative-move packets (`handleTeleportEntity`, `handleMoveEntity`; 1.21.11 also `handleEntityPositionSync`); relative moves are decoded with the entity's `VecDeltaCodec`; `teleport` is true for teleports | read-only |
| `MovePlayer` | outgoing `ServerboundMovePlayerPacket`, at `ClientPacketListener.send(Packet)` | `position`, `rotation`, `on_ground` mutable (written to the packet's fields; absent parts stay absent) |

- `Pass` lets the packet through, `Drop` discards it, `Hold` keeps it.
- `release_held(kind)` delivers the held packets of that kind in arrival order (incoming: handled
  on the game thread; outgoing: sent without re-entering the hook). Unsubscribe and
  `Session::disconnect` release everything held. Packets held when the game's server connection
  closes are discarded (nothing to deliver to).
- Callbacks run on the game thread and must not block; mutations are validated (non-finite →
  the packet passes unchanged and the error is reported to the subscription).

## Out of scope

OpenGL hooks, chams, X-ray renderer, animations, client GUI, friends list; any module or feature
logic; sockets, raw packets and packet bytes.

## Tests

| Test | Expected |
| --- | --- |
| Entity behind the camera | present in `entities()` |
| `hit()` | follows the crosshair |
| `set_key(Attack, false)` while the attack button is held | `physical_key(Attack)` stays true |
| `block()` on air and on stone | distinct samples (`air` true / false, `collision_empty`) |
| `attack(id)` | the target is hit and the main hand swings |
| Hook order | pre-tick before the movement packet, tick after |
| `Hold` on `MovePlayer` | the server-side position freezes until `release_held(MovePlayer)` |
| `EntityMotion` velocity set to zero | the entity does not move from that packet |
| `inventory()` with a chest closed | armor and offhand read |
| `set_tick_scale(2.0)` | twice the ticks per second |
| mc-mappings | every `minecraft.*` symbol of the appendix is in the mc-mappings catalog count and resolves on every configuration (audit) |

## Integration contracts

- **jvm-link**: `Connection::connect_existing` borrows the JVM; every game access happens inside
  `Connection::with_env` scopes; local references never escape a scope; global references and
  cached ids belong to the session and are released before `Connection::close`. No parallel
  attach mechanism and no JVM destruction (B2, B10).
- **mc-mappings**: `Dataset::minecraft_v1()` then `Dataset::select(version, launcher, loader)` with
  the detected environment; `Selection::class/field/method` give runtime names, descriptors and
  the static flag. A `ResolveError` aborts the operation (no fallback); an unsupported environment
  fails `connect`. mc-bridge copies no runtime name.
- **Bootstrap without circular resolution (B3)**: the environment is detected before any mapping is
  used — loader from its own classes (`net/fabricmc/loader/api/FabricLoader`,
  `net/minecraftforge/fml/loading/FMLLoader`), launcher from Lunar's classes
  (`com/moonsworth/lunar/...`), version from the game's `version.json` resource — then the
  profile's `minecraft.client` class must load from the game class loader, which proves the
  selection.
- **External members** (JDK, LWJGL, authlib, fastutil) are called by their own names and are not
  mc-mappings symbols: `Object.toString`, `Iterable.iterator`, `Iterator.hasNext/next`,
  `List.size/get`, `Map.values`, `UUID` bits, `Enum.name`, `Number.intValue/doubleValue`,
  `Double.valueOf`, fastutil `Object2IntMap.Entry` `getKey/getIntValue`, GLFW
  `glfwGetKey/glfwGetMouseButton`. `GameProfile` differs by authlib
  version: `getName()`/`getId()` in 1.21 (authlib 6.0.54), record accessors `name()`/`id()` in
  1.21.11 (authlib 7.0.61).

## Appendix: mc-mappings symbols

Checked on 2026-11-11 against the generated mappings of all ten configurations (standard and
Lunar), with the mc-mappings export and audit. mc-mappings `data_version` 1 holds only the first
row of the count below; the common symbols, with their types, are the generator input
`mc_mappings_generator/consumers/mc_mappings/catalog-v2-common.json`.

### Symbol count

| Group | Classes | Fields | Methods |
| --- | --- | --- | --- |
| Catalog `data_version` 1 (before this revision) | 19 | 13 | 36 |
| Added, common to every configuration (verified) | 58 | 75 | 127 |
| Added, same symbol with a version-specific type | 0 | 1 | 4 |
| Added, 1.21 only | 0 | 0 | 8 |
| Added, 1.21.11 only | 2 | 0 | 10 |
| **Total per 1.21 profile** | **77** | **89** | **175** |
| **Total per 1.21.11 profile** | **79** | **89** | **177** |

### Prerequisites in mc-mappings and its generator

1. **Version-specific types** (supported by table format 1, whose types are per profile): the
   generator catalog gains `type_by_version` / `parameters_by_version` / `returns_by_version`.
   Symbols: `entity.fall_distance` (`float` 1.21, `double` 1.21.11),
   `game_renderer.get_fov` (returns `double` / `float`),
   `item_cooldowns.get_cooldown_percent` (`(Item, float)` / `(ItemStack, float)`),
   `move_entity_packet.get_y_rot` and `get_x_rot` (`getyRot()`/`getxRot()` returning `byte` in
   1.21, `getYRot()`/`getXRot()` returning `float` in 1.21.11).
2. **Version-scoped symbols** (needs mc-mappings `schema_version` 2: a catalog entry may name the
   Minecraft versions it applies to; profiles of other versions omit it and `select` reports it
   as absent for that version). They decode packet content whose structure changed:
   - 1.21 only: `set_entity_motion_packet.get_xa/get_ya/get_za` (velocity),
     `teleport_entity_packet.get_x/get_y/get_z/get_y_rot/get_x_rot`.
   - 1.21.11 only: `set_entity_motion_packet.get_movement`, `teleport_entity_packet.change`,
     `teleport_entity_packet.relatives`, `position_move_rotation` (class) `.position/.y_rot/.x_rot`,
     `entity_position_sync_packet` (class) `.id/.values/.on_ground`,
     `client_packet_listener.handle_entity_position_sync`.

   Without schema 2, `EntityMotion` reads and `EntityPosition` cannot be implemented on both
   versions; every other operation needs at most prerequisite 1.

### Common symbols (all ten configurations)

Mojang classes are relative to `net/minecraft/` (`com/mojang/` for `blaze3d`). `→` gives the
1.21.11 Mojang name when Mojang renamed the member (same semantic id). Every `data_version` 1
symbol keeps its id. Classes without members are declaring or parameter classes.

| Id | Mojang class | Members |
| --- | --- | --- |
| `minecraft.client` | `client/Minecraft` | `player`, `level`, `gameMode`, `gameRenderer`, `hitResult`, `options`, `screen`, `rightClickDelay`, `missTime`, `timer`→`deltaTracker`, `mouseHandler`, `getInstance()`, `getUser()`, `tick()`, `getConnection()`, `getWindow()` |
| `minecraft.blockable_event_loop` | `util/thread/BlockableEventLoop` | `execute()`, `isSameThread()` |
| `minecraft.user` | `client/User` | `getName()`, `getProfileId()` |
| `minecraft.entity` | `world/entity/Entity` | `xo`, `yo`, `zo`, `tickCount`, `yRot`, `xRot`, `yRotO`, `xRotO`, `getId()`, `getUUID()`, `getType()`, `getX()`, `getY()`, `getZ()`, `getYRot()`, `getXRot()`, `setYRot()`, `setXRot()`, `getDeltaMovement()`, `setDeltaMovement()`, `isAlive()`, `isRemoved()`, `onGround()`, `getBoundingBox()`, `getEyeHeight()`, `isShiftKeyDown()`, `isSprinting()`, `isSwimming()`, `isInvisible()`, `isCurrentlyGlowing()`, `getCustomName()`, `getVehicle()`, `setPos()`, `setOnGround()`, `getPositionCodec()` |
| `minecraft.entity_type` | `world/entity/EntityType` | `getDescriptionId()` |
| `minecraft.living_entity` | `world/entity/LivingEntity` | `hurtTime`, `getHealth()`, `getMaxHealth()`, `isFallFlying()`, `getItemBySlot()`, `getAbsorptionAmount()`, `isUsingItem()`, `getUseItemRemainingTicks()`, `getMainHandItem()`, `getActiveEffects()`, `getAttributes()`, `swing()` |
| `minecraft.player` | `world/entity/player/Player` | `containerMenu`, `getInventory()`, `getGameProfile()`, `getFoodData()`, `getAbilities()`, `getAttackStrengthScale()`, `getCooldowns()` |
| `minecraft.local_player` | `client/player/LocalPlayer` | `sendPosition()`, `setSprinting()`, `drop()`, `respawn()` |
| `minecraft.inventory` | `world/entity/player/Inventory` | `selected`, `getItem()`, `getContainerSize()` |
| `minecraft.item_stack` | `world/item/ItemStack` | `isEmpty()`, `getCount()`, `getItem()`, `getDamageValue()`, `getMaxDamage()`, `getEnchantments()` |
| `minecraft.item` | `world/item/Item` | `getDescriptionId()` |
| `minecraft.level` | `world/level/Level` | `dimension()`, `getBlockState()`, `clip()` |
| `minecraft.client_level` | `client/multiplayer/ClientLevel` | `entitiesForRendering()`, `tickRateManager()`, `getEntity()` |
| `minecraft.resource_key` | `resources/ResourceKey` | `location()`→`identifier()` |
| `minecraft.resource_location` | `resources/ResourceLocation` | — |
| `minecraft.vec3` | `world/phys/Vec3` | `x`, `y`, `z`, `<init>(3)` |
| `minecraft.multi_player_game_mode` | `client/multiplayer/MultiPlayerGameMode` | `destroyProgress`, `handleInventoryMouseClick()`, `getPlayerMode()`, `isDestroying()`, `attack()`, `useItem()`, `useItemOn()`, `releaseUsingItem()`, `interact()`, `startDestroyBlock()`, `continueDestroyBlock()`, `stopDestroyBlock()` |
| `minecraft.abstract_container_menu` | `world/inventory/AbstractContainerMenu` | `containerId`, `slots`, `getCarried()` |
| `minecraft.click_type` | `world/inventory/ClickType` | `PICKUP`, `QUICK_MOVE`, `SWAP`, `THROW` |
| `minecraft.aabb` | `world/phys/AABB` | `minX`, `minY`, `minZ`, `maxX`, `maxY`, `maxZ` |
| `minecraft.component` | `network/chat/Component` | `getString()` |
| `minecraft.equipment_slot` | `world/entity/EquipmentSlot` | `MAINHAND`, `OFFHAND`, `FEET`, `LEGS`, `CHEST`, `HEAD` |
| `minecraft.food_data` | `world/food/FoodData` | `getFoodLevel()`, `getSaturationLevel()` |
| `minecraft.game_type` | `world/level/GameType` | `SURVIVAL`, `CREATIVE`, `ADVENTURE`, `SPECTATOR` |
| `minecraft.abilities` | `world/entity/player/Abilities` | `flying`, `mayfly`, `instabuild` |
| `minecraft.item_enchantments` | `world/item/enchantment/ItemEnchantments` | `entrySet()` |
| `minecraft.holder` | `core/Holder` | `getRegisteredName()` |
| `minecraft.game_renderer` | `client/renderer/GameRenderer` | `pick()`, `getMainCamera()` |
| `minecraft.hit_result` | `world/phys/HitResult` | `getType()`, `getLocation()` |
| `minecraft.hit_result_type` | `world/phys/HitResult$Type` | `MISS`, `BLOCK`, `ENTITY` |
| `minecraft.block_hit_result` | `world/phys/BlockHitResult` | `getBlockPos()`, `getDirection()`, `isInside()`, `<init>(4)` |
| `minecraft.block_pos` | `core/BlockPos` | `<init>(3)`, `getX()`, `getY()`, `getZ()` |
| `minecraft.direction` | `core/Direction` | `get3DDataValue()`, `from3DDataValue()` |
| `minecraft.entity_hit_result` | `world/phys/EntityHitResult` | `getEntity()` |
| `minecraft.vec3i` | `core/Vec3i` | — |
| `minecraft.block_state` | `world/level/block/state/BlockState` | `isAir()`, `canBeReplaced()`, `getCollisionShape()`, `getFluidState()`, `getBlock()` |
| `minecraft.block_state_base` | `world/level/block/state/BlockBehaviour$BlockStateBase` | — |
| `minecraft.block_getter` | `world/level/BlockGetter` | — |
| `minecraft.voxel_shape` | `world/phys/shapes/VoxelShape` | `isEmpty()` |
| `minecraft.fluid_state` | `world/level/material/FluidState` | `isEmpty()` |
| `minecraft.block` | `world/level/block/Block` | `getDescriptionId()` |
| `minecraft.block_behaviour` | `world/level/block/state/BlockBehaviour` | — |
| `minecraft.clip_context` | `world/level/ClipContext` | `<init>(5)` |
| `minecraft.clip_context_block` | `world/level/ClipContext$Block` | `COLLIDER`, `OUTLINE` |
| `minecraft.clip_context_fluid` | `world/level/ClipContext$Fluid` | `NONE`, `ANY` |
| `minecraft.camera` | `client/Camera` | `getPosition()`→`position()`, `getYRot()`→`yRot()`, `getXRot()`→`xRot()`, `isDetached()` |
| `minecraft.options` | `client/Options` | `keyAttack`, `keyUse`, `keyUp`, `keyDown`, `keyLeft`, `keyRight`, `keyJump`, `keyShift`, `keySprint`, `keyDrop`, `keySwapOffhand`, `keyHotbarSlots`, `fov()`, `gamma()`, `sensitivity()` |
| `minecraft.option_instance` | `client/OptionInstance` | `value`, `get()` |
| `minecraft.screen` | `client/gui/screens/Screen` | — |
| `minecraft.chat_screen` | `client/gui/screens/ChatScreen` | — |
| `minecraft.inventory_screen` | `client/gui/screens/inventory/InventoryScreen` | — |
| `minecraft.creative_mode_inventory_screen` | `client/gui/screens/inventory/CreativeModeInventoryScreen` | — |
| `minecraft.abstract_container_screen` | `client/gui/screens/inventory/AbstractContainerScreen` | — |
| `minecraft.non_null_list` | `core/NonNullList` | — |
| `minecraft.slot` | `world/inventory/Slot` | `getItem()` |
| `minecraft.mob_effect_instance` | `world/effect/MobEffectInstance` | `getEffect()`, `getAmplifier()`, `getDuration()` |
| `minecraft.attribute_map` | `world/entity/ai/attributes/AttributeMap` | `attributes` |
| `minecraft.attribute_instance` | `world/entity/ai/attributes/AttributeInstance` | `getAttribute()`, `getBaseValue()`, `getValue()`, `setBaseValue()` |
| `minecraft.client_packet_listener` | `client/multiplayer/ClientPacketListener` | `getOnlinePlayers()`, `sendChat()`, `sendCommand()`, `send()`, `handleSetEntityMotion()`, `handleTeleportEntity()`, `handleMoveEntity()` |
| `minecraft.player_info` | `client/multiplayer/PlayerInfo` | `getProfile()`, `getLatency()`, `getGameMode()`, `getTabListDisplayName()` |
| `minecraft.key_mapping` | `client/KeyMapping` | `clickCount`, `key`, `isDown()`, `setDown()` |
| `minecraft.input_constants_key` | `blaze3d/platform/InputConstants$Key` | `getValue()`, `getType()` |
| `minecraft.input_constants_type` | `blaze3d/platform/InputConstants$Type` | `MOUSE` |
| `minecraft.window` | `blaze3d/platform/Window` | `getWindow()`→`handle()` |
| `minecraft.item_cooldowns` | `world/item/ItemCooldowns` | — |
| `minecraft.delta_tracker_timer` | `client/DeltaTracker$Timer` | `msPerTick` |
| `minecraft.tick_rate_manager` | `world/TickRateManager` | `tickrate()`, `setTickRate()` |
| `minecraft.mouse_handler` | `client/MouseHandler` | `accumulatedDX`, `accumulatedDY`, `turnPlayer()` |
| `minecraft.interaction_hand` | `world/InteractionHand` | `MAIN_HAND`, `OFF_HAND` |
| `minecraft.interaction_result` | `world/InteractionResult` | — |
| `minecraft.client_common_packet_listener_impl` | `client/multiplayer/ClientCommonPacketListenerImpl` | — |
| `minecraft.packet` | `network/protocol/Packet` | — |
| `minecraft.set_entity_motion_packet` | `network/protocol/game/ClientboundSetEntityMotionPacket` | `getId()`, `<init>(2)` |
| `minecraft.teleport_entity_packet` | `network/protocol/game/ClientboundTeleportEntityPacket` | `getId()`→`id()`, `isOnGround()`→`onGround()` |
| `minecraft.move_entity_packet` | `network/protocol/game/ClientboundMoveEntityPacket` | `getEntity()`, `getXa()`, `getYa()`, `getZa()`, `hasPosition()`, `hasRotation()`, `isOnGround()` |
| `minecraft.vec_delta_codec` | `network/protocol/game/VecDeltaCodec` | `decode()` |
| `minecraft.move_player_packet` | `network/protocol/game/ServerboundMovePlayerPacket` | `x`, `y`, `z`, `yRot`, `xRot`, `onGround`, `hasPos`, `hasRot` |
