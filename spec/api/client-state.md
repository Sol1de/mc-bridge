# Camera, timing and options

## Types

| Type | Content | Units and ranges |
| --- | --- | --- |
| `Camera` | `position: Vec3`, `rotation: Rotation`, `fov_degrees: f32`, `third_person: bool` | values of the last rendered frame |
| `Effect` | `id: String` (e.g. `minecraft:speed`), `amplifier: i32`, `duration_ticks: i32` | amplifier 0 = level I; duration −1 = infinite |
| `Attribute` | `id: String` (e.g. `minecraft:movement_speed`), `base`, `value: f64` | `value` includes modifiers |
| `GameOptions` | `sensitivity: f64`, `fov_degrees: i32`, `gamma: f64` | sensitivity 0..=1, FOV 30..=110, gamma as set |

## Operations

### `camera() -> Option<Camera>`

Camera of the last rendered frame: position, rotation, effective FOV (`GameRenderer.getFov`),
third person (`Camera.isDetached`).

- **Out of world:** `None`
- **Criteria:** follows F5 and FOV effects
- **Symbols:** `game_renderer.get_main_camera`, `camera.*`, `game_renderer.get_fov` (versioned
  type)

### `effects() -> Option<Vec<Effect>>`

Active effects of the local player.

- **Out of world:** `None`
- **Criteria:** match the inventory screen
- **Symbols:** `living_entity.get_active_effects`, `mob_effect_instance.*`,
  `holder.get_registered_name`

### `attributes() -> Option<Vec<Attribute>>` / `set_attribute(id, base: f64) -> Option<()>`

Attribute instances of the local player; the setter writes the base value (client side).

- **Out of world:** `None`
- **Criteria:** unknown id or non-finite → `InvalidArgument`; the server may recompute
- **Symbols:** `living_entity.get_attributes`, `attribute_map.attributes`, `attribute_instance.*`

### `item_cooldown() -> Option<f32>`

Cooldown fraction of the main-hand item.

- **Out of world:** `None`
- **Criteria:** in [0, 1]; 0 without cooldown
- **Symbols:** `player.get_cooldowns`, `item_cooldowns.get_cooldown_percent` (versioned
  parameter), `living_entity.get_main_hand_item`

### `right_click_delay()` / `set_right_click_delay(i32)`

Game's use cooldown in ticks.

- **Out of world:** works on the title screen
- **Criteria:** negative → `InvalidArgument`
- **Symbols:** `client.right_click_delay`

### `miss_time()` / `set_miss_time(i32)`

Game's attack miss cooldown in ticks.

- **Out of world:** as above
- **Criteria:** negative → `InvalidArgument`
- **Symbols:** `client.miss_time`

### `tick_scale() -> f32` / `set_tick_scale(f32)`

Client tick speed relative to vanilla (`1.0` = vanilla). The setter writes the client timer's
`msPerTick` **and** the client level's tick rate, since the timer uses the larger of the two
(`Math.max`, checked by disassembly); re-applied when the level or the server's ticking state
changes.

- **Out of world:** scale kept on the title screen
- **Criteria:** `<= 0` or non-finite → `InvalidArgument`; `set_tick_scale(2.0)` doubles ticks per
  second
- **Symbols:** `client.timer`, `delta_tracker_timer.ms_per_tick`,
  `client_level.tick_rate_manager`, `tick_rate_manager.*`

### `options() -> GameOptions` / `set_gamma(f64)`

Sensitivity, FOV and gamma; `set_gamma` writes the option's value without range check or saving.

- **Out of world:** —
- **Criteria:** non-finite or negative → `InvalidArgument`
- **Symbols:** `options.sensitivity`, `options.fov`, `options.gamma`, `option_instance.get`,
  `option_instance.value`

### `send_chat(String) -> Option<()>`

Sends a chat message, or a command when it starts with `/`.

- **Out of world:** `None`
- **Criteria:** empty, over 256 characters or containing control characters → `InvalidArgument`
- **Symbols:** `client_packet_listener.send_chat`, `client_packet_listener.send_command`

## Tests

| Test | Expected |
| --- | --- |
| `set_tick_scale(2.0)` | twice the ticks per second |
