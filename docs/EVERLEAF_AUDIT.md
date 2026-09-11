# EverLeaf OpenStory Compatibility Audit

Baseline audited: `master` at `2df47de30ed443891694642d07a27ec15dcc1fcb`

## Summary

OpenStory is a strong base for the EverLeaf v83 client. Its core v83 packet tables, login flow, character creation packet shape, map/channel handoff, UI systems, renderer, input, and NX-backed data model align closely with the current EverLeaf server.

The safest migration path is to preserve the existing v83 protocol behavior first, remove or gate OpenStory-specific custom packets, rebrand/configure the client for EverLeaf, then validate login -> world/channel -> character select -> map entry before enabling additional client features.

## Repository / build

- C++17 client built with CMake.
- Windows build target: `OpenStory`.
- Uses OpenGL + GLFW.
- Bundled dependencies include NoLifeNx, GLEW, FreeType, BASS, stb, and GLFW.
- Client data is NX-based and expected in the runtime `wz/` directory.
- Existing GitHub Actions workflow builds the Windows client and .NET 8 launcher and publishes a rolling prerelease.
- Existing launcher is WPF/.NET 8 with SHA-256 file validation and atomic replacement.

## Core server compatibility

The current EverLeaf server uses the same v83/Odin/Cosmic-derived opcode model that OpenStory targets. Core inbound and outbound values line up across login, world/channel selection, character operations, gameplay, inventory, NPCs, mobs, social systems, shops, storage, cash shop, and movement/combat.

Examples verified against the current EverLeaf server:

| Flow | Client opcode | Server opcode | Result |
| --- | ---: | ---: | --- |
| Login | `0x01` | `LOGIN_PASSWORD 0x01` | match |
| Character list | `0x05` | `CHARLIST_REQUEST 0x05` | match |
| Server status | `0x06` | `SERVERSTATUS_REQUEST 0x06` | match |
| Server list | `0x0B` | `SERVERLIST_REQUEST 0x0B` | match |
| Character select | `0x13` | `CHAR_SELECT 0x13` | match |
| Channel login | `0x14` | `PLAYER_LOGGEDIN 0x14` | match |
| Name check | `0x15` | `CHECK_CHAR_NAME 0x15` | match |
| Character create | `0x16` | `CREATE_CHAR 0x16` | match |
| Character delete | `0x17` | `DELETE_CHAR 0x17` | match |
| PIC register | `0x1D` | `REGISTER_PIC 0x1D` | match |
| PIC select | `0x1E` | `CHAR_SELECT_WITH_PIC 0x1E` | match |
| Change map | `0x26` | `CHANGE_MAP 0x26` | match |
| Change channel | `0x27` | `CHANGE_CHANNEL 0x27` | match |
| Enter cash shop | `0x28` | `ENTER_CASHSHOP 0x28` | match |

The server-to-client login opcodes also match the OpenStory packet switch: login result `0x00`, server status `0x03`, server list `0x0A`, character list `0x0B`, server IP `0x0C`, name response `0x0D`, new-character response `0x0E`, delete-character response `0x0F`, channel change `0x10`, and ping `0x11`.

## Character creation

OpenStory sends:

```text
name:string
job:int
face:int
hair:int
hairColor:int
skin:int
top:int
bottom:int
shoes:int
weapon:int
gender:byte
```

EverLeaf's current `CreateCharHandler` reads the same fields in the same order and widths.

The current EverLeaf server supports these creation-family values:

- `0` - Cygnus
- `1` - Explorer
- `2` - Aran
- `3` - Evan

This makes OpenStory's source-level character creation UI a much cleaner place to extend EverLeaf than continuing to binary-patch the stock v83 executable.

## PIC / hardware handoff

OpenStory's select/register PIC packets include character ID plus its hardware-info strings. EverLeaf's `RegisterPicHandler` and `CharSelectedWithPicHandler` expect the same general structure: character ID, MAC/hardware strings, host/HWID string, and PIC where applicable.

EverLeaf's current delete-character handler intentionally consumes the legacy PIC string but does not gate deletion on PIC, which is compatible with OpenStory's delete packet shape.

One remaining validation item is to compare OpenStory's exact `write_hardware_info()` string generation with EverLeaf's `Hwid.fromHostString()` expectations during the first live compatibility test.

## OpenStory-specific packets to gate initially

These should not be enabled in the initial EverLeaf compatibility build because the current server does not expose matching handlers/opcodes:

- `LOGIN_START` (`0x23`) - OpenStory custom login-start packet.
- `REQUEST_EVENT_INFO` (`0xF1`) and inbound `EVENT_INFO` (`0xC3`).
- Monster Life custom operations (`0x400`) and updates (`0x190`).
- Monster Battle custom operations (`0x401`) and updates (`0x191`).
- OpenStory-specific bot inventory operation where used.

Initial EverLeaf integration should feature-gate or suppress these instead of changing core v83 opcode numbers.

## Branding / configuration debt

Current fork still contains upstream project/server defaults that must be changed for an EverLeaf build:

- window/project title `OpenStory`
- default server `72.60.176.12:8484`
- Nexon website/account/password/PIC/NX URLs
- GenMs launcher branding
- GenMs update/feed/news endpoints
- launcher executable/client names
- CI artifact names
- icons/logo/resources

The launcher currently contains a TLS exception scoped to the old GenMs IP; EverLeaf integration should remove that exception and use normal HTTPS validation against EverLeaf endpoints.

## NX data dependency

OpenStory currently compiles with `USE_NX` and reads data through NoLifeNx. EverLeaf therefore needs a repeatable canonical v83 WZ -> NX build/export pipeline for releases. Do not hand-maintain player NX files independently from the canonical EverLeaf content source.

## Existing useful QoL / modernization

OpenStory already supplies or has source-level implementations for several goals on the EverLeaf roadmap:

- resizable/fullscreen rendering foundation
- UI scaling
- modern OpenGL rendering
- gamepad support
- Unicode input and RTL text support
- drag/drop UI windows
- quest helper
- modernized chat/icons/emotes
- spatial audio
- source-level login/world/character selection
- source-level character creation
- modern launcher/update flow

## Initial integration gates

Do not replace the production client until all of these pass against the current EverLeaf server:

1. Clean Windows Release build.
2. Client boots with canonical EverLeaf NX data.
3. Login succeeds.
4. World/channel list parses correctly.
5. Character list parses correctly.
6. Existing character enters a channel/map.
7. Movement/chat/inventory/skills function without protocol errors.
8. Create Explorer, Cygnus, Aran, and Evan test characters.
9. Delete-character flow works with EverLeaf's current PIC policy.
10. Channel change, map change, logout/relog, and reconnect work.
11. Launcher updates client and NX files from EverLeaf manifests.
12. No GenMs/OpenStory endpoint or branding remains in the player build.

## Integration approach

Use `everleaf-integration` as the working branch. Keep `master` as the baseline until the compatibility build is proven.

Recommended order:

1. Compatibility flags / suppress unsupported custom packets.
2. EverLeaf endpoint and title configuration.
3. Launcher rebrand and manifest endpoint migration.
4. CI artifact rename and build verification.
5. NX pipeline validation.
6. Live-server smoke test.
7. EverLeaf UI/branding migration.
8. Additional client QoL and custom protocol work only after baseline parity is stable.
