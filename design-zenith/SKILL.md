---
name: design-zenith
description: "Auto-triggers on 'design', 'poster', 'diagram', 'slide', 'social graphic',
             'brand', 'flowchart', 'deck', 'brochure', 'create a design', 'make a poster',
             'design a slide', 'architecture diagram', 'social media post', 'client deliverable'.
             Orchestrates Zenith (.zen) for deterministic design creation."
---

# Design Zenith — Agent-Native Design Orchestration
*Create deterministic, version-controlled designs via Zenith's .zen format*

## Activation

When this skill activates, determine what the user wants to design and execute the matching workflow.

## Context Guard

| Context | Status |
|---------|--------|
| **User describes a design need** | ACTIVE — determine template + workflow |
| **User says "design X" / "poster" / "diagram"** | ACTIVE — enter design workflow |
| **User says "render" / "preview"** | ACTIVE — render current .zen file |
| **User says "validate"** | ACTIVE — validate current .zen file |
| **Mid-conversation (no design request)** | DORMANT |

---

## Prerequisites

- Zenith MCP server configured (`@zenitheditor/zenith-mcp` in Claude Code settings)
- Or: Zenith binary installed (`cargo install zenith-tool`)
- `.zen` files are KDL syntax — human-readable, diffable, validatable

---

## Design Workflow

### Standard Flow
```
Brief → Select template → Scaffold .zen → Build layout → Validate → Render → Iterate → Deliver
```

### Step 1: Understand the Brief
- [ ] Ask: What is being designed? (poster, slide, diagram, social post, flyer)
- [ ] Ask: What are the dimensions? (1080x1080 social, 1920x1080 slide, A4 print, etc.)
- [ ] Ask: What brand/style? (colors, fonts, mood)
- [ ] Ask: What content? (text, images, data)

### Step 2: Select or Create Template
Choose based on request type:

| Request | Template | Dimensions |
|---------|----------|------------|
| Social media post | `social-square` | 1080×1080 |
| Presentation slide | `presentation-slide` | 1920×1080 |
| Architecture diagram | `architecture-diagram` | 1920×1080 |
| Client flyer | `client-flyer` | 595×842 (A4) |
| Banner | `banner` | 1200×630 |
| Custom | Ask dimensions | User-specified |

### Step 3: Scaffold .zen File
- [ ] Use `zenith new` or write from template
- [ ] Define tokens first (colors, fonts, dimensions)
- [ ] Build page structure (background → content → overlay)
- [ ] Every node needs a unique `id` attribute

### Step 4: Build Layout Incrementally
- [ ] Add nodes one at a time
- [ ] Validate after each significant change: `zenith validate`
- [ ] Use `zenith inspect` to discover node IDs
- [ ] Use `zenith tokens` to verify token resolution

### Step 5: Render and Iterate
- [ ] Render preview: `zenith render doc.zen --png out.png` (or `--pdf out.pdf`)
- [ ] Review output
- [ ] Edit via transactions: `zenith tx doc.zen edits.json` (dry-run first), then `--apply`
- [ ] Re-render until satisfied

### Step 6: Generate Variants (if needed)
- [ ] Document must contain a `variants { variant id=… source="page.main" w=… h=… }` block
- [ ] `zenith variant doc.zen --out-dir out/` — one `.zen` + `.png` per variant
- [ ] Common: square 1080×1080 → banner 1200×630 → story 1080×1920

### Step 7: Deliver
- [ ] Final render to PNG/PDF
- [ ] Save .zen file for future edits (version-controlled)
- [ ] If client deliverable: copy to client folder

---

## Named Workflows

### 1. Client Deliverable Workflow
*brief → .zen → validate → render → iterate → variant → deliver*

```
Brief → .zen scaffold → validate → render preview → iterate tx →
workspace_candidate → workspace_promote → variant → final render → deliver
```

**Steps:**
1. **Brief** — gather: purpose, dimensions, brand colors, copy, deadline
2. **Scaffold** — `zenith new client-[name].zen` using `client-flyer` template
3. **Populate** — fill in copy, swap brand tokens, place visual elements
4. **Validate** — `zenith_validate` (MCP) or `zenith validate` (CLI) — fix all hard errors
5. **Render preview** — `zenith_render format="png"` — review composition
6. **Iterate** — use `zenith_tx` (dry-run first) for targeted edits; re-render
7. **Lock candidate** — `zenith_workspace_candidate` to save approved iteration
8. **Promote** — `zenith_workspace_promote` when client approves
9. **Variants** — `zenith variant` for size/format variants (A4, social-square, banner)
10. **Final render** — `zenith_render format="pdf"` for print-ready delivery
11. **Deliver** — copy final PNG/PDF to `clients/<name>/deliverables/`; commit .zen source

