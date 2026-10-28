---
name: blog-banner
description: Generate an abstract, minimal, pastel banner image for a blog post on Thiyagaraj's Hugo blog (thiyaga.xyz). Trigger when the user asks for a blog post banner/cover image, "generate an image for this post", or /blog-banner.
---

# Blog Banner Generator

Generate a cover/banner image for a blog post in Thiyagaraj's established style: **very abstract, very minimal** — a pastel background with 2–10 simple geometric shapes (circle, square, triangle, line, arc, bar) that form a subtle visual metaphor for the post's title or core idea.

## Style rules (non-negotiable)

- **Canvas:** 1200×630 PNG (OpenGraph size). Draw at 2× (2400×1260) and downscale with LANCZOS for antialiasing.
- **Background:** one flat, bright pastel, chosen loosely to fit the post's mood. Each post gets a *different* colour so the listing page looks varied. Palette in use (2026-10): sky `#A9D6FF`, rose `#FFBFC9`, lavender `#D4C4FF`, peach `#FFCBA0`, yellow `#FFE27A`, pink `#FFB4D9`, mint `#A8EBC9`, aqua `#9EE5E0`. Pick a new tone in the same saturation range; avoid the washed-out pastels used before.
- **Two files per post:** the PNG (for og:image / link previews) and an SVG twin at `static/images/<slug>-banner.svg` with the same shapes, inlined on the site and animated only on card hover (one quick, sub-second, single-run motion under `:is(.post-entry,.home-tile,.entry-cover):hover`, scoped by a unique class prefix). Frontmatter sets both `cover.image` and `cover.animated`.
- **Shapes:** dark grey `(58, 58, 58)` only. Filled vs. outlined is a meaningful contrast (e.g. filled = solid/working, outlined = hollow/failed/moving). For "faded/ghost" elements, blend the dark grey toward the background color, never use a second hue.
- **Composition:** generous padding — shapes occupy roughly the middle 50–60% of the canvas; nothing near the edges. Fewer shapes is better. No text, no gradients, no icons, no drop shadows.
- **Metaphor:** the shapes should encode ONE idea from the title, readable in a second. Examples from existing banners:
  - *Fast AI or Slow AI* → circle outline with fading ghost trails + speed dashes (fast) vs. a static filled square (slow).
  - *90/10 rule* → a row of 9 filled squares + 1 hollow circle.
  - *Structure in time (RNN)* → a row of filled circles with the last hollow (next prediction) and an arc looping back (recurrence).
  - *Speculative routing* → three parallel streams, two faded stopping short, the winner passing through a triangle (router) to a filled dot.
  - *Reliable voice agents* → a voice waveform of rounded bars, one hollow (failure) but the wave continues.

## Workflow

1. Read the post (usually in `content/posts/`) and pick the single strongest metaphor from the title/thesis. Propose shapes mentally; keep it to one idea.
2. Write a small Python + PIL script (PIL is installed; no ImageMagick/rsvg on this machine). Template below. **Run it from the blog repo root** — it writes to `static/images/<slug>-banner.png` by relative path.
3. View the generated PNG with the Read tool to check composition and padding. Iterate if shapes crowd the edges or the metaphor muddles.
4. Add/update the post frontmatter:
   ```yaml
   cover:
     image: "/images/<slug>-banner.png"
     alt: "<one-line literal description of the shapes>"
     caption: "<one-line tie back to the post's idea>"
   ```
5. Build with `/opt/homebrew/bin/hugo --quiet` (NOT `./hugo` — that's an empty stub file in the repo). The deployable `public/` output is committed to git along with content and static files.
6. Only push if the user asks.

## Script template

```python
from PIL import Image, ImageDraw

W, H, S = 1200, 630, 2          # canvas + 2x supersample
DARK = (58, 58, 58)
BG = (r, g, b)                   # pastel of choice

img = Image.new("RGB", (W * S, H * S), BG)
d = ImageDraw.Draw(img)

def blend(c1, c2, t):            # for ghost/faded shapes
    return tuple(round(a + (b - a) * t) for a, b in zip(c1, c2))

# ... draw shapes, all coordinates multiplied by S ...
# strokes ~9-11 * S; keep shapes in the middle ~55% of the canvas

img.resize((W, H), Image.LANCZOS).save("static/images/<slug>-banner.png", "PNG")
```

Useful PIL calls: `d.ellipse(box, fill=/outline=, width=)`, `d.rectangle`, `d.rounded_rectangle(box, radius=)`, `d.polygon(points, outline=, width=)` for triangles, `d.arc(box, start=, end=, width=)` for loops, `d.line(..., width=)` for streams/dashes.
