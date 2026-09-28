---
name: contrast-guardrails
description: Use BEFORE any edit to a template/view that adds or changes text color, background color, or a UI-kit component like a badge, ghost button, muted text element, or modal. Codifies the specific low-contrast failure modes that keep shipping to production — white text on light-tint badge, white text on white background, muted-gray placeholder rendered where the value should be, colored-on-colored flash messages, ghost buttons that become invisible on light backgrounds, dark-theme utility classes (`bg-*/5`, `text-white/60`, `border-white/10`) copy-pasted into components that render on a LIGHT app shell, and dark-on-dark pills in dark mode. If you are about to write `text-white`, `text-zinc-500`, a UI-kit badge/text component, `variant="ghost"`, any `text-*-500` on a light background, or `bg-*/5` / `text-white/*` / `border-white/*` anywhere in a light-themed component, invoke this skill first. Also use when the user reports "I can't see the text", "invisible until I select all", "white on white", "gray on white", "green on green", "no border on the input", "washed out modal", "apply contrast skill", or shows a screenshot where text is missing.
---

# Contrast Guardrails

A pre-flight skill for any UI edit that touches color. Every change must satisfy WCAG AA at minimum (4.5:1 for body text, 3:1 for UI elements and large text). This skill exists because the same three-or-four failure modes keep shipping to production. Each one is documented below with the specific class combinations that fail, why they fail, and the fix pattern.

Written with Tailwind CSS class names as the concrete vocabulary, but the failure modes and diagnostics apply to any utility-CSS system (UnoCSS, Panda, Windi, or hand-rolled design tokens with the same shade scale).

---

## Failure mode 1 — UI-kit badges default to white text on light tint

**Symptom:** A component-library badge like `<Badge color="yellow">Head Admin</Badge>` renders as an INVISIBLE badge (light yellow background, near-white text). Same for `color="blue"`, `color="red"`, `color="green"`. User can only see the text after selecting-all with Ctrl+A.

**Root cause:** Many component libraries (Flux, Chakra, NextUI, some Radix presets, etc.) pick their badge foreground from a token that was designed for DARK app shells. On a light app, the badge text ends up near-white while the badge background is a light tint of the same color — zero contrast.

**Diagnostic (grep-time):**
```bash
grep -rn '<Badge\|flux:badge\|<Chip\|<Tag' src/ views/ resources/ 2>/dev/null | grep -v 'class='
```
Any badge without an explicit `class="..."` (or `className`) override is at risk.

**Fix pattern:** Do NOT use the library `<Badge>` on light-themed pages. Use a plain `<span>` with explicit utilities:
```html
<span class="inline-flex items-center rounded-md bg-amber-100 px-2 py-0.5 text-xs font-semibold text-amber-900 ring-1 ring-amber-200">
    Head Admin
</span>
```

Rule of thumb: `bg-{color}-100` + `text-{color}-900` + `ring-{color}-200` is always safe. The 900 shade on a 100 background always passes AA.

---

## Failure mode 2 — Framework inputs render typed content as muted gray

**Symptom:** User types into an input/textarea and can barely read what they typed. The text is a very light gray, the same shade as the placeholder.

**Root cause:** Many form kits set the input color to a muted token (roughly `zinc-500`) so that dozens of inputs don't compete for attention on a dashboard. That default is wrong for wizard-style or high-attention inputs where the input IS the primary action.

**Diagnostic (grep-time):**
```bash
grep -rn '<Input\|<Textarea\|flux:input\|flux:textarea' src/ resources/views/ 2>/dev/null
```

**Fix pattern:** For any wizard-style or high-attention input, use a hand-rolled `<input>` / `<textarea>` instead of the library component:
```html
<input type="text"
    placeholder="e.g. Handmade leather bags"
    class="block w-full rounded-lg border border-zinc-300 bg-white text-zinc-900 placeholder:text-zinc-400 px-3.5 py-2.5 text-sm shadow-sm focus:outline-none focus:border-violet-500 focus:ring-2 focus:ring-violet-100 transition"
/>
```

