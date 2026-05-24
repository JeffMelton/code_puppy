# Color Accessibility

> Code Puppy must never override the user's terminal color palette.
> ANSI named colors only. Hardcoded hex palettes are an **accessibility
> regression**, not a stylistic preference.

This document explains *why* that rule exists, *what* it means in
practice, and *how* to check your work against it. Read this before
adding any color literal — `color="#..."`, `style="bold blue"`, a
`RenderStyle` field, a Pygments token map — anywhere in the codebase.

The operative principle is **palette-respect**: surrender color choice
to the terminal so accessibility tooling (high-contrast themes,
color-blind-safe palettes, screen-reader-paired environments) actually
reaches the user. Everything below justifies and operationalizes that
rule.

---

## Why This Is Accessibility, Not Preference

It is tempting to frame "respect the user's terminal palette" as a
matter of taste: *the user curated their iTerm theme and we should
honor it.* That framing is true but downstream. The actual driver is
accessibility, and hardcoded palettes break it in three concrete,
measurable ways:

### 1. Contrast failure (WCAG 2.2 Level AA)

Monokai-style palettes — Code Puppy's historical default — assume a
dark background. Computed contrast ratios for the currently-hardcoded
token palette in `code_puppy/tools/common.py`:

| Token color (hex)                  | vs white | AA-normal (4.5:1) | AA-large (3:1) | vs black | AA-normal |
|------------------------------------|---------:|:-----------------:|:--------------:|---------:|:---------:|
| `#e6db74` (pale yellow, strings)   |   1.42:1 | ❌                | ❌             |  14.75:1 | ✅        |
| `#75715e` (olive, comments)        |   4.91:1 | ✅                | ✅             |   4.28:1 | ❌        |
| `#a6e22e` (lime, function names)   |   1.55:1 | ❌                | ❌             |  13.54:1 | ✅        |
| `#66d9ef` (pale cyan, builtins)    |   1.65:1 | ❌                | ❌             |  12.74:1 | ✅        |
| `#f92672` (hot pink, keywords)     |   3.79:1 | ❌                | ✅             |   5.55:1 | ✅        |
| `#ae81ff` (pale plum, numbers)     |   2.84:1 | ❌                | ❌             |   7.38:1 | ✅        |

And the termflow `RenderStyle` chrome palette:

| Chrome color (hex)                  | vs white | AA-normal | AA-large | vs black | AA-normal |
|-------------------------------------|---------:|:---------:|:--------:|---------:|:---------:|
| `#dda0dd` (plum, borders & bullets) |   2.07:1 | ❌        | ❌       |  10.15:1 | ✅        |
| `#87ceeb` (sky blue, H1/H2)         |   1.74:1 | ❌        | ❌       |  12.06:1 | ✅        |
| `#98fb98` (pale green, H3)          |   1.27:1 | ❌        | ❌       |  16.59:1 | ✅        |
| `#6699cc` (muted blue, links)       |   3.00:1 | ❌        | ✅       |   6.99:1 | ✅        |

Ratios computed per the WCAG 2.2 relative-luminance definition
([W3C: Relative luminance][rl], [W3C: Contrast ratio][cr]).

Two things jump out:

- **Almost every value passes against black and fails against white.**
  That asymmetry is not coincidence — it is the defining property of a
  palette designed against a dark background. The internal consistency
  Monokai achieves on dark terminals is precisely what *hides* the
  catastrophic failure on light terminals: nothing inside the palette
  warns us, because every color is fine relative to every other color.
- **The chrome palette fails too**, not just the syntax tokens. The
  plum border (`#dda0dd`) is mid-luminance enough to remain *visible*
  against a white background even while failing AA — borders register
  at 2.07:1 contrast while the code-block contents inside them fail
  catastrophically. Visibility is not legibility, and a palette that
  lets users see *where* code blocks are while preventing them from
  reading *what is in them* is arguably a worse failure mode than
  total invisibility, not a partial success.

Normally-sighted users on light terminals see washed-out or invisible
text across both token and chrome surfaces. This is the failure mode
that surfaced the original bug (`white-on-white in light terminals`).

[rl]: https://www.w3.org/TR/WCAG22/#dfn-relative-luminance
[cr]: https://www.w3.org/TR/WCAG22/#dfn-contrast-ratio

### 2. Color-blind unusability

Even on a background where the palette *passes* contrast, the colors
chosen for token differentiation collapse under common color-vision
deficiencies:

- **Deuteranopia / protanopia (red–green):** Hot-pink keywords
  (`#f92672`) and lime function names (`#a6e22e`) flatten into
  near-identical mid-grey. Keyword vs identifier vs callable becomes
  indistinguishable.
- **Tritanopia (blue–yellow):** The pale-cyan-builtin
  (`#66d9ef`) / pale-yellow-string (`#e6db74`) pairing in diff
  highlights collapses similarly.

As many as **8% of men and 0.5% of women of Northern European
ancestry** have inherited red–green color-vision deficiency — the most
common form — per the [NIH National Eye Institute][nei]. A
code-reading tool that renders keywords and identifiers as the same
shade for that population is not usable, regardless of how well the
same palette renders for trichromatic users.

[nei]: https://www.nei.nih.gov/learn-about-eye-health/eye-conditions-and-diseases/color-blindness/types-color-blindness

### 3. Assistive-tech bypass

Users who have configured high-contrast terminal themes, color-blind
safe palettes, low-vision tuning profiles, or screen-reader-aware
color schemes have done so **deliberately** as accessibility
mitigations. Hardcoded hex strings in our renderers do not just ignore
those choices — they actively override them. The user takes a
deliberate accessibility action; we silently undo it. That is the
bypass we are eliminating.

---

## The ANSI Named-Colors Approach

