# Renderium

A client-side **NeoForge** mod for **Minecraft 26.3** that makes your effective render distance larger than what the server streams to you.

Normally, a server only sends chunks within its own view distance. Once a chunk leaves that radius, the server tells your client to drop it and it eventually turns into void.

**Renderium** catches those chunks before they are dropped and keeps rendering them, allowing areas you have already visited to remain visible far beyond the server's view distance.

The concept is similar to the well-known **Bobby** mod.

> ⚠️ **Experimental:** Renderium has not yet been fully tested in-game. It compiles against the real Minecraft 26.3 Mojmap API and all mixins target existing methods, but Minecraft 26.3 contains significant renderer changes. The render trigger is therefore the most likely area to require tuning.

---

## ✨ Features

### 🧠 In-memory cache

Chunks that the server drops are kept in memory and continue rendering for the rest of the session.

This means:

- Previously visited terrain stays visible.
- You can walk or fly beyond the server's view distance.
- Areas behind you don't immediately turn into void.
- An LRU cache prevents unlimited memory usage.

The maximum number of cached chunks can be configured with `maxCachedChunks`.

---

### 💾 Persistent disk cache

Renderium can also save cached chunks to disk.

Cached chunks are stored as gzipped NBT files under:

```text
.minecraft/renderium/<server>/<dimension>/
```

The cache survives:

- Disconnecting from a server
- Restarting Minecraft
- Reconnecting later

When Minecraft requests a chunk that is no longer available in memory, Renderium can load it from the disk cache in the background.

Once loaded, the chunk is restored to the client and becomes visible again.

---

### 💡 Lighting support

Renderium stores lighting information alongside the terrain.

Cached chunks include:

- Block light
- Sky light

This means chunks loaded from disk are properly lit instead of appearing completely dark.

---

### 📦 Disk cache limit

The disk cache has a configurable size limit.

When the configured limit is reached, older cached chunk files are automatically removed.

This prevents the cache from growing indefinitely.

---

## ⚠️ Current limitations

Renderium currently does **not** persist block entities.

This means the following are not currently restored from the disk cache:

- Chest contents
- Sign text
- Banners
- Other block entities

Terrain and lighting **are** persisted.

Block entity persistence is planned for a future update.

---

## 📋 Requirements

| Requirement | Version |
|---|---|
| Minecraft | `26.3` |
| NeoForge | `26.3.0.115+` |
| Java | `25` |
| Gradle | `9.2+` |

Minecraft 26.3 ships with Java 25, so development and runtime environments should use JDK 25.

---

## 🎮 Usage

Renderium is a **client-side mod**.

The server does not need to have Renderium installed.

After installing Renderium, set your Minecraft **render distance higher than the server's view distance**.

For example:

```text
Server view distance: 12
Client render distance: 32
```

As you explore the world, Renderium caches chunks that the server later unloads.

Those chunks can continue to be rendered outside the server's active view distance.

> Renderium cannot show terrain that the client has never received from the server. It only caches and renders chunks that have previously been loaded.

---

## ⚙️ Configuration

Renderium uses:

```text
config/renderium.json
```

Default configuration:

```json
{
  "enabled": true,
  "maxCachedChunks": 8192,
  "diskCacheEnabled": true,
  "maxDiskCacheMB": 2048
}
```

### Configuration options

| Key | Default | Description |
|---|---:|---|
| `enabled` | `true` | Master switch. `false` disables Renderium and restores vanilla behaviour. |
| `maxCachedChunks` | `8192` | Maximum number of chunks kept in memory. |
| `diskCacheEnabled` | `true` | Enables persistent disk caching. |
| `maxDiskCacheMB` | `2048` | Maximum size of the persistent disk cache in MB. |

---

## 🔧 How it works

Renderium hooks into the client-side chunk lifecycle.

### 1. Intercepting chunk unloads

`ClientChunkCacheMixin` intercepts chunks immediately before the server-driven:

```text
drop(ChunkPos)
```

Instead of allowing the chunk to disappear completely, Renderium stores it in its cache.

---

### 2. Cached chunk lookup

When Minecraft requests a chunk that is no longer available in the normal client chunk cache, Renderium checks its own cache.

If a cached version exists, it can be supplied instead of returning an empty chunk.

---

### 3. In-memory LRU cache

`FakeChunkManager` manages the in-memory cache.

The cache is limited by:

```text
maxCachedChunks
```

When the limit is reached, older chunks are removed according to the LRU policy.

---

### 4. Persistent storage

When disk caching is enabled, cached chunks are written to disk as compressed NBT files.

The cache is separated by server and dimension:

```text
renderium/
├── <server>/
│   ├── minecraft/
│   ├── minecraft_the_nether/
│   └── minecraft_the_end/
│
└── <another-server>/
    └── minecraft/
```

This prevents chunks from different servers or dimensions from being mixed together.

---

### 5. Asynchronous disk loading

Disk operations are performed in the background.

When a chunk needs to be restored from disk:

1. Renderium queues the disk load.
2. The NBT file is read in the background.
3. The chunk is reconstructed.
4. The chunk is queued for insertion on the Minecraft client thread.
5. The renderer is notified that the relevant sections need to be processed again.

