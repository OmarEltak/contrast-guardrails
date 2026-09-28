# contrast-guardrails

A [Claude Code Skill](https://docs.anthropic.com/en/docs/claude-code/skills) that runs a pre-flight color-contrast review before any UI edit — so your agent stops shipping the same "invisible button", "white-on-white text", "washed-out modal" bugs to production.

Codifies **eight specific low-contrast failure modes** that recur across component-library-heavy codebases (Flux, Chakra, Radix presets, NextUI, and hand-rolled Tailwind), with the exact class combinations that fail, why they fail, and the copy-paste fix.

## When it triggers

Claude will auto-invoke this skill when:

- You are about to write `text-white`, `text-zinc-500`, any `text-*-500` on a light background, `bg-*/5`, `text-white/*`, or `border-white/*` in a light-themed component
- You are about to use a UI-kit badge, ghost button, muted text component, or modal
- The user reports symptoms like *"I can't see the text"*, *"invisible until I select all"*, *"white on white"*, *"gray on white"*, *"green on green"*, *"washed out modal"*, or pastes a screenshot with missing text
- The user explicitly says *"apply contrast skill"*

## The eight failure modes covered

1. **UI-kit badges default to white text on light tint** (`<Badge color="yellow">` = invisible)
2. **Framework inputs render typed content as muted gray** (typed value indistinguishable from placeholder)
3. **`variant="ghost"` on light backgrounds = invisible button**
4. **Library `<Text>` inside a tinted alert loses its color** (green-on-green flash messages)
5. **White text on gradient with low-opacity color tint** (`bg-*/10` + `text-white`)
6. **Dark text on dark-mode background** (missing paired `dark:text-*`)
7. **Gray-on-gray** (`text-zinc-500` on `bg-zinc-800`)
8. **Dark-theme classes surviving inside a light-theme modal** (the full "washed-out modal" pattern)

Each failure mode ships with a grep-time diagnostic, root-cause explanation, and safe fix pattern.

## Install

Drop it into your project or your global Claude Code skills directory:

```bash
# project-scoped
mkdir -p .claude/skills/contrast-guardrails
curl -o .claude/skills/contrast-guardrails/SKILL.md \
  https://raw.githubusercontent.com/OmarEltak/contrast-guardrails/main/SKILL.md

# or globally for all projects
mkdir -p ~/.claude/skills/contrast-guardrails
curl -o ~/.claude/skills/contrast-guardrails/SKILL.md \
  https://raw.githubusercontent.com/OmarEltak/contrast-guardrails/main/SKILL.md
```

Restart Claude Code (or run `/skills` to reload). No config required.

## Works with

- Any agent that reads Claude Code Skills (Claude Code CLI, VS Code / JetBrains Claude extension, Copilot CLI's `skill` tool, Gemini CLI's `activate_skill`)
- Any UI stack using utility-CSS: **Tailwind**, UnoCSS, Panda, Windi, or hand-rolled tokens on the same shade scale
- Component libraries mentioned by name: **Flux (Livewire)**, **Chakra**, **NextUI**, **Radix presets** — the failure modes exist in all of them

## Adapting to your stack

The class names in the skill are Tailwind. If your project uses a different token system, translate the shade concept (`-50`/`-100`/`-900`) to your scale — the failure modes themselves (dark-theme classes on light shell, ghost buttons on white, muted input text) are universal.

## License

MIT — see [LICENSE](./LICENSE).
