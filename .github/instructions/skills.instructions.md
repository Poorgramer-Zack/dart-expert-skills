---
applyTo: skills/**/*.md
---

# Agent Skill Authoring Best Practices

These rules apply whenever creating or updating any `SKILL.md` file in this repository.

## YAML Frontmatter Requirements

Every SKILL.md must begin with valid YAML frontmatter:

```yaml
---
name: "skill-name-in-gerund-form"
description: "Third-person description of what the skill does and when to use it."
metadata:
  last_modified: "YYYY-MM-DD HH:MM:SS (GMT+8)"
---
```

**`name` constraints:**
- Max 64 characters
- Lowercase letters, numbers, and hyphens only
- Prefer gerund form: `processing-pdfs`, `managing-state`, `testing-code`
- Never: `helper`, `utils`, `tools`, or reserved words (`anthropic`, `claude`)

**`description` constraints:**
- Max 1024 characters, must be non-empty
- Always write in **third person**: "Processes Excel files..." not "I can help you..."
- Include both *what* the skill does and *when* to use it (specific triggers/contexts)
- Include key terms that enable discovery when 100+ skills are loaded

## Conciseness Rules

The context window is shared. Every token in SKILL.md competes with conversation history.

- Assume Claude already knows standard library/framework concepts — do not over-explain
- Challenge every paragraph: "Does Claude need this, or does it already know it?"
- Keep SKILL.md body under 500 lines
- Split long content into separate reference files (e.g., `references/ADVANCED.md`)

## Content Structure

### Progressive Disclosure

SKILL.md is the table of contents. Detailed material lives in separate files loaded on demand:

```
skills/<name>/
  SKILL.md              # metadata + concise overview + links to references
  references/
    REFERENCE.md        # detailed API / schema reference
    EXAMPLES.md         # input-output example pairs
    ADVANCED.md         # edge cases, migration guides, old patterns
```

- Keep all reference links **one level deep** from SKILL.md — never nest references inside references
- For reference files over 100 lines, include a table of contents at the top

### Degrees of Freedom

Match instruction specificity to task fragility:

| Situation | Style | Example |
|-----------|-------|---------|
| Multiple valid approaches, context-dependent | High freedom (prose) | Code review guidance |
| Preferred pattern exists, some variation OK | Medium freedom (pseudocode) | API call patterns |
| Fragile operation, exact sequence required | Low freedom (exact script) | Database migrations |

### Consistent Terminology

Pick one term per concept and use it everywhere:
- Endpoint **or** route — not both
- Field **or** property — not both
- Fetch **or** retrieve — not both

### Avoid Time-Sensitive Information

Do not hardcode dates, "latest" claims, or version-specific statements as permanent facts.
Instead, use an "Old patterns / Deprecated" section for historical context:

```markdown
## Old Patterns (pre-v2)
These patterns still work but are superseded by the approach above.
```

## Version Update Rules (for automated weekly updates)

When updating package versions in a SKILL.md:

1. **PATCH bump only** (`x.y.Z`): Skip — no changes needed.
2. **MINOR bump** (`x.Y.z`): Update all version pins in pubspec.yaml code blocks. Preserve all existing content and structure.
3. **MAJOR bump** (`X.y.z`): Update version pins AND revise any code examples or descriptions that reference changed APIs. Move old patterns to a "Old Patterns" section rather than deleting them.

When revising code examples for a MAJOR bump:
- Preserve the overall SKILL.md structure and section headings
- Do not remove working examples — move them to `references/` if necessary
- Update the `metadata.last_modified` field with today's date

## Quality Checklist

Before writing or updating a SKILL.md, verify:

- [ ] Frontmatter `name` uses gerund form and lowercase-hyphenated
- [ ] `description` is third-person and includes discovery triggers
- [ ] No explanation of things Claude already knows
- [ ] Body is under 500 lines (or splits to reference files)
- [ ] All reference links are one level deep
- [ ] Terminology is consistent throughout
- [ ] No time-sensitive claims in permanent sections
- [ ] `metadata.last_modified` is up to date
