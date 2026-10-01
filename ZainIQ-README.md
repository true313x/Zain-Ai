# Zain IQ AI Agent

An agentic telecom + commerce + financial-assistance + cybersecurity Android
app for Zain Iraq users. This is an **unofficial, independent engineering
scaffold** — not affiliated with, endorsed by, or built using any non-public
material from Zain Iraq. No Zain-owned logos, wordmarks, or other
brand assets are used anywhere; the color palette is an original purple
scheme only "inspired by" the general telecom-brand family described in the
build spec. Every backend integration is a mock/interface pair until real
API credentials and specs exist — see **Mock vs. Production** below.

## Current status: Phases 1-4 done

Phase 1, Phase 2 (Account, AI Chat, Voice Assistant, Settings), Phase 3
(Internet/Telecom/Offers agents), and Phase 4 (Zain Cash — wallet, recharge,
bill payment, digital cards; Security Center — device score, URL scanner,
SMS scanner) are all done, per the build spec's own phase plan. The Security
Center pieces are worth noting specifically: unlike almost everything else
in this build, they're **not mocked** — device scanning and URL/SMS
analysis run on real Android APIs and real heuristic logic, because none of
it needs a backend Zain hasn't given us. See **What's real vs. placeholder**
for the exact line on every module.

This project was generated in a sandboxed environment with **no Android SDK
and no network access** — so while every file is hand-written, real Kotlin
and Gradle Kotlin DSL, **it has not been compiled or run**. Open it in a
current Android Studio, let Gradle sync, and fix whatever the first sync
surfaces (expect maybe a version-catalog patch bump here or there — see
**Toolchain versions**) before trusting it as a working build.

## What's real vs. placeholder

