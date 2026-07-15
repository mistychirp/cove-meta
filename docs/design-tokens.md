# Cove — Design Token Contract (draft v0.1)

> A companion to `phase-0-contract.md`. That document makes many *clients*
> possible; this one makes many *themes* possible. Same instinct: name a stable
> seam, then let anyone plug into it.

## 0. The core idea

Material 3 is two things bundled together:

1. A **colour system** — HCT, tonal palettes, every role generated from a seed.
2. **Everything else** — shape, motion, typography, state layers, elevation.

Cove keeps (2) wholesale and demotes (1) from *the* source of colour to *a*
source of colour.

The reason is concrete. Feed a hand-authored palette such as Tokyo Night into
M3's dynamic-colour generator as a seed and what comes out is not Tokyo Night:
HCT requantises the hues and imposes its own tonal steps, so you get "M3's idea
of a blueish dark theme". The character people love is exactly what gets lost.
A hand-authored palette must therefore be able to reach the components
*directly*, without passing through the generator.

Hence: **the token layer is the contract, and colour sources are pluggable.**

```
  Tokyo Night (static map) ─┐
  M3 dynamic (seed → HCT)  ─┼─→  --cove-* tokens  ─→  components
  DMS (wallpaper → HCT)    ─┘
```

No source is privileged. Components only ever read `--cove-*` and know nothing
about where the values came from.

**Why the tokens are not named `--md-sys-color-*`:** that naming would make M3's
role set the contract, and a hand palette would spend its life pretending to be
an M3 output. Cove's own vocabulary keeps every source on equal footing. The
cost is that off-the-shelf `material-web` components do not drop in — acceptable,
since Cove hand-rolls its components anyway.

## 1. Rules

1. **Components read tokens only.** A hard-coded colour in a component is a bug:
   it is invisible to every theme. Two deliberate exceptions, both marked in the
   CSS: the video letterbox and the modal scrim are black in every scheme (M3
   defines its scrim the same way), so making each palette restate them would be
   busywork, not flexibility.
2. **Every token in §2 is mandatory.** A theme that omits one is incomplete;
   there is no inheritance fallback to guess from.
3. **Derive states, never assume tonal distance.** Hover/pressed/selected come
   from `color-mix()` on the token itself. M3's dynamic colour guarantees fixed
   tonal steps between roles; a hand palette does not, so anything relying on
   those steps breaks on half the themes.
4. **A theme is colour only.** Shape, motion and type tokens belong to the
   design language, not to the palette. Themes do not override them.
5. Palettes are dumb data. Anything a theme cannot express as the §2 token set
   is a gap in the contract — widen the contract, don't special-case the theme.
6. **Every token has a consumer.** The set is deliberately minimal: a token that
   nothing reads is a promise nobody can check, and theme authors would be told
   to fill in a value whose mistakes are invisible. Mentions, warnings and a
   filled danger button will bring `--cove-mention`, `--cove-warning` and
   `--cove-on-danger` back when they actually land — three lines per palette.

## 2. Colour tokens

### Surfaces — a ladder from furthest to nearest
| Token | Use |
|-------|-----|
| `--cove-bg` | app background, the furthest plane |
| `--cove-surface` | panels sitting on the background: sidebar, header, composer |
| `--cove-surface-raised` | lifted above the panel: menus, modals, hovered rows |
| `--cove-surface-sunken` | recessed into the panel: input wells, code blocks |

### Content
| Token | Use |
|-------|-----|
| `--cove-text` | primary text |
| `--cove-text-muted` | secondary: timestamps, metadata |
| `--cove-text-faint` | tertiary: placeholders, disabled |

### Accent
| Token | Use |
|-------|-----|
| `--cove-accent` | primary action, brand |
| `--cove-on-accent` | text/icons on top of `--cove-accent` |
| `--cove-accent-soft` | accent at low emphasis: selected row, active channel |

### Semantic
| Token | Use |
|-------|-----|
| `--cove-danger` | destructive action, errors |
| `--cove-success` | online presence, confirmation |
| `--cove-link` | links inside message content |

### Structure
| Token | Use |
|-------|-----|
| `--cove-outline` | visible borders |
| `--cove-outline-faint` | subtle dividers |

## 3. Shape, motion, type, elevation

Owned by the design language (M3 Expressive), **not** by themes.

```
/* Shape */
--cove-radius-xs  4px
--cove-radius-sm  8px
--cove-radius-md  12px
--cove-radius-lg  20px
--cove-radius-xl  28px
--cove-radius-full 999px

/* Motion — M3 Expressive leans on spring physics */
--cove-motion-fast      150ms
--cove-motion-standard  250ms
--cove-motion-slow      400ms
--cove-ease-standard    cubic-bezier(0.2, 0, 0, 1)
--cove-ease-emphasized  cubic-bezier(0.2, 0, 0, 1)
--cove-ease-spring      linear(...)   /* spring curve, see styles.css */

/* Type */
--cove-font       UI stack
--cove-font-mono  code and code blocks

/* Type scale — five named steps. Sizes had drifted to nine ad-hoc values with
   no system, which is exactly what a scale exists to prevent. */
--cove-type-label     11px   uppercase headers, timestamps
--cove-type-small     12px   hints, metadata, captions
--cove-type-body      15px   messages and default text
--cove-type-title     18px   modal and section titles
--cove-type-headline  24px   the login screen

/* Elevation */
--cove-shadow-2 / --cove-shadow-3
```

Two sizes are deliberately outside the scale, both marked in the CSS: `0.9em`
for inline code and `1.1em` for markdown headings scale *relative to the message
they sit in, and `16px` on the chip's × sizes a glyph as an icon, not as text.

## 3a. Density — the second axis

This is where M3 Expressive and a terminal aesthetic genuinely pull apart: M3
wants air, the Tokyo Night crowd wants rows on screen. Rather than pick a winner
it is a user-selectable axis, **independent of the palette** — every theme works
at every density.

| Token | comfy | compact |
|-------|-------|---------|
| `--cove-gap` | 12px | 8px |
| `--cove-row-pad` | 8px | 4px |
| `--cove-msg-gap` | 10px | 5px |
| `--cove-pane-pad` | 16px | 10px |

Applied as `[data-density='compact' \| 'comfy']` on the root, exactly like a
palette. Density is *not* theme-owned: a theme must never set these.

## 4. How a theme is applied

Two mechanisms, deliberately layered so they compose:

- **Built-in palettes** are plain CSS under `[data-theme="<id>"]`. No JS, no
  flash, works before React mounts. The selector is not anchored to `:root` on
  purpose: any element can opt into a palette, which is how the picker renders a
  live swatch of a theme that is not currently active.
- **Generated palettes** (M3 seed, DMS) are written by JS as inline properties on
  `document.documentElement`. Inline style beats any selector, so a generated
  palette naturally overrides the built-in default.

Every theme declares `color-scheme: light | dark` so native controls,
scrollbars and form widgets follow along.

## 5. Adding a theme

A theme is the §2 token set plus a mode. Nothing else:

```css
:root[data-theme="tokyo-night"] {
  color-scheme: dark;
  --cove-bg: #16161e;
  --cove-surface: #1a1b26;
  /* … every token from §2 … */
}
```

Register it in `src/themes/index.ts` so the picker can list it. That is the
whole procedure — a theme is roughly twenty lines of data, never a fork.
