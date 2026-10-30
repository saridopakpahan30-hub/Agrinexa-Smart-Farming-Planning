<p align="center">
  <img src="assets/images/logo.png" alt="Agrinexa Logo" width="120">
</p>

<h1 align="center">Agrinexa</h1>
<p align="center">
  <strong>Smart farm planning and profit simulation for Indonesian farmers</strong>
</p>

<p align="center">
  <a href="#features">Features</a> •
  <a href="#technology">Technology</a> •
  <a href="#running-it">Running It</a> •
  <a href="#architecture">Architecture</a> •
  <a href="#license">License</a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Flutter-3.x-blue?logo=flutter" alt="Flutter">
  <img src="https://img.shields.io/badge/Tests-158-green" alt="Tests">
  <img src="https://img.shields.io/badge/Commodities-69-orange" alt="Commodities">
  <img src="https://img.shields.io/badge/License-MIT-yellow" alt="License">
</p>

---

## About

Agrinexa is a mobile and web application that helps Indonesian farmers plan their
planting season optimally. By combining real-time market prices from official
government sources, a linear-programming optimization algorithm, and a complete
agronomy guide, Agrinexa answers one question:

> *"Given my land and my capital, which crops should I plant, and how much area
> should each one get, to maximize profit?"*

Live web app: https://agrinexa-5082d.web.app

---

## Features

### Planning & optimization

| Feature | Description |
|---------|-------------|
| **Multi-crop simulation** | Linear-programming (simplex) optimization that divides the land area automatically |
| **69 commodities** | Complete data: price, cost, yield, season, land suitability |
| **Visual land map** | The `PlotMap` widget shows the colour split per crop |
| **Per-cycle & annual margin** | Profit projection for one season plus a yearly estimate |
| **Diversification limit** | Slider for the maximum share per crop (35% by default); it widens automatically when few crops are selected |
| **Floor constraint** | Every selected crop receives a meaningful minimum allocation |

### Real-time market prices

| Feature | Description |
|---------|-------------|
| **Automatic update once per 24 hours** | Supabase cron job (`pg_cron` + `pg_net`) fetches from two official sources |
| **Kemendag SP2KP** | National average prices, daily, per province |
| **Bank Indonesia PIHPS** | Food prices per province from directly monitored markets |
| **Supabase Realtime** | Price changes appear in the app without a refresh |
| **Manual entry** | A price recorded by the user is never overwritten by an automatic source |
| **Honest daily chart** | One data point per date, with an accurate daily percentage change |
| **Fallback simulation** | Commodities without an official source: ±5% of the base price, deterministic per date |

### Weather & location

- **Open-Meteo API** — real-time weather, no API key required, CORS-friendly
- **GPS + Nominatim** — automatic location detection, address resolved from OpenStreetMap
- **Planting advice** — recommendations based on current weather conditions

### Farming tools

- **Fertilizer calculator** — total kg, per plant, schedule by growth stage, application method, and the consequences of over- or under-application
- **Pest & disease centre** — 44+ entries with visual symptoms and treatment
- **Cash book** — income and expenses of the farm
- **Planting calendar** — seasonal schedule per commodity
- **Field mode** — voice-guided daily work list (`flutter_tts`, on-device and offline)

### Account & synchronization

- **Three sign-in methods:** email/password, Google OAuth, magic link
- **Firestore sync** — data synchronized across devices with last-write-wins conflict resolution
- **Offline first** — local data remains available without a connection and syncs when back online
- **Two languages:** Indonesian (default) and English

---

## Technology

```
┌─────────────────────────────────────────────────────┐
│  Frontend: Flutter 3.x (Android + Web)              │
│  State:    AppState (InheritedWidget pattern)       │
│  Testing:  158 unit & widget tests                  │
├─────────────────────────────────────────────────────┤
│  Auth:     Firebase Authentication                  │
│  Hosting:  Firebase Hosting (agrinexa-5082d.web.app)│
│  Sync:     Firestore (LWW, offline-first)           │
├─────────────────────────────────────────────────────┤
│  Prices:   Supabase Postgres + pg_cron + pg_net     │
│  Realtime: Supabase Realtime (WebSocket)            │
│  Weather:  Open-Meteo API (keyless)                 │
│  Maps:     Nominatim / OpenStreetMap                │
│  Voice:    flutter_tts (on-device, offline)         │
└─────────────────────────────────────────────────────┘
```