This prevents disk operations from unnecessarily blocking the client thread.

---

### 6. Renderer refresh

After restoring a cached chunk, Renderium nudges the renderer to re-process the affected sections.

The current implementation uses the renderer's section invalidation system.

One particularly important area is:

```text
SectionOcclusionGraph#invalidate
```

Because Minecraft 26.3 contains significant renderer changes, this is currently one of the most likely areas to need adjustments during testing.

---

### 7. Login and dimension changes

The in-memory cache is cleared when:

- Connecting to a server
- Changing dimensions

The disk cache is automatically redirected to the appropriate server and dimension.

This prevents cached terrain from one world accidentally appearing in another.

---

## 💡 Example

Imagine a server configured with:

```text
Server view distance: 10
Client render distance: 32
```

Normally, when you move far enough away from an area, the server tells your client to unload those chunks.

Without Renderium:

```text
        SERVER VIEW DISTANCE

        ████████████
        ████████████
        ███ PLAYER ███
        ████████████
        ████████████

                ↓

              VOID
```

With Renderium:

```text
      CACHED TERRAIN
████████████████████████████
████████████████████████████
██████████ PLAYER ██████████
████████████████████████████
████████████████████████████
      CACHED TERRAIN
```

The terrain that you have already visited can remain visible even after the server has stopped actively sending those chunks.

---

## 🧪 Development status

Renderium is currently **experimental**.

### Implemented

- [x] Client-side chunk interception
- [x] In-memory chunk caching
- [x] LRU memory cache
- [x] Persistent disk caching
- [x] Gzipped NBT storage
- [x] Block state persistence
- [x] Biome persistence
- [x] Block light persistence
- [x] Sky light persistence
- [x] Asynchronous disk loading
- [x] Disk cache size limit
- [x] Server-specific cache directories
- [x] Dimension-specific cache directories
- [x] Cache reset on login
- [x] Cache reset on dimension change

### Planned

- [ ] Block entity persistence
- [ ] More extensive in-game testing
- [ ] Renderer compatibility improvements
- [ ] Performance improvements
- [ ] Cache statistics
- [ ] Cache management/debug screen
- [ ] Additional configuration options

---

## 🐛 Known risks

Minecraft 26.3 introduced substantial changes to the rendering system.

The most likely area to require adjustments during testing is the renderer refresh mechanism.

In particular:

```text
SectionOcclusionGraph#invalidate
```

If chunks are successfully restored from the cache but do not immediately become visible, the renderer invalidation and section rebuild process should be one of the first areas investigated.

---

## 📁 Project structure

A simplified project structure looks like this:

```text
Renderium/
├── src/
│   └── main/
│       ├── java/
│       │   └── ...
│       └── resources/
│           └── ...
│
├── config/
├── dist/
├── build.gradle
├── gradle.properties
└── README.md
```

### Important components

| Component | Purpose |
|---|---|
| `ClientChunkCacheMixin` | Intercepts chunk unloading and cached chunk lookups |
| `FakeChunkManager` | Manages the in-memory cache and asynchronous disk loading |
| `ChunkSerializer` | Converts chunks to and from NBT |
| `ChunkStorage` | Handles persistent chunk storage and disk limits |
| `ClientPacketListenerMixin` | Handles login and dimension changes |
| Main mod class | Handles configuration and cache initialization |

---

## 🏗️ Building

Renderium requires:

- JDK 25
- Gradle 9.2 or newer

Build the project with:

```bash
gradle build
```

The compiled JAR will be placed in:

```text
build/libs/
```

Prebuilt JARs can also be found in:

```text
dist/
```

### Gradle wrapper

The repository currently does not include the Gradle wrapper.

You can generate one with:

```bash
gradle wrapper
```

After generating it, Linux/macOS users can build with:

```bash
./gradlew build
```

Windows users can use:

```powershell
.\gradlew.bat build
```

---

## 🔒 Compatibility

Renderium is currently designed specifically for:

```text
Minecraft 26.3
NeoForge 26.3.0.115+
Java 25
```

Minecraft internals change frequently, especially around rendering and chunk management.

Other Minecraft versions are **not guaranteed to work**.

---

## ⚠️ Important

Renderium does **not** increase the server's actual view distance.

It does not allow the client to request or reveal chunks that the server has never sent.

The basic concept is:

```text
Server
   │
   │ sends chunk
   ▼
Client
   │
   ├── normal chunk cache
   │
   └── Renderium cache
          │
          ├── RAM
          │
          └── Disk
```

Renderium can only preserve terrain that the client has already received.

Cached terrain can also become outdated if the world changes after the chunk was cached.

---

## ❤️ Inspiration

Renderium is inspired by the concept behind **Bobby**, which demonstrates how client-side chunk caching can be used to extend the effective render distance.

Renderium is an independent implementation designed for:

- NeoForge
- Minecraft 26.3
- Java 25

---

## 📜 License

**All Rights Reserved.**

Copyright © 2026 Renderium contributors.

The source code and associated assets may not be copied, modified, redistributed, republished, or used commercially without explicit permission from the copyright holder.

This project is provided as-is, without warranty of any kind.
