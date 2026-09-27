# 3D Generator Studio

## Purpose
Build polished browser-based 3D generator and customization tools for MakerLab. Use this skill for focused tools where users configure, compose, preview, and export objects for 3D printing, CNC, laser cutting, signage, fabrication, prototyping, or workshop use.

Generators may be parametric, text-driven, template-driven, composition-driven, or hybrid.

Core principle: **Choose → Customize → Preview → Export**.

A useful default object should appear immediately. Do not build a full CAD application unless explicitly required.

## Generator families

### Parametric
Dimensions drive geometry: gears, pulleys, bolts, nuts, spacers, bushings, couplers, brackets, pipe fittings, enclosures, adapters, knobs, handles and jigs.

Pipeline: parameters → validation → geometry → preview/export.

### Text / Identity
Text becomes manufacturing geometry: number plates, name plates, badges, door labels, house numbers, keychains, tags and raised lettering.

Support font, size, spacing, alignment, multiline, extrusion, raised/recessed/cut-through modes and automatic fitting where appropriate.

### Composition
Users combine backing shapes, text, icons, QR codes, SVGs and other elements. Use an object/layer model with position, rotation, scale/dimensions, depth, operation, visibility and locking where useful.

### Template
Start from editable useful designs rather than an empty canvas. Templates should instantiate normal editable objects.

### Hybrid
Combine approaches when the physical product needs it. Example: a keychain can combine a parametric body, text geometry, a mounting hole, optional icon and separate multicolor parts.

## Shared architecture
Prefer a reusable Studio shell with generator metadata, parameters/presets/templates, validation, geometry, workspace, measurements and export. Adding a generator should primarily add a generator definition, geometry implementation and generator-specific controls.

Keep UI, geometry, generator definition, export and state separated. Reuse existing dependencies and architecture when modifying an existing project.

## Generator definition
Each generator should conceptually provide: id, name, category, description, version and capabilities. Capabilities can include parametric, text, layers, templates, svg, icons, qr, boolean, multi-part, multi-color, 2d-editor, stl, 3mf, obj and svg-export.

## Parameters
Prefer real-world parameters: width, height, diameter, thickness, wall thickness, hole diameter, length, angle, radius, pitch, tooth count, clearance, tolerance, text height, letter spacing and extrusion height.

Use millimetres internally unless there is a strong reason not to. Always show units. Precise dimensions must allow keyboard input; sliders should not be the only input method.

Automatically derive dimensions that the user should not have to calculate. Show important derived values.

## Smart defaults and presets
Defaults must immediately generate a valid, useful and visually understandable object. Never default to invalid or empty geometry unless blank state is fundamental.

Provide presets for common standards/configurations where useful, while allowing overrides. Never claim standards compliance unless actually implemented.

## Validation
Prevent impossible geometry and explain problems in physical/user language rather than geometry-engine jargon. Example: “Hole diameter is larger than the keychain height,” not “Boolean operation failed.”

Warnings may inform about thin walls, tiny text strokes or edge proximity. Block export only for genuinely invalid/unusable geometry.

## Tolerance
For mating parts, support explicit configurable clearance. Never secretly alter dimensions. Display the applied tolerance.

## Text and fonts
Text intended for manufacture must become actual geometry. Support useful built-in fonts and local TTF/OTF uploads where practical. Handle unsupported glyphs gracefully.

For constrained layouts, auto-fit by preferring font-size, spacing and layout adjustments before non-uniform character distortion.

## Backing shapes
Where appropriate support reusable profiles such as rectangle, rounded rectangle, capsule, circle, oval, arch, shield, hexagon, octagon, tag, arrow and custom SVG. Common parameters include width, height, corner radius, thickness, border, bevel/chamfer and mounting holes.

## Objects and layers
Composition tools may support select, move, duplicate, delete, lock, hide, reorder, center, align and distribute. Add grouping only when useful. Avoid Illustrator/CAD-level complexity.

Support helpful snapping to canvas center, axes, object edges/centers and optional grid. Allow disabling snapping when appropriate.

## 2D workspace
Use a 2D workspace when precise placement matters. It should represent physical dimensions and may include millimetre rulers, grid, safe area, guides, snapping, zoom, pan and selection bounds. Maintain predictable mapping between editor and export.

## 3D preview
Provide orbit, pan, zoom, reset camera and fit object. Add top/front/side/orthographic views when useful. Automatically fit geometry after generation or major size changes.

Use a clean CAD/workshop presentation: neutral background, subtle grid, soft lighting, readable edges and strong depth perception. The object is the hero.

## Live geometry
Typical pipeline: input → validation → derived values → profiles → geometry → preview → measurements. Debounce expensive work. Use Web Workers where beneficial. Avoid unnecessary rerenders and scene rebuilds.