| Area | State |
|---|---|
| Gradle multi-module setup, version catalog | Real |
| core:common, core:model, core:design, core:ui | Real |
| core:network (config + OkHttp/Retrofit DI, no live endpoints) | Real |
| core:database (Room schema: 13 entities, 3 DAOs wired, rest incremental) | Real |
| core:security (Keystore-backed `SecureStorage`, `BiometricAuthManager`) | Real |
| core:logging, core:analytics (no-op), core:localization | Real |
| domain: entities, repository interfaces | Real |
| domain: `Agent`, `AgentOrchestrator`, `ToolRegistry`, `PolicyEngine`, `FraudRiskEngine`, `LLMProvider` | Real, and now proven end-to-end (see `InternetAgent` below) — `ToolRegistry`/`AgentOrchestrator` are populated via Hilt `Set<Agent>`/`Set<Tool>` multibindings, one `@Binds @IntoSet` pair per concrete Agent/Tool (`:data`'s `AgentBindingsModule`) |
| `LocalLLMProvider` | `isReady()` still honestly `false` — no trained on-device model. `recognizeIntent()` now runs a deterministic **keyword-table fallback** using the exact phrase→intent examples from build spec §53, so the app is functionally demoable before the real model exists. Confidence is capped low so a caller can tell a keyword hit from real model output later |
| `InternetAgent` + `diagnose_internet` Tool | **Real, end-to-end.** "النت ما يشتغل" → keyword match → `AgentOrchestrator` → `InternetAgent` → `diagnose_internet` (LOW risk) → real diagnostics card. build spec §55 Demo Scenario 1 |
| `TelecomAgent` + `get_balance`/`get_usage`/`get_package` Tools | **Real, end-to-end.** "شكد رصيدي" / "شكد باقيلي نت" / "شنو باقتي" → `TelecomAgent` → the matching LOW-risk Tool → real data via `ZainRepository`. build spec §55 Demo Scenario 2 |
| `OffersAgent` + `get_offers`/`subscribe_package` Tools | **Real.** Chat/voice "أريد باقة" surfaces real offers (`get_offers`, LOW). Actually subscribing (`subscribe_package`, MEDIUM — needs confirmation) is reached from the dedicated Offers screen today, through the *same* `PolicyEngine` check chat would use — see `OffersAgent`'s doc comment for why entity extraction (which offer?) is the current limit |
| `ZainCashAgent` + `get_wallet`/`get_transactions`/`recharge` Tools | **Real — first HIGH-risk tool wired end-to-end.** Chat/voice "زين كاش" shows the wallet (`get_wallet`, LOW). The Zain Cash *screen*'s recharge flow runs the full build spec §15 chain for real: `PolicyEngine` (HIGH ⇒ confirm + biometric) → real `BiometricPrompt` (via `MainActivity` now extending `FragmentActivity`) → `FraudRiskEngine.assess()` on the specific transaction → `recharge` Tool → `MockZainCashRepository`. A CRITICAL fraud assessment still blocks the transaction even after biometric success |
| `SecurityAgent` + `scan_url`/`analyze_sms`/`scan_device`/`get_security_score`/`analyze_app_risk` Tools | **Real, end-to-end — including from chat.** "هذا الرابط آمن؟ https://…" extracts the URL and scans it live; a pasted suspicious SMS gets analyzed directly; "فحص الجهاز" runs a real on-device scan. build spec §55 Scenarios 4 and 5 both work from chat, not just a dedicated screen — see `SecurityAgent`'s doc comment for why this one didn't need the screen-first pattern Offers/ZainCash use |
| data: `MockZainRepository` | Real, working, clearly-marked demo data |
| `SupportAgent` + `create_ticket`/`get_ticket`/`escalate_to_human` Tools, `MockSupportRepository`, `MockNotificationRepository` | **Real** (in-memory mocks — no ticketing backend exists). "أريد موظف" → escalation ticket with the user's own words; no history is invented for the handoff context |
| feature:home | Real |
| feature:account | **Real** — balance/package/usage from `ZainRepository`, loading/error states |
| feature:ai (AI Chat) | **Real** — message list, text input, wired through `ProcessUserMessageUseCase` → real `AgentResult`s, confirmation dialog UI for `RequiresConfirmation`. With only `InternetAgent` registered, most phrases still escalate — that's correct, not a bug |
| feature:voice | **Real** — full `SpeechRecognizer`/`TextToSpeech` wiring, runtime `RECORD_AUDIO` permission flow, the exact 6-state machine from build spec §28 (Listening/Thinking/Checking/Executing/Completed/Error) |
| feature:settings | **Real** — language (ar/en) and theme (light/dark/system) selectors, persisted via DataStore (`core:localization`'s `LocaleManager`/`ThemeManager`) |
| feature:internet | **Real** — the exact diagnostic card from build spec §31 (✓/⚠ rows, a working "Fix" action for the APN flag) |
| feature:offers | **Real** — offer list from `ZainRepository`, subscribe flow gated by `PolicyEngine` (confirmation dialog), success state |
| feature:zaincash | **Real** — wallet, transaction history, recharge with biometric confirmation, plus a Store section: **bill payment** (biller list → bill number → looked-up amount → confirm + biometric — electricity is the flagship biller) and **digital card purchases** (internet/telecom/games/entertainment categories, matching the real app's "بطاقات الكترونية" section) |
| feature:security | **Real** — device security score with named +/- factors and recommendations (matching build spec §21's exact format), a URL scanner, and an SMS/message scanner |
| feature:support, tickets, notifications | **Real** — create ticket / hand off to a human, ticket list with expandable timeline, notification list with read state (reachable from Home via 🔔) |
| feature:packages, sim | Still placeholder screens — real, compiling, reachable, phase noted per file. (`packages`/باقاتي has no Home card in the spec — its data already surfaces via Account and `get_package`, so a dedicated screen isn't scheduled) |
| Navigation graph | Real, complete — all 15 destinations wired |
| Applying the Settings language/theme choice app-wide (MainActivity reading `ThemeManager`/`LocaleManager` on launch), Demo Mode's other 8 scenarios, Tests, on-device model, remaining agents/tools | Not started |

## Architecture

```
                        ZAIN IQ AI
                             │
                     AI ORCHESTRATOR            (domain.agents.AgentOrchestrator)
                             │
      ┌─────────┬─────────┬──┴──────┬──────────┬──────────┐
      ↓         ↓         ↓         ↓          ↓          ↓
  Internet   Telecom    Offers   ZainCash   Security   Support
   Agent      Agent     Agent     Agent      Agent    (Phase 5)
      │         │         │         │          │          │
      └─────────┴─────────┴────┬────┴──────────┴──────────┘
                               │
                         TOOL REGISTRY            (domain.tools.ToolRegistry)
                               │
                         POLICY ENGINE            (domain.policies.PolicyEngine)
                               │
                         RISK ENGINE              (domain.risk.FraudRiskEngine)
                               │
                         API LAYER                (core.network, mocked in :data)
```

Sensitive-operation flow (Zain Cash payments, SIM changes, account-security
changes — build spec §8/§9/§46):

```
Intent → Agent → Tool selection → PolicyEngine.evaluate()
       → [LOW: run] / [MEDIUM: confirm] / [HIGH+: confirm + BiometricAuthManager]
       → Tool.execute() → AppResult
```

The LLM never calls a `Tool` directly and never sees a PIN, password, OTP or
CVV — those parameters don't exist on any repository method signature (see
`ZainCashRepository.kt`'s doc comment).

## Module map

```
app/                      — single Activity, ZainNavGraph, DI entry point
core/
  common/                 — AppResult, AppError, DispatcherProvider
  model/                  — Money (currency-safe, minor-units)
  design/                 — Color/Type/Shape/Theme tokens (ZainIQTheme)
  ui/                     — ServiceCard, PlaceholderScreen, Loading/Error/Empty/Offline
  network/                — NetworkConfig (DEV/STAGING/PRODUCTION), OkHttp/Retrofit DI
  database/                — Room: entities, DAOs, ZainDatabase
  security/               — KeystoreSecureStorage, BiometricAuthManager
  logging/                — ZainLogger (redacts obvious PII patterns)
  analytics/               — AnalyticsTracker (no-op binding today)
  localization/            — LocaleManager (Arabic default, RTL, DataStore-backed)
domain/                   — pure Kotlin/JVM, zero Android deps (see its build.gradle.kts)
  entities/, repositories/, agents/, tools/, policies/, risk/, ai/
data/                     — MockZainRepository, MockZainCashRepository (mocked — no backend yet),
                            AndroidSecurityRepository (REAL — on-device APIs, not a mock),
                            Hilt bindings (DataModule, AgentBindingsModule)
feature/
  home, account, ai, voice, settings, internet, offers,
  zaincash (wallet + recharge + bill payment + card store), security
                          — real
  packages, sim, support, tickets, notifications
                          — placeholders, phase noted per file
```

## Toolchain versions

Pinned in `gradle/libs.versions.toml`, grounded in web-search-verified
current releases as of September 2026 (AGP 9.2.0, Kotlin 2.3.21, Compose
BOM 2026.06.00, Hilt 2.60.1, Room 2.8.5). A few fast-moving, tightly-coupled
entries — notably `ksp` — are a best estimate rather than a confirmed exact
patch; **let Android Studio's Upgrade Assistant / version catalog
suggestions correct anything stale on first sync** rather than trusting
these blindly for the long term.

- `compileSdk`/`targetSdk` = 37, `minSdk` = 26
- Kotlin DSL throughout, version catalog, KSP (not kapt)
- Clean Architecture: `domain` has zero Android dependencies by design

## Security & privacy choices worth knowing about

- `core:security.KeystoreSecureStorage` — AES-256/GCM with a hardware-backed
  (where available) Android Keystore key; nothing sensitive touches plain
  SharedPreferences.
- The Room database (`ZainDatabase`) is **not yet** opened with an encrypted
  passphrase — see the `TODO` in `core/database/.../di/DatabaseModule.kt`.
  Don't ship this as-is with real customer data; that wiring is planned
  alongside the security-hardening pass (build spec §42/§43, Phase 6).
- `AndroidManifest.xml` requests only `INTERNET`, `ACCESS_NETWORK_STATE`,
  and `USE_BIOMETRIC`. No location permission is requested anywhere in this
  project, on purpose — see the manifest's own comment. Don't add one for a
  "Find My Phone"-style feature; that's explicitly out of scope.
- `network_security_config.xml` blocks cleartext traffic app-wide.

## How to continue

Phases, per the build spec's own plan:

1. ~~Architecture, Gradle, modules, design system, navigation, Home~~ ✅
2. Account, AI Chat, Voice UI ✅ — Demo Mode's remaining scenarios (human escalation, ticketing) pending Phase 5
3. Agent Orchestrator real `Agent`s: ~~Internet~~ ✅ ~~Telecom~~ ✅ ~~Offers~~ ✅ — Phase 3 done
4. ~~Zain Cash~~ ✅ — recharge, bill payment (electricity + others), and digital card purchases all working with real confirm+biometric flows. ~~Security Center~~ ✅ — device score, URL scanner, SMS scanner, all real (not mocked). Phase 4 done
5. ~~Tickets, human handoff, notifications, support~~ ✅
6. Security hardening, tests, performance, docs

(Orderii — an e-commerce integration originally in the spec — was dropped
from scope; nothing in the codebase references it any more.)

App Risk Analyzer (`analyze_app_risk`) exists as a repository method + Tool
but has no dedicated screen yet — scanning *all* installed apps needs the
`QUERY_ALL_PACKAGES` permission, which Google Play restricts heavily; the
current implementation analyzes one specific, known package name instead
(reachable via the tool, not yet a browsable list in the UI).

Also worth knowing: `MainActivity` now extends `FragmentActivity` instead
of `ComponentActivity` (needed for `BiometricPrompt`) — if you'd already
started customizing it, re-check that diff.

Also still open from Phase 2/3's own scope: MainActivity doesn't yet read
`ThemeManager`/`LocaleManager` on launch, so a Settings change doesn't
propagate to `ZainIQTheme`/RTL layout direction until that's wired in.

Each phase should be buildable and demo-able on its own before the next
starts — that's what kept this one honest about what's real vs. placeholder,
and it's worth keeping up as the project grows.

## Building the APK (GitHub Actions, from a phone)

1. Create a repo on github.com, then *Add file → Upload files* and upload this
   `.zip` **as-is** (GitHub does not unpack it — the workflow does).
2. *Add file → Create new file*, name it `.github/workflows/build-apk.yml`
   (typing `/` makes folders), paste the contents of `build-apk.yml`, commit.
3. *Actions* tab → watch **Build APK**. On success the APK appears under
   *Releases* (direct download) and as a run artifact.

**Never compiled.** This was written without an Android SDK. The first run
will probably surface compile errors — send the failing step's log and fix
them one round at a time. The `ksp`, AGP, and Compose BOM versions in
`gradle/libs.versions.toml` are the likeliest first failures.
