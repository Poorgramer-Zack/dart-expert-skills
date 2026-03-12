# Agent Things 🚀

A high-quality library of modular AI Skills designed for precise, production-ready Agentic AI workflows. This repository provides structured knowledge and development guidelines that allow AI Agents to execute complex tasks with expert-level proficiency across various technical domains.

---

## 🏗️ Skill Architecture

All skills in this library are meticulously crafted following the **Anthropic SKILL structure**. This ensures that AI Agents can reliably parse, understand, and apply the rules defined within each module.

### Directory Structure
```text
skills/
└── <category>/
    └── <skill-name>/
        ├── SKILL.md        # Entry point with goals, instructions, and metadata.
        ├── references/     # (Optional) In-depth documentation for sub-modules.
        └── resources/      # (Optional) Static assets, templates, or helper scripts.
```

### Core Components
1.  **SKILL.md**: The primary definition file. It uses YAML Frontmatter for identification and contains the core logic (Goal, Instructions, Constraints).
2.  **References**: Complex skills (like `supabase` or `flutter-expert`) are decomposed into granular modules within the `references` folder to maintain clarity and focus.
3.  **Standardized Metadata**: Every document includes a `metadata` block to track provenance and updates, ensuring the Agent always operates on the latest standards.

---

## 🛠️ Featured Technical Skills

The library currently focuses on a comprehensive Flutter and full-stack ecosystem:

### 1. Flutter Core & Architecture
- **`flutter-expert`**: Advanced UI/UX, animations, native platform interop, and performance scaling.
- **`flutter-routing`**: Mastering `go_router` for robust navigation and dynamic parameter handling.
- **`flutter-state-management`**: Clean implementations of modern state management (Provider, Riverpod, BLoC).

### 2. Ecosystem & Third-Party Integration
- **`supabase`**: Full-stack integration covering Auth, Realtime Postgres, and Edge Functions.
- **`firebase`**: Professional setup for Analytics, Messaging, and Cloud Storage.
- **`flutter-deeplink`**: Specialized configuration for Android App Links and iOS Universal Links.
- **`revenuecat-flutter`**: Implementation of robust in-app subscription and payment systems.

### 3. Engineering & Automation
- **`flutter-testing`**: Comprehensive automation strategies including Unit, Widget, and Golden tests.
- **`codemagic` / `github-actions`**: Modern CI/CD pipeline configurations for mobile and web.
- **`fastlane`**: Streamlined deployment processes for the App Store and Google Play.

---

## 🎨 Development Philosophy

- **Minimalist Seniority**: We prioritize direct, flat code over unnecessary abstractions.
- **Self-Explaining Design**: Architecture that is intuitive for both humans and AI.
- **Production-Ready**: Every line of code and instruction follows current industry best practices, avoiding technical debt from the start.

---

> [!NOTE]
> All documents in this repository follow the metadata convention:
> `metadata.last_modified: "YYYY-MM-DD HH:MM:SS (GMT+8)"`
