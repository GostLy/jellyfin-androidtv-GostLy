# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Requires JDK 21 (see `.tools-versions`) and the Android SDK. Always use the Gradle wrapper.

```shell
./gradlew assembleDebug              # APK -> app/build/outputs/apk/debug (app id org.jellyfin.androidtv.debug)
./gradlew test                       # unit tests (all modules, all variants)
./gradlew :app:testDebugUnitTest     # faster: debug variant only
./gradlew detekt lint                # both CI static-analysis tasks
```

Run a single test class or test:

```shell
./gradlew :app:testDebugUnitTest --tests "org.jellyfin.androidtv.util.TimeUtilsTests"
./gradlew :app:testDebugUnitTest --tests "*TimeUtils*"
./gradlew :preference:test --tests "*PreferenceStoreTests*"
```

Tests use Kotest (`FunSpec` style, JUnit 5 platform) with MockK. `detekt` and `lint` have
`ignoreFailures`/`abortOnError = false`, so they never fail the build — read their output.

To build against an unreleased Jellyfin SDK, set `sdk.version` in `gradle.properties` to
`local`, `snapshot`, or `unstable-snapshot`.

## Conventions

- **Tabs** for indentation in Kotlin/Kotlin-script, max line length 140, no wildcard imports (`.editorconfig`).
- Logging goes through **Timber** — `android.util.Log` is a lint error.
- **Never edit `app/src/main/res/values-*/`** translation files. They are managed by Weblate and
  translation PRs are not accepted. Only `values/strings.xml` (English source) is edited by hand.
- `master` is the unstable development branch and the PR target; `release-x.y.z` branches are for releases.

## Architecture

Gradle multi-module project (`settings.gradle.kts`) using typesafe project accessors (`projects.playback.core`):

| Module | Purpose |
| --- | --- |
| `:app` | The Android TV application (the only application module) |
| `:design` | Pure design tokens (`org.jellyfin.design.Tokens`: color/radius/space/typography). No UI. |
| `:preference` | Generic typed preference-store abstraction (`Preference<T>`, `SharedPreferenceStore`, `AsyncPreferenceStore`, migrations) |
| `:playback:core` | Player-agnostic playback engine (`PlaybackManager`, services, queue, backends) |
| `:playback:jellyfin` | Jellyfin plugin for the engine: media stream resolution, progress reporting, media segments |
| `:playback:media3:exoplayer` | ExoPlayer `PlayerBackend` implementation |
| `:playback:media3:session` | MediaSession / notification integration |

### Startup and dependency injection

There is almost no logic in `JellyfinApplication` (it only initializes ACRA telemetry). Startup runs
through **androidx.startup `Initializer`s** chained by `dependencies()`:
`LogInitializer` → [KoinInitializer.kt](app/src/main/java/org/jellyfin/androidtv/di/KoinInitializer.kt) → [SessionInitializer.kt](app/src/main/java/org/jellyfin/androidtv/SessionInitializer.kt).

**Koin** is the DI container. All modules live in [app/src/main/java/org/jellyfin/androidtv/di/](app/src/main/java/org/jellyfin/androidtv/di/)
(`androidModule`, `appModule`, `authModule`, `playbackModule`, `preferenceModule`, `utilsModule`) and must be
registered in `KoinInitializer`. New repositories/view models are wired there rather than constructed directly.

### Server API and session

A **single `ApiClient` singleton** is created empty in `AppModule` and mutated at runtime —
[SessionRepository](app/src/main/java/org/jellyfin/androidtv/auth/repository/SessionRepository.kt) sets its
base URL, access token and device info when a session is restored or switched. Consequently, injecting
`ApiClient` anywhere gives the currently-signed-in user's client; there is no per-request client construction.
Server/user credential storage lives in `auth/store/`, and `auth/repository/` holds the
server discovery, authentication and session flows. Real-time server events arrive via `SocketHandler`.

### Preferences

