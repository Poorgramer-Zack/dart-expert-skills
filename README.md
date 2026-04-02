# Flutter Skills 🚀

A comprehensive library of modular AI Skills for Flutter & Dart development, designed for precise, production-ready AI Agent workflows. Each skill provides structured knowledge and expert-level guidance optimized for LLM consumption and human readability.

---

## 🏗️ Skill Architecture

All skills follow a modular, flat structure for maximum clarity and AI compatibility.

### Directory Structure
```text
skills/
└── <skill-name>/
    ├── SKILL.md        # Entry point: Goals, Process, Constraints
    └── references/     # Detailed implementation guides
```

### Core Design Principles
1. **Single Responsibility**: Each skill focuses on one technology or framework
2. **LLM-Optimized**: Documents are structured for AI context windows with semantic headings
3. **Production-Ready**: Working code snippets, not placeholders
4. **Version-Aware**: All packages use latest stable versions (as of 2026-03-31)

---

## 📚 Skill Categories

### State Management
- **`flutter-provider`**: Provider (v6.1.5+1) - ChangeNotifier + MVVM with memory leak prevention and disposal patterns
- **`flutter-riverpod`**: Riverpod (v3.3.1) - Riverpod 3.0 compile-safe reactive state; `Consumer` scoped rebuilds; generic providers; mutations & offline persistence (experimental)
- **`flutter-bloc`**: BLoC (v9.1.1) - Event-driven architecture with transformers and Freezed integration

### Database & Storage
- **`flutter-shared-preferences`**: SharedPreferences (v2.5.5) - Simple key-value for non-sensitive settings
- **`flutter-secure-storage`**: SecureStorage (v10.0.0) - Encrypted storage with biometric support, iOS Keychain & Android Keystore
- **`flutter-hive`**: Hive CE (v2.19.x) - High-performance NoSQL object storage
- **`flutter-drift`**: Drift (v2.32.x) - Type-safe reactive SQL with WAL mode and sqlite3 v3.x

### Routing & Navigation
- **`flutter-gorouter`**: GoRouter (v17.1.0) - Official declarative routing with deep linking and redirect loop prevention
- **`flutter-autoroute`**: AutoRoute (v11.1.0) - Type-safe routing with code generation

### Authentication & Social Login
- **`flutter-social-auth`**: OAuth/SSO for Google, Apple, Facebook, LINE with error handling and common gotchas
- **`flutter-deeplink`**: App Links and Universal Links configuration

### Advanced Features
- **`flutter-expert`**: Architecture decision matrix, Clean/Feature-First patterns, performance optimization
- **`flutter-responsive`**: Multi-screen layouts with breakpoints
- **`flutter-animate`**: High-performance animations and physics-based effects
- **`flutter-hooks`**: Functional widget lifecycle management
- **`flutter-isolate`**: Concurrency with ReceivePort cleanup and memory leak prevention
- **`flutter-testing`**: Unit, widget, integration, golden tests with CI/CD troubleshooting
- **`flutter-genui`**: AI-powered UI generation
- **`marionette`**: AI-driven Flutter E2E testing via VM Service CLI — tap, scroll, enterText, screenshot on live debug builds

### UI Components & Theming
- **`shadcn-flutter`**: Modern accessible UI components with theming
- **`effective-dart`**: Dart 3.10 best practices — dot shorthands, null-aware collection elements, wildcard variables, records, patterns, sealed classes

### Data Visualization
- **`fl-chart`**: FL Chart (v1.2.0) — Line, Bar, Pie, Scatter, Radar, Candlestick charts with implicit animations, touch tooltips, and pan/zoom transformations

### Backend Integration
- **`supabase`**: Realtime Postgres, Edge Functions, Authentication
- **`serverpod`**: Full-stack Dart server with type-safe endpoints
- **`firebase`**: Analytics, Crashlytics, Authentication, Storage

### Utilities & Tools
- **`freezed`**: Immutable models and union types with pattern matching
- **`fpdart`**: Functional programming (Option, Either, Task)
- **`openapi-to-dart`**: Manual Dart API client implementation from OpenAPI specs using Dio + Equatable/Freezed
- **`ts-to-dart`**: TypeScript to Dart conversion utilities
- **`sentry-flutter`**: Error tracking and performance monitoring
- **`revenuecat-flutter`**: Subscription billing and payment integration
- **`flutter-ads`**: AdMob monetization and mediation