Key ingredients:
- `text-zinc-900` for typed content (never zinc-500, zinc-600, or zinc-700 — they all read as "muted/disabled")
- `placeholder:text-zinc-400` so hint text is visually distinct from typed content
- `border-zinc-300` at rest (many kits only show a border on focus by default — that misleads users about where they can type)
- `focus:ring-2 focus:ring-{brand}-100` for a visible focus ring

---

## Failure mode 3 — `variant="ghost"` on light backgrounds = invisible button

**Symptom:** A `<Button variant="ghost">Permissions</Button>` renders as invisible text on white. Users don't know a button is there.

**Root cause:** Ghost variant = no background, no border, just text at reduced opacity of the theme accent. On a white page, the button visually disappears.

**Diagnostic (grep-time):**
```bash
grep -rn 'variant="ghost"' src/ resources/views/
```

**Fix pattern:** Use `variant="outline"` for secondary buttons on light backgrounds. Ghost is only appropriate INSIDE dark surfaces (nav bars, dark cards, etc):
```html
<Button variant="outline" size="sm" icon="pencil">Permissions</Button>
```

If you truly need a no-border secondary CTA on a light bg, use an explicit outline pattern with a real border:
```html
<button class="rounded-lg border border-zinc-300 bg-white px-3 py-1.5 text-sm font-medium text-zinc-700 hover:bg-zinc-50 transition">
    Cancel
</button>
```

---

## Failure mode 4 — Library `<Text>` inside a tinted alert loses its color

**Symptom:** A success/error flash message: `bg-green-50` container with a library `<Text class="text-green-700">...</Text>` inside. Text appears WASHED OUT green-on-green because the `<Text>` component overrides the class-level color with its own muted default.

**Root cause:** Typography components frequently apply an internal `text-{muted}` on top of the `class`/`className` you pass. On low-opacity backgrounds the two blend into unreadability.

**Fix pattern:** Use a plain `<p>` instead of the library text component inside tinted containers:
```html
<div class="mb-6 rounded-lg bg-emerald-50 border border-emerald-200 p-4">
    <p class="text-sm font-medium text-emerald-900">Admin created</p>
</div>
```

Always use the `-900` text shade with a `-50` background. Never `text-{color}-500` or `text-{color}-600` on `bg-{color}-50` — contrast lands around 2.5–3.0, failing AA.

---

## Failure mode 5 — White text on gradient with low-opacity color tint

**Symptom:** Card with `bg-gradient-to-br from-indigo-500/10 via-violet-500/10 to-fuchsia-500/10` + `text-white` copy. Text becomes near-invisible because a 10% opacity gradient over a white page background is still ~90% white.

**Root cause:** Copying a "committed color" pattern without understanding that the opacity modifier means the background is mostly the page background. White text needs a background at least ~50% saturated to have real contrast.

**Diagnostic:**
```bash
grep -rn 'text-white' src/ resources/views/ | xargs grep -l 'bg-.*/1[05]' 2>/dev/null
```

**Fix pattern:** Two options:
1. **Solid saturated background:**
   ```html
   <div class="bg-gradient-to-br from-indigo-600 via-violet-600 to-fuchsia-600 p-5 text-white">
       ...
   </div>
   ```
2. **Keep the tint, switch text to dark:**
   ```html
   <div class="bg-indigo-500/10 border border-indigo-500/30 p-5 text-indigo-900">
       ...
   </div>
   ```

---

## Failure mode 6 — Dark text on dark-mode background (dynamic color pills)

**Symptom:** Pills like `<span class="bg-{$color}-100 dark:bg-{$color}-900/30 text-zinc-900">` render fine in light mode, but the dark-mode variant only overrides the background, leaving `text-zinc-900` (near-black) on a dark colored background = unreadable.