`:preference` provides the mechanism; `app/preference/` declares the actual stores:
`UserPreferences` and `SystemPreferences` (device-local `SharedPreferences`),
`UserSettingPreferences`, `LibraryPreferences`, `LiveTvPreferences` (server-synced *display preferences*, see
`preference/store/DisplayPreferencesStore.kt`), and `AuthenticationPreferences`. Preferences are declared as
`companion object` vals (`enumPreference("key", Default)`) and read with `preferences[UserPreferences.someKey]`.
When changing a preference's key or type, add a migration in the store's `init` block. `PreferencesRepository`
reloads the server-backed stores on session change.

### UI: three overlapping generations

This is the single most important thing to understand before editing UI code. The app is mid-migration
and all three styles are live:

1. **Leanback + Java fragments (legacy).** ~36 `.java` files remain (`ui/browsing/BrowseGridFragment.java`,
   `ui/itemdetail/FullDetailsFragment.java`, `ui/livetv/LiveTvGuideFragment.java`, `ui/itemhandling/ItemRowAdapter.java`).
   These use Leanback `Presenter`s from `ui/presentation/` and `BaseRowItem` subclasses from `ui/itemhandling/`.
2. **Fragment navigation via `NavigationRepository`.** [NavigationRepository.kt](app/src/main/java/org/jellyfin/androidtv/ui/navigation/NavigationRepository.kt)
   holds a `Stack<Destination.Fragment>` and emits `NavigationAction`s consumed by `AppNavigationHost` inside
   `MainActivity`'s `setContent`. Destinations are declared in
   [Destinations.kt](app/src/main/java/org/jellyfin/androidtv/ui/navigation/Destinations.kt) as `fragmentDestination<T> { bundle }`.
3. **Compose + androidx navigation3 `Router`.** [router.kt](app/src/main/java/org/jellyfin/androidtv/ui/navigation/router.kt)
   is a string-route/`RouteContext` router used by the new settings UI (`ui/settings/routes.kt`, `ui/settings/screen/`)
   and the new players (`ui/player/`). New screens should be written this way.

`MainActivity` is a Compose host that composes `AppBackground`, `AppNavigationHost` (legacy fragments),
`InAppScreensaver` and `MainActivitySettings` (the nav3 settings overlay) into one screen.

**There is no Compose Material dependency.** The app has its own design system in
[ui/base/](app/src/main/java/org/jellyfin/androidtv/ui/base/): `JellyfinTheme` provides `colorScheme`/`typography`/`shapes`
through composition locals, with `Text`, `Icon`, `button/`, `form/`, `list/`, `dialog/`, `popover/` primitives built on
`androidx.compose.foundation`. Use these, never `androidx.compose.material*`.

### Playback: two engines

- **Legacy** (`ui/playback/`): `PlaybackController.java` + `VideoManager.java` + `CustomPlaybackOverlayFragment.java`,
  still used for audio and Live TV paths.
- **Rewrite** (`:playback:*` modules): `PlaybackManager` is a service container — plugins (`exoPlayerPlugin`,
  `media3SessionPlugin`, `jellyfinPlugin`) contribute a `PlayerBackend`, `PlayerService`s and `MediaStreamResolver`s
  at build time in `Scope.createPlaybackManager()` in [PlaybackModule.kt](app/src/main/java/org/jellyfin/androidtv/di/PlaybackModule.kt).
  State is exposed through `PlayerState` flows; the queue is a `QueueService` with `QueueSupplier`s.
  `RewriteMediaManager` adapts the new engine to the legacy `MediaManager` interface, so legacy call sites
  transparently drive the new player. New player UI is in `ui/player/`.

Device capability negotiation for both engines goes through `util/profile/createDeviceProfile`, which is
built from `UserPreferences` codec settings and `MediaCodec` capability probing (`util/profile/codec/`).

### Other integrations

`integration/` contains the Android TV home-screen channels (`LeanbackChannelWorker`, a WorkManager job),
the search content provider (`MediaContentProvider`), and the daydream/screensaver (`integration/dream/`).
Crash reporting is ACRA, gated by `TelemetryPreferences`.
