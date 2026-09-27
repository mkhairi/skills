# Implementation Prompt

Read and follow `SKILL.md` before making changes.

Build the requested 3D generator as a purpose-built manufacturing tool. First inspect the existing codebase and reuse its framework, rendering engine, geometry stack, state patterns, components, styling and export implementation wherever practical.

Infer sensible product requirements from the requested object instead of waiting for every parameter to be specified. Classify the generator as parametric, text-driven, template-driven, composition-driven or hybrid, and borrow the relevant established patterns from `references/generator-patterns.md`.

Before implementation, determine the minimum useful controls, smart defaults, derived dimensions, validation, tolerances, output parts and appropriate export formats. The generator must open with a valid useful example already visible.

Keep geometry logic separate from UI. Prefer reusable primitives and existing shared components. Do not introduce another rendering or geometry stack without a strong reason. Do not rewrite unrelated working functionality.

Prioritize physical/manufacturing correctness over decorative UI. Maintain real-world millimetre dimensions and explicit tolerances. Ensure exported geometry matches the editor dimensions.

Provide responsive desktop/mobile UX, System/Light/Dark themes, live preview, camera fit and useful user-facing validation. Use multi-part aligned geometry when it materially helps multi-color printing or assembly.

Before finishing, run through `templates/review-checklist.md` and fix discovered regressions or obvious geometry/UX issues.