---

## Architecture

```
lib/
├── core/       # Theme, i18n (ID/EN), rupiah formatting, constants
├── models/     # Commodity, CropGuide, PlanResult, Reco, Saran
├── data/
│   └── farm_repository.dart        # Cross-device data abstraction
├── services/
│   ├── market_service.dart         # Prices: manual > live > simulated
│   ├── recommendation.dart         # Engine.optimize (LP simplex)
│   └── lp.dart                     # Pure-Dart linear-programming solver
├── screens/    # 21 screen files: dashboard, simulation, market, profile...
├── widgets/    # PlotMap, PhotoBanner, StaggerIn, price chart...
└── state/      # AppState — single source of truth

supabase/
├── harga_69.sql            # Initial price data for 69 commodities
├── kunci_harga.sql         # RLS: the price table is read-only for the app
├── harga_riwayat.sql       # Trigger: archive the OLD value of every change
├── harga_multi_sumber.sql  # Cron job: fetch + ingest prices once per 24 hours
└── harga_realtime.sql      # Enable Supabase Realtime for the price table

firestore/
└── firestore.rules         # Ownership enforced on the server
```

---

## Running It

### Prerequisites

- [Flutter SDK](https://flutter.dev/docs/get-started/install) 3.x
- [Android Studio](https://developer.android.com/studio) (for building the APK)
- A [Firebase](https://firebase.google.com) project (auth + hosting)
- A [Supabase](https://supabase.com) project (real-time prices)

### Development

```bash
git clone https://github.com/<username>/agrinexa.git
cd agrinexa
flutter pub get
flutter run
```

### Release APK

```bash
flutter build apk --release --dart-define=BUILD=22
# Output: build/app/outputs/flutter-apk/app-release.apk (~74 MB)
```

The build number is shown on the splash screen, which makes it possible to tell
installed APK versions apart from a screenshot alone.

### Web deployment

```bash
firebase login
firebase init hosting   # choose build/web as the public directory
firebase deploy
```

### Supabase setup (automatic prices)

Run these files in order in the **Supabase SQL Editor** (one complete script at a
time, in this sequence):

```
1. supabase/harga_69.sql            → fill the table with initial prices
2. supabase/kunci_harga.sql         → lock the table to read-only
3. supabase/harga_riwayat.sql       → trigger that archives price changes
4. supabase/harga_multi_sumber.sql  → cron job that fetches prices once per 24 hours
5. supabase/harga_realtime.sql      → enable the Realtime WebSocket
```

After step 4 the job runs on its own (`pg_cron` uses UTC):

```
08:30 WIB  harga_kirim()           → HTTP request to Kemendag + Bank Indonesia
08:40 WIB  harga_serap()           → parse the JSON, store it in the price table
08:45 WIB  harga_simulasi_harian() → fill commodities without an official source
09:45 WIB  harga_bersihkan()       → delete HTTP responses older than 2 days
```

### Firebase setup

1. Create a project in the [Firebase Console](https://console.firebase.google.com)
2. **Authentication:** enable Email/Password, Google, and Email Link
3. **Android:** register the package name + SHA-1, download `google-services.json` → `android/app/`
4. **Web:** add a web app, then copy `firebaseConfig` into `lib/services/firebase_gate.dart`
5. **Firestore:** create the database and paste the security rules from `firestore/firestore.rules`
6. **Hosting:** `firebase init hosting` → `firebase deploy`

---

## Testing

```bash
flutter test                              # 158 tests, all must pass
flutter test test/plot_lp_test.dart       # LP solver tests
flutter test test/plan_sim_test.dart      # simulation screen tests
flutter test test/price_source_test.dart  # price source tests
```

---

## Contributing

Bug reports, feature requests, and pull requests are welcome. Please open an
[Issue](https://github.com/<username>/agrinexa/issues).

---

## License

- **Application code:** MIT. Add a `LICENSE` file to the repository root to make
  this notice binding.
- **Crop photographs:** [Wikimedia Commons](https://commons.wikimedia.org) —
  CC BY-SA / CC BY. Full attribution is available on the **Image Sources**
  screen inside the app.

---

<p align="center">
  Made with ❤️ for Indonesian farmers.
</p>
