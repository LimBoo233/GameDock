# GameDock

**An Android app that puts PC game giveaways, cross-store price comparison, bundles and price-drop alerts in one place.**

Coursework project for **COMP3011J Mobile Computing**
Beijing-Dublin International College (Beijing University of Technology × University College Dublin)
November 2025 – January 2026 · Kotlin · Jetpack Compose · Hilt

---

## Motivation

Game deals are scattered by design. Free giveaways live on each store's own page, price history lives on a website built for desktop, and bundles show up somewhere else again. PC-side tools dominate, and there is no mobile app dedicated to spending less on games.

GameDock consolidates PC game information from several stores (Steam, Epic Games Store, GOG, Ubisoft Connect and the rest of the shops reachable through IsThereAnyDeal) into a single Android app:

- track free giveaways across stores and claim them
- compare the current price of one title across stores, against its historical low
- keep a watchlist and get notified when a tracked game drops in price
- link real Steam / Epic accounts and open authenticated store pages inside the app

## Features

### Freebies — cross-store free game feed, claimed in-app
- Merges two independent sources: Epic's own free-promotion endpoint and the GamerPower giveaway feed (non-Epic entries filtered out).
- Splits results into **active** and **upcoming**; start and end times are parsed from ISO strings and shown as countdowns.
- Results are cached locally, and the cached list is served instead of an error when the network fails.
- **Claim** opens the redemption page in an in-app `WebView` with the right session already injected. No linked account → the user is prompted; several accounts → the app asks which one; unrecognised platform → the page is handed to the external browser.

### Compare — cross-store price comparison
- Game search and pricing through the **IsThereAnyDeal API** (`games/search`, `games/overview`, `games/prices`, `games/bundles`).
- Each offer card shows the store, the current price, the **historical low** and artwork; the best offer is highlighted together with the gap to the runner-up.
- Region-aware pricing: the country code is resolved from the device locale and validated against the supported set.
- Shop filtering, optional voucher-inclusive prices, sorting, refresh, loading skeletons and empty/error states.

### Bundles
- Bundle search plus a live bundle feed from IsThereAnyDeal, showing the package price, the games included and the time remaining.

### Watchlist
- Persisted locally and exposed as a `StateFlow`; supports add and remove, a preferred-store filter and a per-item notification toggle.
- An entry can jump back into Compare to re-run its query.

### Background price alerts
- A `WorkManager` job (`PriceCheckWorker`) re-prices every watchlist entry on a schedule, applies the preferred-store filter and raises a notification on a price drop — or `FREE NOW` when a tracked title reaches zero.
- Honours both the global notification switch and the per-item switch; creates its own notification channel and handles the Android 13+ runtime permission.

### Store account linking
- **Steam**: a `WebView` login captures the `steamLoginSecure` / `sessionid` cookies, resolves the SteamID from them and fetches the profile name and avatar through the Steam Web API.
- **Epic**: the OAuth authorization code is intercepted and exchanged for access/refresh tokens; tokens can be refreshed from the account detail screen.
- Credentials are kept in `EncryptedSharedPreferences` and injected into `WebView`s as cookies or `Authorization` headers, so claiming a game or opening a profile uses the genuinely logged-in session rather than a mock.

## Data sources

| Service | Used for |
|---|---|
| [IsThereAnyDeal](https://isthereanydeal.com/) | game search, multi-store prices, historical lows, bundles |
| [Epic Games Store](https://store.epicgames.com/) | current and upcoming weekly free games |
| [GamerPower](https://www.gamerpower.com/) | free-game giveaways across platforms |
| [Steam Web API](https://steamcommunity.com/dev) | profile name and avatar for linked Steam accounts |
| Epic OAuth | authorization-code / token exchange for linked Epic accounts |

## Architecture

A single-activity Compose app layered top-down — `ui → repository → remote / local` — with Hilt wiring the layers together.

```
app/src/main/java/com/example/gamedock/
├─ ui/                    Compose surfaces and navigation
│  ├─ home/               account screens, WebView login/claim activities, ViewModels
│  ├─ components/         GameCard, PriceCard, SectionHeader, AccountCard …
│  └─ theme/              Material 3 colour, typography, theme
├─ data/
│  ├─ model/              domain models — Game, Offer, Freebie, BundleDeal, PlatformType …
│  ├─ remote/             Retrofit services, response DTOs and per-source adapters
│  │  ├─ itad/            search · price overview · price details · bundles
│  │  ├─ epic/            free-promotion feed + store adapter
│  │  └─ gamerpower/      giveaway feed + store adapter
│  ├─ local/              encrypted credential stores and offline caches
│  ├─ repository/         deals · accounts · Epic auth · settings · credential provider
│  └─ watchlist/          watchlist entity and repository
├─ di/                    Hilt modules — NetworkModule, RepositoryModule
├─ workers/               PriceCheckWorker
└─ notifications/         PriceDropNotifier and notification channel
```

**Data flow.** Every external service gets its own Retrofit instance and an adapter that maps its response DTOs into a shared domain model (`Offer`, `Freebie`, `BundleDeal`). Repositories are the only thing the UI ever sees: `DealsRepositoryImpl` fans out to the Epic, GamerPower and IsThereAnyDeal adapters, merges the results or falls back to the cache, and hands plain domain objects to the Compose screens. Account credentials never leave the `local/` layer, except as cookies or headers passed to a `WebView`.

## Tech stack

| Area | Choice |
|---|---|
| Language | Kotlin |
| UI | Jetpack Compose, Material 3, Navigation Compose |
| Dependency injection | Hilt (Dagger) + Hilt Work |
| Networking | Retrofit 2 + Gson |
| Local state | EncryptedSharedPreferences (credentials) · SharedPreferences + Gson (watchlist, caches) |
| Background work | WorkManager |
| Image loading | Coil |
| Build | Gradle Kotlin DSL with a version catalogue · minSdk 24, compile/target SDK 36, Java 11 |

## Build and run

```bash
git clone https://github.com/LimBoo233/GameDock.git
cd GameDock
./gradlew assembleDebug
```

Needs Android Studio with the Compose tooling, JDK 11+ and SDK 36.

> The price comparison and bundle features call the IsThereAnyDeal API and require an API key of your own. The freebie feed and store account linking work without one.

## Team and contributions

Three-person team; the commit history in this repository is the record.

| Member | Primary area |
|---|---|
| [@yukunliu110100001111](https://github.com/yukunliu110100001111) | Application shell, Compose screens and theming, navigation, store login flows, settings |
| 李思予 | Epic and GamerPower freebie pipelines, watchlist repository, freebie time and filter logic |
| [@LimBoo233](https://github.com/LimBoo233) | Hilt dependency-injection migration; the IsThereAnyDeal price-comparison slice end to end — Retrofit service, response DTOs, adapter, repository, Compose UI; bundle search; multi-store price display; subsequent refactoring, bug fixes and UI polish |

## Documentation

The `document/` folder is the design record written during the course:

- `GameDock.md` — product definition and competitive positioning
- `alpha.md` / `beta.md` — staged feature specifications (PDF versions included)
- `api.md`, `account_api.md` — external API inventory and account-API notes
- `current_features.md` — how each feature is implemented in the current code
- `分工.md`, `mobile分工.md` — team responsibility split

## Known limitations

- `FakeDealsRepository` remains from the first milestone, when the UI ran on mock data. The wired-up implementation is `DealsRepositoryImpl`.
- Room is declared in the Gradle build but persistence currently goes through `SharedPreferences` + Gson.
- Prices cover only the shops IsThereAnyDeal exposes; in-app purchasing is deliberately out of scope.