### Web Development
- **`jaspr`**: Dart-only SSR/SPA web framework

### DevOps & CI/CD
- **`flutter-setup`**: Environment setup with PATH configuration and troubleshooting for macOS/Linux/Windows
- **`github-actions`**: CI/CD automation workflows
- **`fastlane`**: Automated deployment pipelines
- **`codemagic`**: Cloud CI/CD for Flutter apps

---

## 🎯 Development Philosophy

> "Simplicity is the ultimate sophistication."

- **Minimalist Approach**: Focus on user experience, not engineering abstractions
- **Modern Best Practices**: Latest stable releases with version documentation
- **Type Safety**: Leverage Dart 3+ features (pattern matching, records, sealed classes)
- **Zero Placeholder Policy**: All code snippets are production-ready

---

## 🚀 Recent Updates (2026-04-02)

### fl-chart — Flutter Data Visualization (v1.2.0)
- ✅ **6 chart types**: LineChart, BarChart, PieChart, ScatterChart, RadarChart, CandlestickChart
- ✅ **Quick start examples** for all chart types with production-ready code
- ✅ **Common config patterns**: titles/axes, grid, border, extra lines, touch interactivity, animations, pan/zoom
- ✅ **Advanced patterns**: stacked bars with per-layer tooltip, multi-series gradient lines, donut chart touch expansion, multi-dataset radar overlay, horizontal bar rotation
- ✅ **7 reference files**: full API property tables for each chart type + shared base components

### Previous Updates (2026-04-01)

### Fleet Mode Quality Audit — All 36 Skills
- ✅ **36 SKILL.md files** audited and standardized to Agent Skill Best Practices
- ✅ **Gerund naming** enforced (`applying-*`, `managing-*`, `testing-*`)
- ✅ **Third-person descriptions** with trigger keywords for precise AI activation
- ✅ **500-line limit** enforced — `flutter-bloc` (594→147 lines) and `flutter-riverpod` (625→162 lines) split with new `references/` files

### Riverpod 3.0 Full Coverage
- ✅ **Breaking changes**: `==` equality for all state comparisons, fresh Notifier instances on rebuild (resource leak risk), `FamilyNotifier` removed, `Ref` generics removed
- ✅ **Consumer vs ConsumerWidget**: decision table; prefer `Consumer` + `ref.watch().select()` for surgical rebuilds
- ✅ **New APIs**: generic providers, `@Dependencies` scoping, `AsyncValue` sealed + `progress`, weak listeners, `overrideWithBuild`
- ✅ **Mutations (experimental)**: `tsx.get()` pattern, exhaustive 4-state switch (`Idle/Pending/Error/Success`)
- ✅ **Offline Persistence (experimental)**: `riverpod_sqflite`, `persist()` in `build()`, `StorageOptions`

### effective-dart — Dart 3.7–3.10 Features
- ✅ **Dot shorthands** (3.10): `.blue` instead of `Color.blue`, `.all(16)` instead of `EdgeInsets.all(16)`
- ✅ **Null-aware collection elements** (3.8): `[?nullable]` auto-excludes nulls
- ✅ **Wildcard variables** (3.7): non-binding `_`, multiple `_` in same scope
- ✅ **New lints** (3.9): `switch_on_type`, `unnecessary_unawaited`, `@awaitNotRequired`

### openapi-to-dart — Reframed as Manual Implementation Guide
- ✅ Removed code-generation framing; now covers Dio client setup, Equatable vs Freezed decision table, Service/Repository layers, `DioException → ApiError` mapping

---

## 📖 Usage

Each skill is self-contained and can be used independently. AI agents can load specific skills based on task requirements for precise, expert-level guidance.

### Skill Structure
```markdown
---
name: "skill-name"
description: "Detailed description with trigger keywords for AI agents"
metadata:
  last_modified: "YYYY-MM-DD HH:MM:SS (GMT+8)"
---

## Goal
Clear objective and use cases

## Process
Step-by-step implementation guide

## Reference Documentation
Links to detailed guides in references/

## Constraints
Critical rules and best practices
```

---

## 🤝 Contributing

All skills follow standardized formats optimized for LLM consumption. When adding new skills:
1. Use single responsibility principle
2. Include version-specific package information
3. Provide working code examples
4. Add trigger keywords in description for precise AI activation

---

> [!IMPORTANT]
> All documents follow the metadata convention:
> `metadata.last_modified: "YYYY-MM-DD HH:MM:SS (GMT+8)"`

