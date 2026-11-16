# Combat, use and blocks

All go through `MultiPlayerGameMode`, as the game does for mouse input.

## Types

| Type | Content | Units and ranges |
| --- | --- | --- |
| `Hand` | `Main`, `Off` | — |

## Operations

### `attack(entity_id) -> Option<()>`

Attacks the entity, then swings the main hand (vanilla order).

- **Out of world:** `None`
- **Criteria:** unknown id → `InvalidArgument`; the swing is visible
- **Symbols:** `client_level.get_entity`, `multi_player_game_mode.attack`, `living_entity.swing`,
  `interaction_hand.*`

### `use_item(Hand) -> Option<()>`

Uses the item in hand.

- **Out of world:** `None`
- **Criteria:** e.g. eating starts
- **Symbols:** `multi_player_game_mode.use_item`, `interaction_result`

### `use_on(Hand, Hit) -> Option<()>`

Uses the item on a block hit.

- **Out of world:** `None`
- **Criteria:** non-`Block` hit → `InvalidArgument`
- **Symbols:** `multi_player_game_mode.use_item_on`, `block_hit_result.new`

### `release_use() -> Option<()>`

Releases the item being used.

- **Out of world:** `None`
- **Criteria:** bow fires
- **Symbols:** `multi_player_game_mode.release_using_item`

### `interact(entity_id, Hand) -> Option<()>`

Interacts with an entity.

- **Out of world:** `None`
- **Criteria:** unknown id → `InvalidArgument`
- **Symbols:** `multi_player_game_mode.interact`

### `start_destroy(BlockPos, Direction)` / `continue_destroy(BlockPos, Direction)` / `stop_destroy()` `-> Option<()>`

Block breaking as with a held attack key.

- **Out of world:** `None`
- **Criteria:** progress reported by `destroy_progress`
- **Symbols:** `multi_player_game_mode.start_destroy_block`, `continue_destroy_block`,
  `stop_destroy_block`

### `destroy_progress() -> Option<f32>`

Current block-breaking progress.

- **Out of world:** `None`
- **Criteria:** 0 when not breaking
- **Symbols:** `multi_player_game_mode.destroy_progress`, `multi_player_game_mode.is_destroying`

### `swing(Hand) -> Option<()>`

Arm swing animation and packet.

- **Out of world:** `None`
- **Criteria:** swing visible
- **Symbols:** `living_entity.swing`

## Tests

| Test | Expected |
| --- | --- |
| `attack(id)` | the target is hit and the main hand swings |