**Key rule**: Never deliver without validating first. Commit the .zen file — it's the source of truth.

---

### 2. Documentation Workflow
*architecture description → .zen flowchart → render → embed*

```
Describe architecture → scaffold architecture-diagram.zen →
add boxes/labels/arrows → validate → render PNG → embed in docs
```

**Steps:**
1. **Describe** — list all components, their relationships, and flow direction
2. **Scaffold** — `zenith new arch-[topic].zen` using `architecture-diagram` template
3. **Map components** — add one `rect` + `text` node pair per component
4. **Add connectors** — use `line` nodes with `stroke` for arrows between boxes
5. **Use mono font** — `font.mono` token for all labels (readability in diagrams)
6. **Validate** — `zenith_validate` before rendering
7. **Render** — `zenith_render format="png" scale=2` for retina-quality output
8. **Embed** — copy PNG to `docs/assets/` or repo root; link in README: `![arch](docs/assets/arch-[topic].png)`
9. **Commit both** — `.zen` source + rendered PNG (PNG for viewers, .zen for edits)

**Key rule**: Commit both the .zen file and the rendered PNG. The .zen is editable; the PNG is embeddable.

---

### 3. Social Content Workflow
*brand kit → .zen template → merge (CSV) → publish*

```
Load brand tokens → social-square template → fill copy →
bulk: zenith_merge CSV → render all → review → publish
```

**Steps (single post):**
1. **Brand tokens** — ensure `tokens` block uses brand colors/fonts (use `zenith_theme_new` from brand hex if needed)
2. **Template** — scaffold from `social-square` template; parameterize text nodes
3. **Fill copy** — populate heading, body, CTA via `zenith_tx`
4. **Validate + render** — `zenith_validate` → `zenith_render format="png"`
5. **Publish** — copy to `clients/<name>/social/` or social content output folder

**Steps (bulk — CSV merge):**
1. **Prepare CSV** — columns: `heading`, `body`, `filename` (one row per post; header names = data fields)
2. **Parameterize template** — mark text nodes with `role="data.<column>"` (e.g. `role="data.heading"`); do **not** use `{{mustache}}` placeholders
3. **Merge** — `zenith merge social.zen posts.csv --out-dir renders/ --name-by filename`
4. **Review** — scan all rendered PNGs; flag any layout issues
5. **Publish** — distribute to scheduler or copy to client folder

**Key rule**: Always review merged renders before bulk publish — CSV typos don't fail validation.

---

## Schema Rules (hard errors if violated)

| Rule | Correct | Wrong |
|------|---------|-------|
| Visual props must be tokens | `stroke-width=(token)"size.stroke"` | `stroke-width=(px)2` |
| Rounded corners | `radius=(token)"size.radius"` | `rx=(px)8` (unknown property) |
| Text edits via tx | `{"op":"replace_text","node":"h","spans":[{"text":"Hi"}]}` | `"text":"Hi"` field (invalid) |
| Render flags | `zenith render d.zen --png out.png` | `--format png` |
| Tx apply | `zenith tx d.zen edits.json --apply` | `zenith tx d.zen --ops '[]'` |
| Paths from WSL | Prefer native WSL `zenith` binary; Win `.exe` needs `C:/Users/...` paths | `/mnt/c/...` with `zenith.exe` fails (os error 2/3) |
| Contrast | Dark bg + light text (APCA Lc ≥ 45) | Dark navy + mid blue subtitle → `contrast.low` warning |
| Unused tokens | Reference every token (or drop it) | Defined-but-unused → advisory only |

---

