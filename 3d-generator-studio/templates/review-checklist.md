# Generator Review Checklist

Review the implementation against `SKILL.md`.

## Product
- Does a useful default object appear immediately?
- Can a normal user understand what to change without CAD knowledge?
- Are common controls separated from advanced controls?

## Dimensions and geometry
- Are physical dimensions correct and consistently in mm?
- Are derived dimensions correct?
- Are invalid combinations handled?
- Are booleans, holes, text and SVG geometry correct?
- Is the mesh slicer-friendly/manifold where possible?
- Are tolerances explicit rather than hidden?

## Preview
- Does live regeneration work reliably?
- Does camera fit work after major dimension changes?
- Are orbit/pan/zoom usable?
- Is geometry readable in light and dark themes?

## Composition/text, if applicable
- Does text become actual geometry?
- Does auto-fit behave predictably?
- Do layers, alignment and snapping work?
- Are uploaded fonts/SVGs handled locally where practical?

## Multi-part, if applicable
- Do exported parts preserve a common origin and align automatically?
- Are combined/separate output modes correct?

## Export
- Does STL work at correct scale?
- Do all other advertised formats genuinely work?
- Are filenames descriptive and sanitized?
- Is invalid geometry prevented from export?

## UX/layout
- Does desktop fit the browser without unnecessary page scrolling?
- Do side panels scroll internally where needed?
- Is mobile usable rather than merely shrunk?
- Do System, Light and Dark themes work?

## Regression
- Do existing generators still work?
- Are existing exports unchanged unless intentionally modified?
- Were unnecessary dependencies or duplicate geometry/rendering stacks avoided?
