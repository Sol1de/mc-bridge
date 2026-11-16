# Inventory and menus

## Types

| Type | Content | Units and ranges |
| --- | --- | --- |
| `InventorySnapshot` | `selected_hotbar_slot: u8`, `main: [Option<ItemStackInfo>; 36]`, `armor: [Option<ItemStackInfo>; 4]`, `offhand: Option<ItemStackInfo>` | `main` 0..=8 is the hotbar; armor feet → head; independent of any open screen |
| `MenuSnapshot` | `container_id: i32`, `slots: Vec<Option<ItemStackInfo>>`, `carried: Option<ItemStackInfo>` | indices are the open menu's (`player.containerMenu`, the player inventory menu when no container is open); they are **not** `inventory()` indices |
| `ClickAction` | `Pickup`, `QuickMove`, `Swap`, `Throw` | — |
| `ScreenKind` | `None`, `Chat`, `Inventory` (survival or creative inventory), `Container` (any other container screen), `Other` | — |

## Operations

### `inventory() -> Option<InventorySnapshot>`

`main[36]` (`Inventory.getItem(0..36)`), armor and offhand from the player's equipment slots;
independent of the open screen.

- **Out of world:** `None`
- **Criteria:** armor and offhand read with a chest closed and open alike
- **Symbols:** `player.get_inventory`, `inventory.get_item`, `inventory.selected`,
  `living_entity.get_item_by_slot`, `equipment_slot.*`, item readers: `item_stack.*`,
  `item.get_description_id`, `item_enchantments.entry_set`, `holder.get_registered_name`

### `select_hotbar_slot(u8) -> Option<()>`

Selects a hotbar slot like scrolling (the game sends the change on its next tick).

- **Out of world:** `None`
- **Criteria:** slot > 8 → `InvalidArgument`
- **Symbols:** `inventory.selected`

### `menu() -> Option<MenuSnapshot>`

`player.containerMenu`: its slots in menu order and the carried stack.

- **Out of world:** `None`
- **Criteria:** indices match `click_slot`
- **Symbols:** `player.container_menu`, `abstract_container_menu.*`, `slot.get_item`, item readers

### `click_slot(slot: i32, button: u8, ClickAction) -> Option<()>`

Clicks slot `slot` **of `player.containerMenu`** through the game's handler (client and server
stay in sync); menu indices, not `inventory()` ones.

- **Out of world:** `None`
- **Criteria:** slot outside the menu or invalid button → `InvalidArgument`; result visible in
  `menu()`
- **Symbols:** `client.game_mode`, `multi_player_game_mode.handle_inventory_mouse_click`,
  `abstract_container_menu.container_id`, `click_type.*`

### `screen() -> ScreenKind`

Kind of the open screen.

- **Out of world:** works on any screen: the title screen is `Other`, `None` only when no screen is
  open
- **Criteria:** chat, inventory, chest distinguished
- **Symbols:** `client.screen`, `chat_screen`, `inventory_screen`,
  `creative_mode_inventory_screen`, `abstract_container_screen`

### `drop(entire_stack: bool) -> Option<()>`

Drops the selected hotbar stack (or one item) like the drop key.

- **Out of world:** `None`
- **Criteria:** count decreases accordingly
- **Symbols:** `local_player.drop`

## Tests

| Test | Expected |
| --- | --- |
| `inventory()` with a chest closed | armor and offhand read |