### social-square
```kdl
zenith version=1 {
  project id="proj.social" name="Social Square"
  tokens format="zenith-token-v1" {
    token id="color.bg" type="color" value="#ffffff"
    token id="color.ink" type="color" value="#111827"
    token id="color.accent" type="color" value="#2563eb"
    token id="font.heading" type="fontFamily" value="Noto Sans"
    token id="font.body" type="fontFamily" value="Noto Sans"
    token id="size.heading" type="dimension" value=(px)48
    token id="size.body" type="dimension" value=(px)24
  }
  document id="doc.social" title="Social Post" {
    page id="page.main" w=(px)1080 h=(px)1080 {
      rect id="bg" x=(px)0 y=(px)0 w=(px)1080 h=(px)1080 fill=(token)"color.bg"
      rect id="accent-bar" x=(px)64 y=(px)180 w=(px)120 h=(px)8 fill=(token)"color.accent"
      text id="heading" x=(px)64 y=(px)64 w=(px)952 h=(px)100 fill=(token)"color.ink" font-family=(token)"font.heading" font-size=(token)"size.heading" { span "Heading" }
      text id="body" x=(px)64 y=(px)220 w=(px)952 h=(px)600 fill=(token)"color.ink" font-family=(token)"font.body" font-size=(token)"size.body" { span "Body text goes here." }
    }
  }
}
```

### presentation-slide
```kdl
zenith version=1 {
  project id="proj.slide" name="Presentation Slide"
  tokens format="zenith-token-v1" {
    token id="color.bg" type="color" value="#0f172a"
    token id="color.ink" type="color" value="#f8fafc"
    token id="color.accent" type="color" value="#93c5fd"
    token id="font.heading" type="fontFamily" value="Noto Sans"
    token id="font.body" type="fontFamily" value="Noto Sans"
    token id="size.title" type="dimension" value=(px)64
    token id="size.subtitle" type="dimension" value=(px)32
  }
  document id="doc.slide" title="Slide" {
    page id="page.main" w=(px)1920 h=(px)1080 {
      rect id="bg" x=(px)0 y=(px)0 w=(px)1920 h=(px)1080 fill=(token)"color.bg"
      text id="title" x=(px)120 y=(px)120 w=(px)1680 h=(px)160 fill=(token)"color.ink" font-family=(token)"font.heading" font-size=(token)"size.title" { span "Title" }
      text id="subtitle" x=(px)120 y=(px)320 w=(px)1680 h=(px)80 fill=(token)"color.accent" font-family=(token)"font.body" font-size=(token)"size.subtitle" { span "Subtitle" }
    }
  }
}
```

### architecture-diagram
```kdl
zenith version=1 {
  project id="proj.arch" name="Architecture Diagram"
  tokens format="zenith-token-v1" {
    token id="color.bg" type="color" value="#f8fafc"
    token id="color.box" type="color" value="#e2e8f0"
    token id="color.ink" type="color" value="#1e293b"
    token id="color.line" type="color" value="#64748b"
    token id="font.mono" type="fontFamily" value="Noto Sans Mono"
    token id="size.label" type="dimension" value=(px)18
    token id="size.stroke" type="dimension" value=(px)2
    token id="size.radius" type="dimension" value=(px)8
  }
  document id="doc.arch" title="Architecture" {
    page id="page.main" w=(px)1920 h=(px)1080 {
      rect id="bg" x=(px)0 y=(px)0 w=(px)1920 h=(px)1080 fill=(token)"color.bg"
      rect id="box1" x=(px)100 y=(px)100 w=(px)300 h=(px)120 radius=(token)"size.radius" fill=(token)"color.box" stroke=(token)"color.line" stroke-width=(token)"size.stroke"
      text id="label1" x=(px)120 y=(px)140 w=(px)260 h=(px)40 fill=(token)"color.ink" font-family=(token)"font.mono" font-size=(token)"size.label" { span "Service A" }
      rect id="box2" x=(px)500 y=(px)100 w=(px)300 h=(px)120 radius=(token)"size.radius" fill=(token)"color.box" stroke=(token)"color.line" stroke-width=(token)"size.stroke"
      text id="label2" x=(px)520 y=(px)140 w=(px)260 h=(px)40 fill=(token)"color.ink" font-family=(token)"font.mono" font-size=(token)"size.label" { span "Service B" }
      line id="conn1" x1=(px)400 y1=(px)160 x2=(px)500 y2=(px)160 stroke=(token)"color.line" stroke-width=(token)"size.stroke"
    }
  }
}
```