**Root cause:** Developers remember to add `dark:bg-*` for the surface but forget the paired `dark:text-*`. Result: dark-on-dark specifically in dark mode.

**Diagnostic:**
```bash
grep -rn 'dark:bg-' src/ resources/views/ | grep -v 'dark:text-\|dark:border-'
```
Any line with `dark:bg-*` that does not also mention `dark:text-*` on nearby text is at risk.

**Fix pattern:** Every `dark:bg-*` MUST be paired with `dark:text-*`. For dynamic pills, follow the light-100 → dark-900 mirror pattern:
```html
<span class="inline-flex items-center rounded-md
             bg-{{ $color }}-100 dark:bg-{{ $color }}-900/40
             text-{{ $color }}-900 dark:text-{{ $color }}-100
             ring-1 ring-{{ $color }}-200 dark:ring-{{ $color }}-800/50
             px-2 py-0.5 text-xs font-medium capitalize">
    {{ $label }}
</span>
```
For dynamic Tailwind classes, ensure the color families are in your `tailwind.config.js` `safelist` so all needed shades get compiled.

---

## Failure mode 7 — Gray-on-gray (table headers, muted text in dark mode)

**Symptom:** `<thead class="bg-zinc-50 dark:bg-zinc-800/50 text-zinc-500">` — column labels look fine in light mode, but in dark mode `text-zinc-500` on `bg-zinc-800` is gray-on-slightly-lighter-gray. Same problem for `text-zinc-400` as tertiary text on dark panels.

**Root cause:** `text-zinc-500` is a middle-gray that reads on white (contrast ~5:1) but fails on dark backgrounds (contrast ~2:1). `text-zinc-400` is worse — nearly invisible against `zinc-800`.

**Diagnostic:**
```bash
grep -rn 'text-zinc-500\|text-zinc-400' src/ resources/views/
```
Every match is suspicious. Especially: tables, panel headers, meta text below primary content, and empty states.

**Fix pattern:** For any secondary/tertiary text that renders in both light and dark themes, use paired variants:
```
Primary text:    text-zinc-900 dark:text-zinc-100
Secondary text:  text-zinc-700 dark:text-zinc-200   (was text-zinc-500)
Tertiary text:   text-zinc-600 dark:text-zinc-300   (was text-zinc-400)
Table header:    text-zinc-700 dark:text-zinc-200
```
Never use `text-zinc-500` or below without a `dark:` variant on dark surfaces. On white-only surfaces, `text-zinc-500` is acceptable for placeholder text only — never for real content.

---

## Failure mode 8 — Dark-theme classes surviving inside a light-theme modal

**Symptom:** A modal renders with:
- An instruction box that looks like a light-blue outlined block with almost-invisible bluish-white body text (`bg-blue-500/5` + `text-blue-200`)
- Amber "Note:" paragraph near-invisible on the light bg (`text-amber-200/90`)
- A "Cancel" button that is just floating text with no visible border (`variant="ghost"`)
- A footer border you can barely see (`border-zinc-200/10`)
- Typed input text that is grayer than the placeholder

User reports "washed out modal", "can barely read the instructions", or pastes the modal source and asks you to "apply contrast skill here".

**Root cause:** The modal was originally written when the app had a DARK shell. On dark bg:
- `bg-blue-500/5` reads as faint blue on ink — fine
- `text-blue-200` reads as pale blue on dark — legible
- `border-white/10` is a faint white separator on ink — visible
- `variant="ghost"` shows white text on ink — a real button

When the app switched to a LIGHT shell, none of these classes got updated. On white/cream:
- `bg-blue-500/5` = ~95% page bg, still page bg
- `text-blue-200` = pastel blue = washes out
- `border-white/10` = 90% invisible
- `variant="ghost"` = text with no bg, no border, no ring = invisible button

