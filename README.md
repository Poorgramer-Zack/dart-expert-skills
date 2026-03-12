# Flutter Skills 🚀

A high-quality library of modular AI Skills designed for precise, production-ready Agentic AI workflows. This repository provides structured knowledge and development guidelines that allow AI Agents (and senior developers) to execute complex tasks with expert-level proficiency across the Flutter & Dart ecosystem.

---

## 🏗️ Skill Architecture

All skills in this library are meticulously crafted following a modular, flat structure designed for maximum LLM-readability and developer clarity.

### Directory Structure
```text
skills/flutter/
└── <skill-name>/
    ├── SKILL.md        # Entry point: Goals, Instructions, and Constraints.
    └── references/     # Granular documentation for sub-modules (No numbering).
```

### Core Design Principles
1.  **Minimalist Logic**: We reject over-engineering. Code is direct, flat, and avoids unnecessary abstractions.
2.  **LLM-First Formatting**: Documents are optimized for AI context windows—no complex numbering systems, just semantic, meaningful headings.
3.  **Credential Bridge Pattern**: Standardized patterns for integrating native social providers and backend services (Firebase/Supabase/Serverpod).

---

## 🛠️ Skill Categories

### 1. Core Framework & UI/UX
- **`flutter-expert`**: Advanced layouts, animations (3.41+), platform views, and performance scaling.
- **`flutter-routing`**: Mastering `go_router` with type-safe parameters and dynamic redirects.
- **`flutter-responsive`**: Handling multi-screen layouts with professional breakpoints.
- **`flutter-animate`**: High-performance micro-animations and physics-based effects.
- **`shadcn-flutter`**: Modern, accessible UI components with robust theming.
- **`flutter-hooks`**: Clean implementation of functional widget lifecycle management.

### 2. State & Data Persistence
- **`flutter-state-management`**: Provider and Riverpod best practices (AsyncNotifier, StreamProvider).
- **`flutter-db`**: Secure storage, Shared Preferences, and complex database architectures.
- **`freezed`**: Professional immutable models and Union types with Dart 3 pattern matching.
- **`fpdart`**: Functional programming patterns (Option, Either, Task) for resilient logic.
- **`openapi-to-dart`**: Efficient API client generation from OpenAPI/Swagger specs.

### 3. Backend & Authentication
- **`supabase`**: Realtime Postgres, Edge Functions, and native authentication flows.
- **`serverpod`**: Full-stack Dart architecture (v2/v3) including migrations and testing.
- **`firebase`**: Professional setup for Analytics, Crashlytics, and App Check security.
- **`flutter-social-auth`**: Native integration for Google, Apple, Facebook, and LINE login.
- **`flutter-deeplink`**: Specialized configuration for App Links and Universal Links.

### 4. Utilities, Marketing & Web
- **`flutter-testing`**: Automation strategies: Unit, Widget, integration (Patrol), and Golden tests.
- **`sentry-flutter`**: Advanced observability and error trapping.
- **`revenuecat-flutter`**: Subscription-based billing and payment gateway integration.
- **`flutter-ads`**: Professional monetization using AdMob and Mediation.
- **`jaspr`**: High-performance web development with Dart-only SSR/SPA (Jaspr framework).
- **`fastlane` / `codemagic`**: Automated CI/CD and production deployment pipelines.

---

## 🎨 Development Philosophy

> "If the code isn't as simple as it can be, it's not finished."

- **Senior Minimalist Approach**: We focus on the core user experience rather than engineer-favored abstractions.
- **Modern Best Practices**: We strictly follow current stable releases (e.g., `google_sign_in` v7.x, `supabase` v2.x).
- **Zero Placeholder Policy**: Every reference contains working, production-ready snippets, not "todo" comments.

---

> [!IMPORTANT]
> All documents in this repository follow the metadata convention:
> `metadata.last_modified: "YYYY-MM-DD HH:MM:SS (GMT+8)"`