### client-flyer (A4)
```kdl
zenith version=1 {
  project id="proj.flyer" name="Client Flyer"
  tokens format="zenith-token-v1" {
    token id="color.bg" type="color" value="#ffffff"
    token id="color.ink" type="color" value="#0f172a"
    token id="color.accent" type="color" value="#059669"
    token id="font.heading" type="fontFamily" value="Noto Sans"
    token id="font.body" type="fontFamily" value="Noto Sans"
    token id="size.heading" type="dimension" value=(px)36
    token id="size.body" type="dimension" value=(px)16
  }
  document id="doc.flyer" title="Client Flyer" {
    page id="page.main" w=(px)595 h=(px)842 {
      rect id="bg" x=(px)0 y=(px)0 w=(px)595 h=(px)842 fill=(token)"color.bg"
      rect id="accent-bar" x=(px)0 y=(px)0 w=(px)595 h=(px)12 fill=(token)"color.accent"
      text id="heading" x=(px)40 y=(px)40 w=(px)515 h=(px)60 fill=(token)"color.ink" font-family=(token)"font.heading" font-size=(token)"size.heading" { span "Client Proposal" }
      text id="body" x=(px)40 y=(px)120 w=(px)515 h=(px)400 fill=(token)"color.ink" font-family=(token)"font.body" font-size=(token)"size.body" { span "Professional deliverable copy goes here." }
    }
  }
}
```

---

## MCP Tools Reference

When MCP is available, use these tools:

| Tool | When to Use |
|------|-------------|
| `zenith_schema` | Learn node kinds, attributes, tx ops on demand |
| `zenith_fonts` | List available fonts (bundled vs local) |
| `zenith_validate` | Check .zen for errors before rendering |
| `zenith_inspect` | Discover node IDs and structure |
| `zenith_tokens` | List all design tokens and resolved values |
| `zenith_tx` | Apply typed edit (dry-run first!) |
| `zenith_render` | Render to PNG/PDF/scene |
| `zenith_fmt` | Canonicalize document formatting |
| `zenith_merge` | Mail-merge template with CSV |
| `zenith_theme_new` | Generate theme pack from brand colors |
| `zenith_workspace_*` | Manage scratch candidates and iterations |

---

## CLI Fallback

If MCP is unavailable, use CLI directly:

```bash
zenith new my-design.zen                              # Scaffold
zenith validate my-design.zen                         # Check errors (exit 0 = no hard errors)
zenith inspect my-design.zen                          # See node tree
zenith tokens my-design.zen                           # List tokens
zenith fmt my-design.zen                              # Format file in place
zenith render my-design.zen --png out.png             # Render PNG
zenith render my-design.zen --pdf out.pdf             # Render PDF
zenith tx my-design.zen edits.json                    # Dry-run transaction
zenith tx my-design.zen edits.json --apply            # Apply transaction
zenith theme new brand --scheme light --primary '#2563eb' --out brand.zen
zenith variant my-design.zen --out-dir variants/      # Requires variants{} block
zenith merge tpl.zen data.csv --out-dir out/ --name-by filename
zenith workspace scratch new my-design.zen            # Record candidate
zenith version my-design.zen "checkpoint-name"        # Named version
zenith schema                                         # Node kinds / ops / tokens
zenith fonts                                          # Available fonts
```

**edits.json example** (replace text):
```json
{"ops":[{"op":"replace_text","node":"heading","spans":[{"text":"New Heading"}]}]}
```

---

## Integration with Other Skills

| Skill | Integration |
|-------|-------------|
| **image-prompt** | Fallback for AI-generated images (non-deterministic) |
| **taste-skill** | Quality gate — review design before delivery |
| **copywriting** | Generate copy for social posts, flyers |
| **brand** | Define brand tokens for consistent designs |

---

## Mandatory Rules

1. **Always validate before render** — `zenith validate` blocks on hard errors
2. **Dry-run transactions first** — `zenith tx` defaults to dry-run; inspect before applying
3. **Every node needs a unique id** — required for transaction targeting
4. **Tokens first** — define colors, fonts, dimensions as tokens before using them
5. **Version control .zen files** — they're plain text, commit them
6. **Deterministic output** — same .zen + same backend = same bytes (trust it)

---

## Level History

- **Lv.1** — Base: design workflow (7 steps), 4 templates (social-square, presentation-slide, architecture-diagram, client-flyer), MCP tools reference, CLI fallback, integration with other skills. (Origin: Zenith integration into Ame — 2026-06-28)
- **Lv.2** — Added 3 named workflows: client-deliverable, documentation, social-content (single + bulk CSV merge). (2026-06-29)
- **Lv.3** — Smoke-validated against zenith 0.0.6/0.0.8 (2026-07-10): fixed architecture template (`radius` + tokenized `stroke-width`, not `rx`/raw px); presentation accent for APCA contrast; social accent-bar; client-flyer template; schema rules table; CLI flags (`--png`/`--pdf`, tx JSON file, `role="data.col"` merge, variants block). Full suite outputs: `Downloads/design-zenith-smoketest/`.
