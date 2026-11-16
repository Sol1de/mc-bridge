# Input

## Types

| Type | Content | Units and ranges |
| --- | --- | --- |
| `Key` | `Attack`, `Use`, `Forward`, `Back`, `Left`, `Right`, `Jump`, `Sneak`, `Sprint`, `Drop`, `SwapOffhand`, `Hotbar(u8)` | `Hotbar` 0..=8; each maps to the player's key binding |

## Operations

### `input() -> Vec<(Key, bool)>`

Logical state of every `Key` binding (`KeyMapping.isDown`).

- **Out of world:** works on any screen
- **Criteria:** reflects `set_key`
- **Symbols:** `client.options`, `options.key_*`, `key_mapping.is_down`

### `physical_key(Key) -> bool`

Physical state of the bound key or mouse button, read from GLFW with the game window
(`glfwGetKey` / `glfwGetMouseButton`, external).

- **Out of world:** works on any screen
- **Criteria:** stays true while held even after `set_key(Attack, false)`
- **Symbols:** `key_mapping.key`, `input_constants_key.*`, `input_constants_type.mouse`,
  `client.get_window`, `window.get_window`

### `set_key(Key, bool)`

Sets the logical state until changed or until the next physical event on that binding.

- **Out of world:** —
- **Criteria:** the game acts on it next tick
- **Symbols:** `key_mapping.set_down`

### `click_key(Key)`

Counts one press, consumed by the game like a real press.

- **Out of world:** —
- **Criteria:** one action per click
- **Symbols:** `key_mapping.click_count`

## Tests

| Test | Expected |
| --- | --- |
| `set_key(Attack, false)` while the attack button is held | `physical_key(Attack)` stays true |
