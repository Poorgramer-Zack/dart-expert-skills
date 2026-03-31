---
name: "supabase-best-practices"
description: "Supabase Flutter integration for PostgreSQL, Auth, Realtime subscriptions, Storage, Edge Functions, and Row Level Security. Use when building backend features with supabase_flutter or configuring database policies."
metadata:
  last_modified: "2026-03-12 11:18:17 (GMT+8)"
---

# Supabase Flutter Ecosystem (`supabase_flutter`)

## Goal
[Supabase](https://supabase.com/) is widely acclaimed as the strongest open-source alternative to Firebase. Its core advantage lies in being built upon a powerful **PostgreSQL** relational database, supporting rich SQL queries, Table Joins, and natively providing an excellent Flutter SDK (`supabase_flutter`).

To ensure modularity and LLM readability, this skill is split into logical chapters. Before answering questions or writing code related to Supabase, you **MUST read the relevant reference chapters**.

## Process/Workflow
1. Identify the core Supabase service the user needs (Database, Auth, Storage, etc.).
2. **Read** the corresponding chapter(s) from the `references/` directory.
   - Example: If the user asks about deep linking Magic Links for login, read `01-setup.md` and `02-auth.md`.
3. Synthesize the guidelines exactly as documented within the chapters. **DO NOT hallucinate third-party packages or outdated practices**.
4. Maintain the Serverpod Mini BFF (Backend-For-Frontend) architecture whenever the user requires executing high-privilege operations that should bypass Row Level Security (RLS) entirely.

## Reference Files

*   **Setup & Deep Links**: Read [01-setup.md](file:///home/zack/projects/agent-things/skills/flutter/supabase/references/setup.md) for core initialization and the `app_links` requirement.
*   **Authentication**: Read [02-auth.md](file:///home/zack/projects/agent-things/skills/flutter/supabase/references/auth.md) for Email/OTP, Apple/Google native sign-in, and listening to stream changes.
*   **Database (Postgres)**: Read [03-database.md](file:///home/zack/projects/agent-things/skills/flutter/supabase/references/database.md) for strongly-typed `Freezed` mapping, `.select()`, and join strategies natively.
*   **Realtime & Presence**: Read [04-realtime.md](file:///home/zack/projects/agent-things/skills/flutter/supabase/references/realtime.md) for Postgres CDC subscriptions and User Presence management.
*   **Storage**: Read [05-storage.md](file:///home/zack/projects/agent-things/skills/flutter/supabase/references/storage.md) for cross-platform (Mobile vs Web) bucket manipulation.
*   **Edge Functions**: Read [06-edge-functions.md](file:///home/zack/projects/agent-things/skills/flutter/supabase/references/edge-functions.md) for executing remote serverless workflows.
*   **Serverpod Mini (BFF)**: Read [07-serverpod-mini.md](file:///home/zack/projects/agent-things/skills/flutter/supabase/references/serverpod-mini.md) for leveraging the pure `supabase` Dart library combined with the `SERVICE_ROLE_KEY` inside a secure backend.
