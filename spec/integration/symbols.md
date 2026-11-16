# mc-mappings symbols

Checked on 2026-11-11 against the generated mappings of all ten configurations (standard and
Lunar), with the mc-mappings export and audit. mc-mappings `data_version` 2 (`schema_version` 2)
provides all of them; their Mojang names and types are the generator input
`mc_mappings_generator/consumers/mc_mappings/catalog-v1.json`.

## Symbol count

| Group | Classes | Fields | Methods |
| --- | --- | --- | --- |
| mc-mappings `data_version` 1 (before this revision) | 19 | 13 | 36 |
| Added, common to every configuration (verified) | 58 | 75 | 127 |
| Added, same symbol with a version-specific type | 0 | 1 | 4 |
| Added, 1.21 only | 0 | 0 | 8 |
| Added, 1.21.11 only | 2 | 0 | 10 |
| **Total per 1.21 profile** | **77** | **89** | **175** |
| **Total per 1.21.11 profile** | **79** | **89** | **177** |

## Version-dependent symbols

1. **Same symbol, version-specific type**: the profile of each version carries its own type.
   `entity.fall_distance` (`float` 1.21, `double` 1.21.11),
   `game_renderer.get_fov` (returns `double` / `float`),
   `item_cooldowns.get_cooldown_percent` (`(Item, float)` / `(ItemStack, float)`),
   `move_entity_packet.get_y_rot` and `get_x_rot` (`getyRot()`/`getxRot()` returning `byte` in
   1.21, `getYRot()`/`getXRot()` returning `float` in 1.21.11). mc-bridge converts the value
   according to the resolved descriptor.
2. **Version-scoped symbols** (mc-mappings `catalog.scoped`): they exist in one Minecraft version
   only, and resolving them in the other one gives `AbsentInVersion`. They decode packet content
   whose structure changed:
   - 1.21 only: `set_entity_motion_packet.get_xa/get_ya/get_za` (velocity),
     `teleport_entity_packet.get_x/get_y/get_z/get_y_rot/get_x_rot`.
   - 1.21.11 only: `set_entity_motion_packet.get_movement`, `teleport_entity_packet.change`,
     `teleport_entity_packet.relatives`, `position_move_rotation` (class) `.position/.y_rot/.x_rot`,
     `entity_position_sync_packet` (class) `.id/.values/.on_ground`,
     `client_packet_listener.handle_entity_position_sync`.

## Common symbols (all ten configurations)

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

## Tests

| Test | Expected |
| --- | --- |
| mc-mappings | every `minecraft.*` symbol of the appendix is in the mc-mappings catalog and resolves on every configuration of the versions it exists in (audit) |