The trap: modal components look like they will adapt to the app theme. They do not. Whatever classes you passed at build time render literally.

**Diagnostic:**
```bash
grep -rEn 'bg-[a-z]+-500/[0-9]{1,2}|text-(white|[a-z]+-100|[a-z]+-200)|border-white/[0-9]{1,2}|variant="ghost"' \
    src/ resources/views/ | grep -iE 'modal|dialog|drawer|panel'
```

**Fix pattern — the "always-safe" swaps:**

| Was (dark theme) | Now (light theme) |
|---|---|
| `bg-blue-500/5` + `text-blue-200` | `bg-blue-50 border-blue-200 text-blue-900` |
| `bg-amber-500/10` + `text-amber-200` | `bg-amber-50 border-amber-200 text-amber-900` |
| `text-white/60` (body copy) | `text-zinc-700` (light) `dark:text-zinc-200` (dark) |
| `text-white/40` (tertiary) | `text-zinc-600` (light) |
| `border-white/10` | `border-zinc-200 dark:border-zinc-700` |
| `variant="ghost"` (secondary button) | `<button class="rounded-lg border border-zinc-300 bg-white px-4 py-2 text-sm font-medium text-zinc-900 hover:bg-zinc-50">` |
| Library `<Input>` (typed text muted) | Hand-rolled `<input>` — see Failure mode 2 pattern |
| Library `<Textarea>` | Hand-rolled `<textarea>` with same focus classes |

For dual-theme support, ALWAYS pair a `dark:*` variant with any surface class:
```html
<div class="rounded-lg border border-blue-200 bg-blue-50
            dark:border-blue-800/60 dark:bg-blue-900/20
            text-blue-900 dark:text-blue-100">
```

---

## Pre-flight checks before writing UI

Before opening any file to add/edit color:

1. **Read the parent's background color.** Inside a card at `bg-white`? Plan around dark text. Inside a `bg-zinc-900` panel? Plan around white text.
2. **Never write `text-white` without checking the resolved background at that specific point in the DOM.** Opacity modifiers on the parent are frequently the trap.
3. **Grep for every library component you're about to use** — badges, tinted `<Text>`, ghost buttons on light bg. If any match, prefer the plain HTML equivalent.
4. **When in doubt, use `-900` on `-100`.** This combo always passes AA.
5. **For inputs and textareas**, always add `text-zinc-900` + `placeholder:text-zinc-400` explicitly. Never rely on the library default.

---

## Always-safe palette combinations

These pairings are all verified AA on a neutral light theme:

| Text | Background | Ring/Border | Use case |
|------|-----------|-------------|----------|
| `text-zinc-900` | `bg-white` | `border-zinc-300` | body text, inputs, cards |
| `text-zinc-700` | `bg-zinc-50` | `border-zinc-200` | secondary text, subtle sections |
| `text-violet-900` | `bg-violet-100` | `ring-violet-200` | primary badges, chips |
| `text-emerald-900` | `bg-emerald-50` | `border-emerald-200` | success flash |
| `text-red-900` | `bg-red-50` | `border-red-200` | error flash |
| `text-amber-900` | `bg-amber-100` | `ring-amber-200` | warning badges |
| `text-blue-900` | `bg-blue-100` | `ring-blue-200` | info chips, tags |
| `text-white` | `bg-{brand}-600` | (none) | primary CTAs, active nav |
| `text-white` | `bg-gradient-to-br from-indigo-600 to-fuchsia-600` | (none) | hero cards |

Introducing a color not in this table is a new visual pattern the app doesn't have. Every new color you introduce is a maintenance cost.

---

## Adapt to your stack

The class names above are Tailwind. If your project uses a different system, translate the shade concept (e.g., `-50`/`-100`/`-900`) to your token scale rather than skipping the guardrail. The failure modes (dark-theme classes on light shell, ghost buttons on white, muted input text) exist in every design system that supports both themes.
