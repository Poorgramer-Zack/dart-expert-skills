# Flutter Skills Repository - Copilot Instructions

## Repository Overview

This is a comprehensive library of modular AI Skills for Flutter & Dart development. Each skill provides structured knowledge and expert-level guidance optimized for both LLM consumption and human readability.

**Core Philosophy**: Minimalist approach focusing on user experience over engineering abstractions. All code snippets are production-ready with zero placeholders. Latest stable package versions with explicit version documentation.

## Architecture

### Directory Structure
```
skills/
└── <skill-name>/
    ├── SKILL.md        # Entry point: Goals, Process, Constraints
    └── references/     # Detailed implementation guides (optional)

.agents/
└── skills/
    └── skill-creator/  # Meta-skill for creating and optimizing skills
```

### Skill Structure
All skills follow a standardized format:
- **YAML frontmatter**: `name`, `description` (required), `metadata.last_modified`
- **Description field**: Acts as primary triggering mechanism for AI agents - includes both what the skill does AND specific contexts for when to use it
- **Body sections**: Goal, Process, Reference Documentation, Constraints
- **Progressive disclosure**: Keep SKILL.md under 500 lines; split larger content into `references/` directory

### Key Design Principles
1. **Single Responsibility**: Each skill focuses on one technology/framework
2. **LLM-Optimized**: Semantic headings, structured for AI context windows
3. **Production-Ready**: Working code snippets with latest stable versions
4. **Version-Aware**: All packages use latest stable versions (last updated 2026-03-31)

## Working with Skills

### Creating or Modifying Skills

**CRITICAL**: When creating or modifying skills:
1. Use the `.agents/skills/skill-creator` skill for guidance
2. Follow the metadata convention: `last_modified: "YYYY-MM-DD HH:MM:SS (GMT+8)"`
3. Include version numbers in package references (e.g., "Riverpod v3.3.1")
4. Add trigger keywords in description for precise AI activation
5. Keep descriptions "pushy" to combat under-triggering - be explicit about when to use

### Skill Categories

**State Management**: provider, riverpod, bloc  
**Database & Storage**: shared-preferences, secure-storage, hive, drift  
**Routing**: gorouter, autoroute  
**Authentication**: social-auth, deeplink  
**Backend**: supabase, serverpod, firebase  
**Testing**: flutter-testing  
**Advanced**: flutter-expert, flutter-genui  
**UI**: shadcn-flutter, flutter-responsive, flutter-animate, flutter-hooks  
**Utilities**: freezed, fpdart, openapi-to-dart, ts-to-dart, sentry-flutter  
**DevOps**: flutter-setup, github-actions, fastlane, codemagic

### Reading Skills

When a skill is referenced:
1. Start with SKILL.md for overview and process
2. Load specific `references/*.md` files only as needed
3. For large reference files (>300 lines), check table of contents first

## Conventions

### Code Style
- All code comments MUST be in English only (per custom instructions)
- All user-facing communication in Traditional Chinese (per custom instructions)
- Follow minimalist principles: flat structure unless modularity explicitly requested
- Leverage Dart 3+ features: pattern matching, records, sealed classes
- No placeholder code - all snippets must be production-ready

### Documentation
- Use semantic markdown headings for LLM navigation
- Include version numbers for all package references
- Add "❌ Anti-pattern" and "✅ Best practice" examples for clarity
- Decision tables for technical choices (when multiple options exist)

### File Organization
- Organize by business domain, not technical type
- Use kebab-case for file and directory names
- Keep skills atomic - one skill per technology/framework

## MCP Servers

This repository uses two MCP servers (configured in `.mcp.json`):
- **exa**: Web search and content crawling (for research and documentation lookup)
- **dart**: Dart/Flutter language server integration

## Common Tasks

### Adding a New Skill
1. Reference `.agents/skills/skill-creator/SKILL.md` for the creation process
2. Create directory: `skills/<skill-name>/`
3. Create `SKILL.md` with YAML frontmatter and standardized sections
4. Add `references/` directory if content exceeds 500 lines
5. Update main README.md skill categories section
6. Set `metadata.last_modified` to current timestamp (GMT+8)

### Updating Package Versions
When updating a skill for new package versions:
1. Update version numbers in SKILL.md description and code examples
2. Review `references/*.md` for version-specific changes
3. Update breaking changes and migration patterns
4. Add edge cases and troubleshooting for new version
5. Update `metadata.last_modified`

### Split Skills (Modular Approach)
Recent refactoring split monolithic skills into focused modules:
- `flutter-state-management` → `flutter-provider`, `flutter-riverpod`, `flutter-bloc`
- `flutter-db` → `flutter-shared-preferences`, `flutter-secure-storage`, `flutter-hive`, `flutter-drift`
- `flutter-routing` → `flutter-gorouter`, `flutter-autoroute`

Original aggregated skills moved to `skills/archived/` for reference.

## Edge Cases & Troubleshooting

### Skill Triggering
- If a skill isn't triggering appropriately, optimize the `description` field
- Make descriptions explicit and "pushy" about when to use
- Include trigger keywords: technical terms, common user phrases, edge cases

### Production Best Practices
All core skills (as of 2026-03-31 update) include:
- Memory leak prevention patterns
- Error handling and edge cases
- CI/CD troubleshooting guides
- Decision tables for technical choices
- Anti-patterns with explanations

### Cross-Platform Considerations
When working with Flutter code:
- Test golden files have platform-specific rendering issues (covered in flutter-testing)
- Secure storage has Android/iOS-specific configuration (covered in flutter-secure-storage)
- Deep linking requires platform-specific setup (covered in flutter-deeplink)

## Version History

**2026-03-31 Fleet Mode Deep Update**:
- 8 core skills updated with production best practices
- 4,980+ lines of new content: memory leaks, error handling, CI/CD
- 85+ code examples with anti-patterns and best practices
- 12 decision tables for technical choices
- Major package updates: GoRouter v17.1.0, Drift v2.32.x, Riverpod v3.3.1, BLoC v9.1.1, SecureStorage v10.0.0
