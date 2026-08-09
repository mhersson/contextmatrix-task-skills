---
name: frontend-design
description: Use when building a standalone page, prototype, or landing/marketing surface that has no existing design system - greenfield work where you choose the palette, fonts, and layout language yourself. Do NOT use when adding to or editing an app that already defines design tokens, a palette, or font tokens (any repo with a theme CSS file or a frontend instruction file); use typescript-react for that work.
---

This skill guides creation of distinctive, production-grade frontend interfaces
that avoid generic "AI slop" aesthetics. Implement real working code with
exceptional attention to aesthetic details and creative choices.

The user provides frontend requirements: a component, page, application, or
interface to build. They may include context about the purpose, audience, or
technical constraints.

## Prerequisite: no existing design system

Check before designing anything. If the surface already has a design system - a
token file of CSS custom properties, a palette or theme provider, declared font
tokens, or an instruction file in the frontend directory (`web/CLAUDE.md`,
`AGENTS.md`) - this skill does not apply. Stop, read that instruction file,
follow the system it describes, and use `typescript-react` for the
implementation. Inside an existing system never add a font, a hex literal, a
palette, a theme mechanism, or a styling approach the project does not already
use.

Inventory before you design, even on greenfield work: list what already exists
nearby, and do not build a near-duplicate of a component that is already there.

## Design Thinking

Before coding, understand the context and commit to a BOLD aesthetic direction:

- **Purpose**: What problem does this interface solve? Who uses it?
- **Tone**: Pick an extreme: brutally minimal, maximalist chaos,
  retro-futuristic, organic/natural, luxury/refined, playful/toy-like,
  editorial/magazine, brutalist/raw, art deco/geometric, soft/pastel,
  industrial/utilitarian, etc. There are so many flavors to choose from. Use
  these for inspiration but design one that is true to the aesthetic direction.
- **Constraints**: Technical requirements (framework, performance,
  accessibility).
- **Differentiation**: What makes this UNFORGETTABLE? What's the one thing
  someone will remember?

**CRITICAL**: Choose a clear conceptual direction and execute it with precision.
Bold maximalism and refined minimalism both work - the key is intentionality,
not intensity.

Then implement working code (HTML/CSS/JS, React, Vue, etc.) that is:

- Production-grade and functional
- Visually striking and memorable
- Cohesive with a clear aesthetic point-of-view
- Meticulously refined in every detail

## Frontend Aesthetics Guidelines

Focus on:

- **Typography** (only when the project ships no font tokens of its own): Choose fonts that are beautiful, unique, and interesting.
  Avoid generic fonts like Arial and Inter; opt instead for distinctive choices
  that elevate the frontend's aesthetics; unexpected, characterful font choices.
  Pair a distinctive display font with a refined body font.
- **Color & Theme**: Commit to a cohesive aesthetic. Use CSS variables for
  consistency. Dominant colors with sharp accents outperform timid,
  evenly-distributed palettes.
- **Motion**: Use animations for effects and micro-interactions. Prioritize
  CSS-only solutions for HTML. Use Motion library for React when available.
  Focus on high-impact moments: one well-orchestrated page load with staggered
  reveals (animation-delay) creates more delight than scattered
  micro-interactions. Use scroll-triggering and hover states that surprise.
- **Spatial Composition**: Unexpected layouts. Asymmetry. Overlap. Diagonal
  flow. Grid-breaking elements. Generous negative space OR controlled density.
- **Backgrounds & Visual Details**: Create atmosphere and depth rather than
  defaulting to solid colors. Add contextual effects and textures that match the
  overall aesthetic. Apply creative forms like gradient meshes, noise textures,
  geometric patterns, layered transparencies, dramatic shadows, decorative
  borders, custom cursors, and grain overlays.

NEVER use generic AI-generated aesthetics like overused font families (Inter,
Roboto, Arial, system fonts), cliched color schemes (particularly purple
gradients on white backgrounds), predictable layouts and component patterns, and
cookie-cutter design that lacks context-specific character.

Interpret creatively and make unexpected choices that feel genuinely designed
for the context. When you own the visual direction, no design should be the
same: vary between light and dark themes, different fonts, different aesthetics,
and never converge on common choices (Space Grotesk, for example) across
generations. When the project already ships font tokens or a theme, none of this
applies - use its font tokens and colour variables as-is, and make every
component work in BOTH light and dark, because the theme is the user's runtime
toggle, not a design decision.

**IMPORTANT**: Match implementation complexity to the aesthetic vision.
Maximalist designs need elaborate code with extensive animations and effects.
Minimalist or refined designs need restraint, precision, and careful attention
to spacing, typography, and subtle details. Elegance comes from executing the
vision well.

Remember: Claude is capable of extraordinary creative work. Don't hold back,
show what can truly be created when thinking outside the box and committing
fully to a distinctive vision.

## Quality floor

Whatever the aesthetic, these are checkable and non-negotiable:

- Visible keyboard focus on every interactive element (`:focus-visible`, never `outline: none` without a replacement).
- Honour `prefers-reduced-motion`; gate every non-essential animation behind it.
- Never `transition: all`. Name the properties.
- Flex and grid children that hold text get `min-w-0` so long strings wrap instead of overflowing.
- Numbers that update in place use tabular figures so they do not jitter.
- Empty, loading, and error states are designed, not left blank.
- Words are design material: control labels say what happens, error text says what to do next, and one term means one thing throughout.
