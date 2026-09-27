# 3D Generator Studio Skill

Reusable skill pack for building future MakerLab browser-based 3D generators.

## Files
- `SKILL.md` — master rules and design/engineering principles.
- `references/generator-patterns.md` — when to borrow patterns from Parts Studio, Plate Studio and SignCraft.
- `templates/generator-spec.md` — lightweight planning template for a new generator.
- `templates/implementation-prompt.md` — reusable prompt for an implementation agent.
- `templates/review-checklist.md` — QA/review prompt after implementation.

## Typical use
Tell your coding agent:

> Read `SKILL.md`. Build a customizable keychain generator. Use `templates/generator-spec.md` to reason about the product, then implement it in the existing project. Preserve existing architecture and functionality.

For an existing generator:

> Read `SKILL.md`. Inspect the existing generator first, then add SVG upload and separate multi-color output without breaking current exports.
