# Errors and the out-of-world rule

Every operation returns `Result<_, BridgeError>`. `BridgeError` variants:

| Variant | Meaning |
| --- | --- |
| `Disconnected` | session closed or never opened |
| `UnsupportedEnvironment` | configuration not in mc-mappings, carries the detected environment |
| `Mapping` | mc-mappings `ResolveError`, carries the semantic id |
| `Java` | Java exception, converted to class name and message |
| `InvalidArgument` | with the argument name and reason |
| `GameThreadUnavailable` | the game no longer processes operations |
| `Cancelled` | — |

Operations that need a world or a local player return `Ok(None)` when there is none (title
screen, loading, disconnected from a server): `Result<Option<T>>`. They never return default
values instead. Arguments are validated before any game call: non-finite numbers, out-of-range
slots, hotbar indices, radii, scales or text are `InvalidArgument`.