ANSI named colors (`ansiblue`, `ansigreen`, `ansimagenta`,
`ansibrightblack`, `ansiyellow`, etc.) are not specific RGB values.
They are **named slots** in the terminal's 16-color palette. When we
emit `ansiblue`, the terminal renders whatever RGB the user has bound
to that slot — which is the only surface where the user's contrast
tuning and color-blind-safe substitutions can take effect.

This makes ANSI named colors the right primitive on all three fronts
above:

- **Contrast:** the terminal palette is where users (and OS-level
  high-contrast modes) define contrast-safe color pairs. Named colors
  inherit that work for free.
- **Color blindness:** color-blind-safe terminal themes (e.g.,
  Wezterm's `OneHalfDark-CBSafe`, iTerm color-blind presets) remap the
  named slots to distinguishable hues. Named colors get this for free
  too.
- **Assistive tech:** screen-reader-paired terminals, high-contrast OS
  modes, and per-environment overrides all operate at the named-slot
  layer. Using named colors means we participate in those systems
  instead of fighting them.

**Limits worth being honest about:**

- Named colors only cover 16 slots. Token palettes that want fine-grain
  distinctions (e.g., separate hues for `Name.Function`,
  `Name.Builtin`, `Name.Class`) need to either accept the 16-slot
  ceiling or fall back to luminance-detected presets — *as a last
  resort, and only when terminal_theme cannot resolve a palette signal
  at all.*
- Some terminals (rare in 2026, but present) ignore OSC 11
  background-color queries. For these, COLORFGBG and explicit user
  config remain the resolution path. See `dsj.1`.
- True-color hexes are not banned outright — they remain valid for
  cases where the user has *explicitly opted in via config*, mirroring
  the existing pattern for diff addition/deletion colors
  (`config:get_diff_addition_color` etc.). The principle is: never
  hardcode them as defaults.

---

## Acceptance Criteria

For every fix that touches a render surface in this codebase:

1. **Palette-override check.** Does Code Puppy override the user's
   terminal palette in *any* theme — light, dark, or custom? If yes,
   not done.
2. **Contrast check (WCAG 2.2 Level AA).** Does the output meet the
   4.5:1 contrast floor for normal text and 3:1 for large text, when
   rendered through the resolved named-ANSI palette against both a
   white and black background? If not measurable, not done.
3. **Color-blind distinguishability.** Is the output distinguishable
   under deuteranopia, protanopia, and tritanopia simulation? Verify
   with a simulator (macOS Sim Daltonism, browser extensions like
   `Let's Get Color Blind`, or equivalent). If two semantically
   distinct tokens collapse to the same hue under any of the three,
   not done.

All three must pass. A fix that satisfies palette-respect but ships a
two-token-collision under deuteranopia is not done; a fix that passes
color-blind checks but only because it hardcodes a "safe" palette is
not done.

---

## Surfaces This Principle Applies To

Code Puppy currently has four independent render surfaces, each with
its own historical palette choices. The principle applies uniformly to
all four.

| Surface                                       | Current state                          | Target state                                     | Tracking |
|-----------------------------------------------|----------------------------------------|--------------------------------------------------|----------|
| Rich `Markdown()` callsites                   | Hardcoded Monokai default              | Route through `current_code_theme()`             | `dsj.2`  |
| termflow streaming-path `Highlighter`         | Pygments `TerminalTrueColorFormatter` or `Terminal256Formatter`, no ANSI-name support | Subclass or upstream — 8-color ANSI formatter when theme is `ansi_*` | `dsj.4`  |
| termflow `RenderStyle` (markdown chrome)      | Hardcoded hex defaults (plum borders, sky-blue headings, pale-green H3, etc.)  | Construct from terminal palette using ANSI named colors | `dsj.3`  |
| Rich-renderer diff token highlighting         | Hardcoded `TOKEN_COLORS` Monokai palette in `tools/common.py` | Route through `current_code_theme()` / terminal_theme to ANSI names | `dsj.5`  |

Plus one foundational dependency that enables all of the above:

| Foundation                                    | Current state                          | Target state                                     | Tracking |
|-----------------------------------------------|----------------------------------------|--------------------------------------------------|----------|
| Terminal background detection                 | `COLORFGBG` env var only; many modern terminals don't set it | OSC 11 query as a fallback before any heuristic | `dsj.1`  |

The `code_puppy/messaging/terminal_theme.py` module is the canonical
resolution path. It returns `current_code_theme()` based on the
detected (or user-configured) terminal background, and any render
surface that emits colors should route through it.

---

## Non-Goals

This document is not, and the codebase should not become:

- **A theme engine.** We are not in the business of generating,
  curating, or shipping color schemes. The terminal already is one.
- **A bundler of light/dark presets.** Presets are themes. Themes are
  the user's job. The principled escape hatch when terminal_theme
  cannot resolve a palette signal at all is *named ANSI colors with
  graceful degradation* — not a Code Puppy preset library.
- **A regression on existing user-config surfaces.** Diff addition
  and deletion colors (`get_diff_addition_color`,
  `get_diff_deletion_color`) already route through user config
  correctly. They are exemplary, not in scope for change.

---

## Enforcement

This document is a principle, not a fence. Prose in `docs/` does not
stop a contributor from adding `color="#dda0dd"` to a new component
six months from now, and the cost of that regression is borne by
users we cannot see on terminals we have not tested.

The contributor-facing checklist ("before you add a color literal,
stop and...") and the enforcement layers that need to sit alongside
it — pre-commit hex-literal grep, ruff custom rule, CI contrast
regression check, or some combination — are out of scope for this
principle doc and are being scoped separately. See the RFC discussion
linked from the dsj epic for the current proposal.

The short version: be paranoid about color literals. That paranoia
should eventually be machine-enforced, not vibes-enforced.