## Geometry
Prefer reusable primitives such as box, roundedBox, cylinder, tube, polygon, path, extrude, revolve, offset, textShape, thread and hole, plus union/subtract/intersect.

Generated models should be manifold/watertight where possible, consistently oriented, dimensionally predictable and slicer-friendly. Avoid excessive polygon counts. Draft/Normal/High quality modes may be useful; default to Normal.

## Boolean modes
Expose understandable operations such as Add, Subtract, Engrave and Cut Through instead of requiring users to know CSG terminology.

## Multi-part and multi-color
Allow combined or separate output when useful. Separate parts should share the same coordinate system so they align automatically in a slicer. Prefer 3MF when richer multi-part metadata can be preserved reliably.

Possible output modes: Combined, Separate Parts, Characters Only, Backing Only.

## Measurements and print-bed awareness
Show important physical dimensions and optionally bounding box, volume and part count. Do not claim accurate print time without slicer data.

Where useful, allow common/custom print-bed sizes and warn when the model exceeds the selected bed. Do not try to replace a slicer.

## Export
Primary 3D format: STL. Support 3MF, OBJ, STEP, SVG or PNG only when technically appropriate. Do not claim true STEP unless the geometry stack produces valid CAD/BREP STEP geometry.

Use descriptive sanitized filenames. Before export validate parameters, geometry existence, non-empty mesh, positive dimensions, required text/objects and known fatal geometry errors.

## SVG, icons and QR
SVG import should parse paths, normalize coordinates, scale predictably, convert to profiles and extrude when required. Handle malformed input gracefully.

QR: data → QR matrix → 2D modules → geometry. Warn if physical size is likely too small to remain usable.

Prefer local processing for uploaded fonts, SVGs, logos, text and QR data. Do not silently upload user assets to third parties.

## URL state and saved designs
Where practical encode simple configuration in the URL for sharing/bookmarking. Store parameters, not generated meshes.

Saved designs should conceptually store generator_id, generator_version, name, parameters, objects and timestamps. Version generator algorithms so old designs remain reproducible where practical.

## Undo/redo
Composition-heavy generators should support meaningful action history such as Add Text, Move Object, Change Font, Delete Icon and Resize Backing.

## Layout
Desktop should normally fit within `100dvh` with no unnecessary browser-level vertical scrolling. Parameter/layer panels can scroll internally while the preview consumes remaining space.

Desktop: sidebar + large workspace. Tablet: compact sidebar + workspace. Mobile: preview plus tabs/drawer/bottom controls. Do not merely shrink desktop UI.

## Theme
Support System, Light and Dark. Default to System, follow OS preference and persist explicit selection. Preview/grid/guides must remain readable in every theme.

## Visual language
Aim for modern maker software + lightweight CAD utility: compact UI, clear hierarchy, subtle borders, restrained shadows, strong numerical readability, purposeful icons, moderate rounding and high information density.

Avoid generic SaaS-dashboard styling, toy 3D viewers and unnecessarily complex professional-CAD patterns.

## Existing project rule
Before changing an existing generator, inspect the project first: framework, rendering engine, geometry engine, state management, exports, styling, shared components and primitives. Reuse working patterns and dependencies. Do not introduce a second rendering/geometry stack or rewrite unrelated functionality without a strong reason.

## Planning a new generator
When the user gives a short request, infer the minimum useful product rather than requiring a complete specification.

Determine:
1. What physical object is being manufactured?
2. What does the user actually need to customize?
3. Is it parametric, text, template, composition or hybrid?
4. What are the essential parameters and smart defaults?
5. What values can be derived automatically?
6. What combinations create invalid geometry?
7. Is tolerance required?
8. Are multiple aligned parts useful?
9. Which export formats genuinely make sense?
10. What default example immediately communicates the tool?

Use progressive complexity: put common controls first and specialized controls under Advanced.

## Established reference patterns
- **3D Parts Studio**: parametric geometry, mechanical dimensions, presets, engineering controls and tolerances.
- **Plate Studio**: constrained typography, physical plate dimensions, raised characters and separate character/backing output.
- **SignCraft**: composition, layers, text, SVG, icons, QR, templates, backing shapes, alignment and multi-part designs.

New generators may combine these patterns. Example: Keychain = Parts Studio dimensions + Plate Studio typography + SignCraft icon/layer concepts.

## Completion checklist
Verify useful default object, understandable controls, visible units, valid geometry, correct dimensions, working cuts/holes/text, aligned separate parts, camera fit, useful errors, reset/presets, exports, descriptive filenames, responsive layout, System/Light/Dark themes and no regression to existing generators.

## Final principle
Every generator should feel like a purpose-built manufacturing tool, not generic CAD with a different title.

The intended experience is:

**open generator → change a few meaningful settings → see exactly what will be made → download → manufacture**.
