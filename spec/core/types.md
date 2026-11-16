# Owned types and units

Types shared by several domains. A type used by one domain only is listed in that domain's file.

| Type | Content | Units and ranges |
| --- | --- | --- |
| `Vec3` | `x`, `y`, `z: f64` | blocks (positions), blocks per tick (velocity) |
| `Rotation` | `yaw`, `pitch: f32` | degrees; `pitch` in [-90, 90], `yaw` unbounded |
| `Aabb` | `min`, `max: Vec3` | blocks, world coordinates |
| `BlockPos` | `x`, `y`, `z: i32` | block coordinates |
| `Direction` | `Down`, `Up`, `North`, `South`, `West`, `East` | the game's 3D data values 0..=5 |
| `GameMode` | `Survival`, `Creative`, `Adventure`, `Spectator` | — |
| `EquipmentInfo` | `main_hand`, `off_hand: Option<ItemStackInfo>`, `armor: [Option<ItemStackInfo>; 4]` | armor feet → legs → chest → head |
| `EntitySnapshot` | `entity_id: i32`, `uuid`, `type_id: String` (translation key, e.g. `entity.minecraft.zombie`), `position`, `previous_position: Vec3`, `velocity: Vec3`, `rotation`, `aabb: Aabb`, `eye_height: f32`, `tick_count: i32`, `alive`, `on_ground: bool`, `sneaking`, `sprinting`, `swimming`, `gliding`, `invisible`, `glowing: bool`, `custom_name: Option<String>`, `profile_name: Option<String>` (players only), `health: Option<f32>`, `absorption: Option<f32>`, `hurt_time: Option<i32>`, `equipment: Option<EquipmentInfo>` (living entities only), `vehicle_id: Option<i32>` | `previous_position` is the position at the start of the current tick; `hurt_time` counts down from 10 |
| `ItemStackInfo` | `item_id: String` (translation key, e.g. `item.minecraft.stone`), `count: i32`, `damage`, `max_damage: i32`, `enchantments: Vec<(String, i32)>` (id such as `minecraft:sharpness`, level) | count ≥ 1; `max_damage` 0 for items without durability |
| `Hit` | `Miss`, `Block { pos: BlockPos, face: Direction, location: Vec3, inside: bool }`, `Entity { entity_id: i32, location: Vec3 }` | — |

Snapshots are consistent for one call (read on the game thread in one operation) and are never
updated afterwards. `entity_id` identifies an entity only within one world generation.
