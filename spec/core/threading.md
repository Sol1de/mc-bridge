# Thread, ticks and frames

- Every read or write of game state runs on the game (render) thread. Operations called from
  another thread are submitted to it and wait for the result; called from the game thread they run
  inline (no deadlock, re-entrant).
- Submission uses `BlockableEventLoop.execute(Runnable)` on the client; `isSameThread()` decides
  inline execution. The native `Runnable` mechanism, bounds, cancellation and shutdown are
  implementation choices outside this catalog.
- The hooks below are fixed here; the hooking mechanism is an implementation choice.

## Types

| Type | Content | Units and ranges |
| --- | --- | --- |
| `Frame` | `partial_tick: f32` | in [0, 1) |
| `GameTick` | `index: u64` | counted from the subscription |

## Hooks

| Hook | Point | Order |
| --- | --- | --- |
| `subscribe_pre_tick(cb)` | head of `LocalPlayer.sendPosition()` | after the player's movement of the tick, **before** the movement packet; not while riding (the game sends vehicle packets instead) |
| `subscribe_ticks(cb)` | end of `Minecraft.tick()` | **after** the movement packet; also on the title screen |
| `subscribe_frames(cb)` | return of `GameRenderer.pick(float)` | once per rendered frame, after the crosshair target is updated; `Frame.partial_tick` is `pick`'s argument |

Callbacks run on the game thread; no callback runs after its unsubscribe returns.

## Operations

### `Session::subscribe_pre_tick(cb)` / `subscribe_ticks(cb)` / `subscribe_frames(cb)`

See [hooks](#hooks); return a `Subscription`.

- **Out of world:** ticks continue on the title screen; pre-tick and frames need a player / a world
- **Criteria:** pre-tick runs before the movement packet, tick after; 20 ± 1 ticks per second at
  scale 1
- **Symbols:** `local_player.send_position`, `client.tick`, `client.game_renderer`,
  `game_renderer.pick`

## Tests

| Test | Expected |
| --- | --- |
| Hook order | pre-tick before the movement packet, tick after |
