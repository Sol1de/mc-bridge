# Local player

## Types

| Type | Content | Units and ranges |
| --- | --- | --- |
| `PlayerSnapshot` | `entity: EntitySnapshot`, `max_health: f32`, `fall_distance: f64`, `food: i32`, `saturation: f32`, `using_item: bool`, `use_ticks_remaining: i32`, `game_mode: GameMode`, `flying`, `may_fly`, `instabuild: bool`, `attack_strength: f32` | food 0..=20; `attack_strength` in [0, 1] (partial tick 0) |

## Operations

### `player() -> Option<PlayerSnapshot>`

Local player state.

- **Out of world:** `None`
- **Criteria:** values match the debug screen (F3); `attack_strength` in [0, 1]
- **Symbols:** entity readers (see [`entities()`](world.md)), `player.get_food_data`,
  `food_data.*`, `living_entity.is_using_item`, `living_entity.get_use_item_remaining_ticks`,
  `multi_player_game_mode.get_player_mode`, `game_type.*`, `player.get_abilities`,
  `abilities.*`, `player.get_attack_strength_scale`, `entity.fall_distance` (versioned type)

### `set_rotation(Rotation) -> Option<()>`

Writes `yRot`, `xRot`, `yRotO` and `xRotO` (no interpolation from the previous rotation).

- **Out of world:** `None`
- **Criteria:** non-finite or pitch outside [-90, 90] → `InvalidArgument`; camera turns at once
- **Symbols:** `entity.y_rot`, `entity.x_rot`, `entity.y_rot_o`, `entity.x_rot_o`

### `turn(dyaw, dpitch) -> Option<()>`

Feeds the delta to `MouseHandler.turnPlayer` as mouse movement (current sensitivity and mouse
inversion apply, as for a real mouse); pitch clamps at ±90.

- **Out of world:** `None`
- **Criteria:** non-finite → `InvalidArgument`; resulting rotation differs from the request only
  by the game's mouse quantisation
- **Symbols:** `client.mouse_handler`, `mouse_handler.accumulated_dx`,
  `mouse_handler.accumulated_dy`, `mouse_handler.turn_player`

### `set_velocity(Vec3) -> Option<()>`

Sets the local player's motion for its next movement.

- **Out of world:** `None`
- **Criteria:** non-finite → `InvalidArgument`; movement observed next tick. Called from the tick
  hook it does **not** undo a knockback already integrated in this tick's movement (use the
  `EntityMotion` [packet hook](packets.md) for that)
- **Symbols:** `entity.set_delta_movement`

### `set_position(Vec3) -> Option<()>`

Moves the local player (client side).

- **Out of world:** `None`
- **Criteria:** non-finite → `InvalidArgument`; the server may correct it
- **Symbols:** `entity.set_pos`

### `set_on_ground(bool) -> Option<()>`

Sets the client on-ground flag.

- **Out of world:** `None`
- **Criteria:** sent with the next movement packet; the server may reject it
- **Symbols:** `entity.set_on_ground`

### `set_sprinting(bool) -> Option<()>`

Starts or stops sprinting through the player's own state change (the game sends the sprint
command).

- **Out of world:** `None`
- **Criteria:** sprint state and server view follow
- **Symbols:** `local_player.set_sprinting`

### `set_sneaking(bool) -> Option<()>`

Holds or releases the logical `Sneak` key, so the game's input produces sneaking and its packets.

- **Out of world:** `None`
- **Criteria:** sneaking follows within one tick
- **Symbols:** key symbols (see [input.md](input.md))

### `respawn() -> Option<()>`

Requests a respawn.

- **Out of world:** `None`
- **Criteria:** player not dead → `InvalidArgument`
- **Symbols:** `local_player.respawn`
