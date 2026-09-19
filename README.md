# PoplarCompat

PoplarCompat is a proof of concept for rendering server-authoritative Minecraft 26.3 Poplar blocks on a Minecraft 1.21.11 Fabric client connected through ViaVersion/ViaBackwards.

The Paper plugin keeps the real 26.3 blocks untouched. After a compatible client completes the handshake on `poplarcompat:main`, the plugin sends position and block-state updates. The client stores those positions and redirects the normal terrain section model lookup to a Poplar model while retaining the ViaBackwards block state for collision, interaction, lighting and networking.

## Built artifacts

Run from this directory:

```powershell
.\gradle-9.7.1\bin\gradle.bat verifyArtifacts --no-daemon
# Optional exact-name release copies:
.\gradle-9.7.1\bin\gradle.bat packageRelease --no-daemon
```

Artifacts:

- `paper/build/libs/PoplarCompat-Paper-26.3-0.1.0.jar`
- `fabric/build/libs/PoplarCompat-Fabric-1.21.11-0.1.0.jar`
- `build/release/PoplarCompat-Paper-26.3.jar`
- `build/release/PoplarCompat-Fabric-1.21.11.jar`

The version suffix is Gradle's normal archive version. These are the files to install.

## Server setup

1. Run Paper 26.3 with Java 25 (the supplied Paper API is class-file version 69).
2. Install ViaVersion 5.12.0 and ViaBackwards 5.12.0 as usual.
3. Copy the Paper artifact to `plugins/` and restart.
4. The plugin declares `depend: [ViaVersion, ViaBackwards]` so a missing compatibility stack is visible at startup.
5. Use `/poplarcompat status`, `/poplarcompat debug <player>`, or `/poplarcompat resend [player]`.

Players without the Fabric mod are allowed to join and continue using the normal ViaBackwards rendering path.

## Client setup

1. Install Fabric Loader 0.19.5 for Minecraft 1.21.11 and a Fabric API release that supports 1.21.11. The build is pinned to `fabric-api 0.141.6+1.21.11`.
2. Copy the Fabric artifact to the client's `mods/` directory.
3. Keep the existing ViaBackwards-Plus resources and modpack. PoplarCompat does not register a networked Poplar block.
4. Start the client and connect to the Paper server. The log should contain `Successfully connected to PoplarCompat server` after the handshake.

## Local model assets

The current POC uses the Poplar model and texture files supplied in the workspace under the parent `pack-src` directory. They are copied into the development resources so the built JAR is immediately testable. They are ignored by this project's `.gitignore` because they are generated/supplied Minecraft assets.

To import supplied assets again:

```powershell
.\tools\import-poplar-assets.ps1 -SourceRoot "..\pack-src"
```

The script only copies the local assets; it does not download packs or game files.

## Protocol

The shared binary protocol is in `common/src/main/java/dev/poplarcompat/common/PoplarProtocol.java`.

- Channel: `poplarcompat:main`
- Protocol version: `1`
- Framed messages: `SERVER_HELLO`, `CLIENT_HELLO`, `SERVER_STATUS`, `BLOCK_SET`, `BLOCK_REMOVE`, `WORLD_CLEAR`
- Payloads are bounded (16 KiB frame, 4 KiB strings) and malformed/unknown versions are ignored.

The first proof of concept sends a server hello, receives a client hello, and sends a status response. The same channel carries position updates. The live test still needs to be run on your actual 26.3 + ViaVersion/ViaBackwards server because no server instance is available in this workspace.

## Rendering approach

The Fabric client uses a position-aware mixin around the 1.21.11 Yarn `SectionBuilder`/`BlockRenderManager` path. It swaps only the baked model returned for a tracked position. It does not create entities, overlays, fake registry blocks, or world block replacements. Section rebuilds are scheduled around each update.

The current POC uses the vanilla note block as a carrier state and the supplied `minecraft:note_block` model mapping to make one Poplar plank model available. This is intentionally small and position-aware, but the carrier resource mapping is global and should be replaced with standalone `assets/poplarcompat/` model loading before a broad block-family release.

## Current scope and limitations

This build proves the end-to-end handshake and server-provided position-based model override. It currently sends updates for Poplar block place/break events and a manual resend command. It does not yet scan chunks, send chunk snapshots, observe every plugin/WorldEdit/piston/explosion mutation, or implement the complete Poplar state family. Door/trapdoor/fence/stair state manifests, resource reload cache rebuilding, client status command, and particle/pick-block polish remain follow-up work.

The client intentionally falls back to normal rendering when the plugin, handshake, protocol version, or position data is unavailable.

## Test checklist

- [ ] Paper 26.3 + ViaVersion 5.12.0 + ViaBackwards 5.12.0 starts with PoplarCompat enabled.
- [ ] Unmodded 1.21.11 client joins normally.
- [ ] Fabric client log shows the successful handshake.
- [ ] Place a Poplar plank; run `/poplarcompat resend` if needed; the tracked position renders with the Poplar plank appearance.
- [ ] Break the block and confirm the client returns to the ViaBackwards block model.
- [ ] Confirm nearby oak/birch/acacia blocks are unchanged.
- [ ] Reconnect, change dimension, teleport, unload/reload a chunk, and use F3+T; verify no stale tracked positions remain.
- [ ] Repeat with Iris/IterationT enabled and watch for terrain/shader regressions.

## Build notes

Gradle 9.7.1 and Fabric Loom 1.18.2 are used because the 1.21.11 client toolchain and current Java runtime require them. The Paper module targets Java 25 to match Paper 26.3; the common and Fabric modules emit Java 21-compatible bytecode where applicable.
