# Session

The subscriptions (`subscribe_pre_tick`, `subscribe_ticks`, `subscribe_frames`) are `Session`
operations described in [threading.md](../core/threading.md).

## Types

| Type | Content | Units and ranges |
| --- | --- | --- |
| `Environment` | `minecraft_version: String`, `launcher: Launcher` (`Standard`, `Lunar`), `loader: Loader` (`None`, `Fabric`, `Forge`) | as selected in mc-mappings |
| `UserInfo` | `name: String`, `uuid: Uuid` (128-bit, owned) | — |

## Operations

### `Bridge::connect() -> Session`

Borrows the JVM through jvm-link, detects the environment, selects the mc-mappings profile, checks
the profile's `minecraft.client` class loads.

- **Out of world:** n/a
- **Criteria:** succeeds on every matrix configuration; unknown or ambiguous environment →
  `UnsupportedEnvironment`; partial failure leaves nothing open
- **Symbols:** `client`

### `Session::disconnect()`

Releases held packets, ends every subscription, frees session references, then closes the
jvm-link connection; later calls return `Disconnected`.

- **Out of world:** n/a
- **Criteria:** idempotent; the game keeps running
- **Symbols:** —

### `Session::environment()`

Detected environment, no game call.

- **Out of world:** n/a
- **Criteria:** equals the running launcher/loader/version
- **Symbols:** —

### `Session::user() -> UserInfo`

Signed-in account.

- **Out of world:** n/a
- **Criteria:** name and UUID match the launcher account
- **Symbols:** `client.get_instance`, `client.get_user`, `user.*`

### `Session::in_world() -> bool`

A client level and a local player exist.

- **Out of world:** `false`
- **Criteria:** flips on join/leave
- **Symbols:** `client.level`, `client.player`

### `Session::run_on_game_thread(f)`

Runs `f` on the game thread and returns its result.

- **Out of world:** n/a
- **Criteria:** inline when already on it
- **Symbols:** `blockable_event_loop.*`
