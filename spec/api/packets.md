# Packets

`subscribe_packets(kinds: &[PacketKind], cb) -> Subscription`; `cb(&mut PacketEvent) ->
PacketDecision`. Only the three kinds below exist; there is no generic packet access and no bytes.

## Types

| Type | Content | Units and ranges |
| --- | --- | --- |
| `PacketKind` | `EntityMotion`, `EntityPosition`, `MovePlayer` | — |
| `PacketEvent` | `EntityMotion { entity_id, velocity: Vec3 }` (velocity mutable), `EntityPosition { entity_id, position: Vec3, rotation: Rotation, on_ground: bool, teleport: bool }` (read-only), `MovePlayer { position: Option<Vec3>, rotation: Option<Rotation>, on_ground: bool }` (mutable; `None` when the packet does not carry it) | — |
| `PacketDecision` | `Pass`, `Drop`, `Hold` | — |

## Kinds

### `EntityMotion`

- **Direction and point:** incoming `ClientboundSetEntityMotionPacket`, at
  `ClientPacketListener.handleSetEntityMotion` on the game thread
- **Decision:** `velocity` mutable: a changed velocity replaces the packet by
  `new(entity_id, velocity)` before the game handles it

### `EntityPosition`

- **Direction and point:** incoming teleport and relative-move packets (`handleTeleportEntity`,
  `handleMoveEntity`; 1.21.11 also `handleEntityPositionSync`); relative moves are decoded with the
  entity's `VecDeltaCodec`; `teleport` is true for teleports
- **Decision:** read-only

### `MovePlayer`

- **Direction and point:** outgoing `ServerboundMovePlayerPacket`, at
  `ClientPacketListener.send(Packet)`
- **Decision:** `position`, `rotation`, `on_ground` mutable (written to the packet's fields; absent
  parts stay absent)

## Decisions and held packets

- `Pass` lets the packet through, `Drop` discards it, `Hold` keeps it.
- `release_held(kind)` delivers the held packets of that kind in arrival order (incoming: handled
  on the game thread; outgoing: sent without re-entering the hook). Unsubscribe and
  `Session::disconnect` release everything held. Packets held when the game's server connection
  closes are discarded (nothing to deliver to).
- Callbacks run on the game thread and must not block; mutations are validated (non-finite →
  the packet passes unchanged and the error is reported to the subscription).

## Tests

| Test | Expected |
| --- | --- |
| `Hold` on `MovePlayer` | the server-side position freezes until `release_held(MovePlayer)` |
| `EntityMotion` velocity set to zero | the entity does not move from that packet |
