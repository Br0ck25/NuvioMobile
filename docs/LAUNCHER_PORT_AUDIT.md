# Nuvio Mobile Launcher: porting audit

Source: `Br0ck25/NuvioTVLauncher` (Android TV fork, `com.nuvio.tv.launcher`).
Target: this repo, a fork of NuvioMobile (Kotlin/Compose Multiplatform, branch `cmp-rewrite`).

This is a source audit only. Nothing here has been built or run: no JDK, Android SDK or device was available in the session that wrote it.

## 1. What the two codebases are

| | TV fork | Mobile (this repo) |
|---|---|---|
| UI | Compose for TV (`androidx.tv.material3`), D-pad focus | Compose Multiplatform, touch-first |
| Language scope | Android only | `commonMain` shared with iOS/desktop; `androidMain`/`iosMain` via expect/actual |
| DI | Hilt | none (singleton objects/repositories) |
| HTTP | OkHttp | Ktor |
| JSON | Gson / org.json | kotlinx.serialization |
| Storage | AndroidX DataStore | per-feature `*Storage` expect/actual |
| Navigation | Navigation Compose | Navigation3, 4 tabs (`AppScreenTab`: Home, Search, Library, Settings) |
| Player | Media3 / libmpv, 4.5k-line `PlayerScreen` | `PlayerEngine` abstraction, Media3 + MPV, touch gestures (`PlayerSurfaceGestures`, `PlayerGestureOverlay`), PiP |

Consequence: **this is a rewrite against mobile's architecture, not a copy.** TV code can be used as a spec and for its pure logic (parsers, schedulers), but nothing compiles unchanged.

## 2. What mobile already has (no port needed)

Addons, catalogs, collections and folders, library lists, Trakt, Simkl, MDBList, debrid (Torbox/Real-Debrid/Premiumize) and P2P streaming, downloads, watch party, plugins, profiles, TMDB, episode-release notifications, subtitle auto-sync, custom poster screen (`core/poster/CustomPosterScreen.kt`), touch gestures and Picture-in-Picture in the player.

Touch controls are therefore not part of this port. They exist already.

## 3. Features to port

| Feature (TV fork) | Size on TV | In mobile? | Portability |
|---|---|---|---|
| **Live TV**: M3U / Xtream / Stalker sources, EPG (XMLTV and server schedules), catch-up, favorites, playlists, categories | ~20k lines incl. UI, `reshaped/livetv/*` | No | Logic (`LiveTvPlaylistParser`, `LiveTvEpg`, `LiveTvProviders`, `LiveTvCatchupLinks`, `ServerSchedule`, `LiveTvGuideCache`) is mostly pure and portable to `commonMain` once OkHttp/Gson are replaced. UI must be redesigned for touch. |
| **List channels**: turn a catalog/collection/library list into a scheduled channel, played via debrid or P2P | `lists/ListChannels.kt` 1.6k lines | No | Depends on mobile's debrid/P2P resolvers (`DirectDebridResolver`, `P2pStreaming`) which exist. Scheduler is portable. |
| **My Lists addon integration**: default charts and collections seed, account link (QR/pairing), per-profile login, watch-history reporting at 90%, Better Posters flag, `If-None-Match` sync | `core/mylists/*` ~1.5k lines + dialog | No | It is a Stremio addon, so catalogs already work. Port the seed, link flow, sync manager and watch reporting. QR pairing is not needed on a phone (use a deep link or paste). |
| Calendar (TVmaze + TMDB, per profile) | `calendar/*` 4.2k lines | Partial (upcoming items in Continue Watching / Home) | Decide whether a standalone Calendar screen is wanted. |
| Better Posters / custom poster resolver | `core/poster/CustomPosterUrlResolver` | Partial | Merge with mobile's existing custom poster support. |
| Launcher: HOME role, accessibility service, Apps screen, icon packs, Button Mapper, ADB setup, TV channels, idle profile switch | large | n/a | **Do not port.** No equivalent on a phone. |
| Android TV remote UX, `LongPressKeyTracker`, ambient screensaver | | n/a | Do not port. |

## 4. Installing next to the official app

Done in the first commit on this branch.

| Item | Official | This fork |
|---|---|---|
| `applicationId` (release, both flavors) | `com.nuvio.app` | `com.nuvio.mobile.launcher` |
| `applicationId` (debug) | `com.nuviodebug.com` | `com.nuvio.mobile.launcher.debug` |
| Launcher label | Nuvio | Nuvio Launcher |
| Trakt / Simkl OAuth redirect | `nuvio://auth/...` | `nuviolauncher://auth/...` (default in `composeApp/build.gradle.kts`, manifest filters updated) |
| `https://nuvio.tv/party/` App Link | autoVerify filter | removed (it cannot verify for another package) |
| FileProvider authority | `${applicationId}.fileprovider` | follows `applicationId`, no change needed |

Code namespaces (`com.nuvio.app`, `com.nuvio.android`) are unchanged on purpose. They are not the installed package name, and keeping them avoids rewriting ~1,400 files and makes merging upstream cheap.

Still to do or decide:
- **Trakt/Simkl**: the new redirect URI must be registered on the client IDs you use. Without that, sign-in fails. Create your own Trakt and Simkl apps, or keep the official ones only if their redirect list allows this scheme.
- **`nuvio://` and `stremio://` filters** remain, so tapping those links on a device with both apps installed shows a chooser. Acceptable; rename if it is a nuisance.
- **Account sign-in / sync backend** (`api.nuvio.tv`, Supabase config) belongs to the original project. As with the TV fork, expect "continue without account" unless you run your own backend.
- **Sentry DSN, Play Store/AltStore metadata, `store.json`, CI** still point to upstream. Review before publishing.
- **iOS** bundle ID (`com.nuvio.app.Nuvio`) not changed; iOS cannot be built or checked from this environment.

## 5. Proposed order of work

1. **Done:** package identity (above).
2. **Live TV core** in `commonMain/.../features/livetv/`: models, parsers, providers, EPG, storage (expect/actual), repository, with unit tests ported from the TV fork's tests.
3. **Live TV UI**: touch guide (channel list, programme grid, categories, favorites), player integration via `PlayerEngine`, settings page for sources.
4. **List channels** on top of the debrid/P2P resolvers.
5. **My Lists**: seed, link flow, sync, watch reporting, Better Posters.
6. **Calendar** (only if wanted).

Keep all fork code in new `features/livetv` and `features/mylists` directories, with small hooks into existing files (nav, settings, home), so that merging upstream stays low-conflict.

## 6. Risks

- **Size:** each of steps 2–5 is a substantial piece of work. Expect several sessions.
- **No build here:** every step needs compiling on a machine with the Android SDK; until then nothing is verified.
- **Navigation:** `AppScreenTab` also feeds iOS's native tab bar. Adding a Live TV tab touches that. The Android-first option is to expose Live TV from Home/Library/Settings.
- **ExoPlayer vs live streams:** mobile's `PlayerEngine` is built around VOD (watch progress, chapters, subtitles). Live playback needs its own controls (no seek bar, channel zapping, reconnect on stall).
- **Legal:** the TV fork supplies no channels or content; keep that stance. The original project's license is GPL-3.0; this fork inherits it.
