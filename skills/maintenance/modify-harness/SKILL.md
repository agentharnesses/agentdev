---
name: modify-harness
description: Update harness structure files — HARNESS.md, SKILLS.md indexes, REFERENCES.md — to keep routing and descriptions accurate as the harness evolves.
---

## Role

Keep the harness self-consistent when skills or references are added, renamed, or removed.

## What to do

1. Use reverse progressive disclosure (via the `agent-harnesses` skill) to find which index files reference the target path
2. Read the current state of each affected file
3. Apply the change: add, update, or remove the relevant entry
4. Ensure descriptions remain accurate and routing summaries reflect actual contents

## Conventions

- Keep `HARNESS.md` `## Skills` and `## References` sections in sync with `skills/SKILLS.md` and `references/REFERENCES.md`
- Update the `description` frontmatter in `HARNESS.md` when the harness scope changes
- Skill descriptions should be actionable: "Use when..." not "This skill..."
- Reference documents should be stable facts; skill buckets contain executable guidance
- Prefer updating existing skill buckets over creating new ones when scope overlaps
