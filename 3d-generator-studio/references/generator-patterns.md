# Generator Pattern Reference

## 3D Parts Studio pattern
Use when the object is primarily dimension/engineering driven.

Typical needs:
- numeric dimensions in mm
- presets/standards
- tolerances and clearance
- holes, booleans, threads
- calculated measurements
- mesh quality
- STL/3MF export

Examples: gear, pulley, pipe fitting, bracket, enclosure, spacer.

## Plate Studio pattern
Use when constrained typography is central to the physical object.

Typical needs:
- fixed or preset backing dimensions
- text/registration input
- font and spacing
- auto-fit within safe bounds
- raised/recessed characters
- characters-only / plate / separate-parts output
- aligned multi-color geometry

Examples: number plate, house number, desk plate, machine label.

## SignCraft pattern
Use when users compose several editable elements.

Typical needs:
- 2D physical workspace
- layers/objects
- backing profiles
- text, icons, SVG and QR
- move/scale/rotate/alignment/snapping
- templates
- extrusion/engrave/cut modes
- multi-part export

Examples: signs, plaques, WiFi QR signs, warning labels, directional signs.

## Hybrid selection examples
- Keychain: Parts + Plate + optional SignCraft.
- Cookie cutter: Parts + SVG composition.
- Stamp: Plate typography + SVG + parametric base/handle.
- Cable label: Parts dimensions + Plate typography.
- Lithophane frame: Parts dimensions + image-specific generator logic.
- Trophy/name plaque: Plate + SignCraft.
