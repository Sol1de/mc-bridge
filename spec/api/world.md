# World and entities

## Types

| Type | Content | Units and ranges |
| --- | --- | --- |
| `WorldInfo` | `dimension: String` (resource location, e.g. `minecraft:overworld`) | — |
| `BlockSample` | `block_id: String` (translation key, e.g. `block.minecraft.stone`), `air`, `replaceable`, `collision_empty`, `fluid: bool` | `fluid` true when the block holds a non-empty fluid |
| `TabEntry` | `name: String`, `uuid`, `latency_ms: i32`, `game_mode: Option<GameMode>`, `display_name: Option<String>` | — |

## Operations

### `world() -> Option<WorldInfo>`

Current client level.

- **Out of world:** `None`
- **Criteria:** dimension changes with portals
- **Symbols:** `level.dimension`, `resource_key.location`, `resource_location` (`Object.toString`)

### `entities() -> Option<Vec<EntitySnapshot>>`

**Every entity the client tracks**, in or out of the view frustum, local player included.
`ClientLevel.entitiesForRendering()` returns `getEntities().getAll()` (checked by disassembly on
1.21.11): it is not the rendered count of F3's `E:`.

- **Out of world:** `None`
- **Criteria:** an entity behind the camera is present; removed entities absent
- **Symbols:** `client_level.entities_for_rendering`, entity readers: `entity.*`,
  `living_entity.*`, `aabb.*`, `entity_type.get_description_id`, `component.get_string`,
  `player.get_game_profile`, `equipment_slot.*`, `living_entity.get_item_by_slot`

### `hit() -> Option<Hit>`

Current crosshair target (`Minecraft.hitResult`).

- **Out of world:** `None`
- **Criteria:** follows the crosshair
- **Symbols:** `client.hit_result`, `hit_result.*`, `hit_result_type.*`, `block_hit_result.*`,
  `entity_hit_result.get_entity`, `block_pos.*`, `direction.*`

### `block(BlockPos) -> Option<BlockSample>`

One block state.

- **Out of world:** `None`
- **Criteria:** air and solid blocks are distinct; unloaded positions read as the game reports them
- **Symbols:** `level.get_block_state`, `block_state.*`, `block.get_description_id`,
  `fluid_state.is_empty`, `voxel_shape.is_empty`, `block_pos.new`

### `blocks(center, radius: u8) -> Option<Vec<(BlockPos, BlockSample)>>`

Cube of side `2 × radius + 1`, x then z then y order.

- **Out of world:** `None`
- **Criteria:** `radius > 32` → `InvalidArgument`
- **Symbols:** as `block`

### `raycast(from, to: Vec3, blocks: bool, fluids: bool) -> Option<Hit>`

Block ray from the game's clip (`blocks`: collision shapes when false, outline shapes when true;
`fluids`: stop on any fluid).

- **Out of world:** `None`
- **Criteria:** non-finite → `InvalidArgument`; matches `hit()` when cast along the view
- **Symbols:** `level.clip`, `clip_context.new`, `clip_context_block.*`, `clip_context_fluid.*`,
  `vec3.new`

### `tab() -> Option<Vec<TabEntry>>`

Tab list entries.

- **Out of world:** `None`
- **Criteria:** matches the tab list
- **Symbols:** `client.get_connection`, `client_packet_listener.get_online_players`,
  `player_info.*`

## Tests

| Test | Expected |
| --- | --- |
| Entity behind the camera | present in `entities()` |
| `hit()` | follows the crosshair |
| `block()` on air and on stone | distinct samples (`air` true / false, `collision_empty`) |
