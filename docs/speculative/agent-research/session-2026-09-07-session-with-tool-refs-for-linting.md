<user>
Search the web for answers. I'm finding that agent writing is really full of AI-isms difficult to read. Fatiguing. How are people solving this?
</user>
<agent>
Three approaches have converged, and people mostly stack them.

**1. Deterministic linters, run before the model self-edits.** `unsloppify` is a CLI prose linter for markdown that splits the work into tiers: punctuation (em dashes, smart quotes) auto-fixed, banned phrases flagged or auto-fixed, and structural clichés left to a judgment pass. Its argument against pure-skill approaches is that asking Claude to scan its own prose costs tokens and misses obvious cases. `slop-cop` takes the same tack as a Go CLI that emits JSON for agent consumption, with a plugin so the agent lints its own draft and silently revises before replying. These plug into pre-commit hooks or CI.

**2. Rules files loaded at generation time.** `agent-style` installs a technical-prose ruleset into CLAUDE.md, AGENTS.md, copilot-instructions, Cursor rules, and so on. Its own benchmark is the useful part: across PR descriptions, commit messages, and design-doc sections, Claude Opus dropped 47% in violations with the ruleset loaded, GPT-5.4 via Codex 46%, Gemini Flash 86%, and Copilot CLI showed no change at all. So roughly half the tics go away, not all of them.

**3. Post-hoc rewrite skills.** `avoid-ai-writing` runs in detect, rewrite, or in-place edit modes, with a second pass to catch patterns that survived the first. Worth noting its own caveat: the flagged patterns are statistically more common in LLM output, but humans under deadline produce the same shapes, so it's a writing-quality tool rather than a verdict.

The judgment layer that seems to matter most: treating prose cleanup as a pass separate from idea development, and targeting plain workmanlike prose rather than literary flourish. And if you want your own voice rather than generic de-slopped voice, people build a profile from a corpus of their actual writing. One project derived sentence-length variance, claim-then-proof paragraph structure, and vocabulary habits from 513,683 words of source material.

For your setup, the cheapest thing to try first is a Claude Code output style, since it modifies the system prompt directly and is meant for exactly the case where you keep re-prompting for the same voice every turn.
</agent>
<user>
What about for response formatting or output style? What you suggested is helpful, but it's for documentation writing
</user>
<agent>
For turn-level responses rather than documents, the mechanism is different: output styles, which inject a dedicated section into the system prompt rather than sitting in a user message like CLAUDE.md does.

**Start with the built-in.** Claude Code v2.1.237 shipped a `Concise` style: leads with the result, skips preamble and narration, keeps responses short, while doing the engineering work as thoroughly as Default. If you ask for detail you still get it in full, and it always preserves complete error reports, security warnings, and destructive-action confirmations. `/config` → Output style. One gotcha people hit constantly: the style is read at session start, so switching mid-session does nothing until `/clear`.

**If you want your own rules**, a custom style is a markdown file in `.claude/output-styles/` with frontmatter. A representative one:

```
---
name: terse
description: Answers only. Minimal prose, no unrequested explanation or preamble.
keep-coding-instructions: true
---
## Response rules
- Default to 1-3 lines. One-word answers are great if they are true.
- No preamble, no postamble, no summary of what you just did.
## When to say more
- A direct request: "why", "explain", "walk me through", "in detail"
```

`keep-coding-instructions: true` matters. Without it, custom styles drop Claude Code's built-in engineering instructions about scoping changes, writing comments, and verifying work.

**Why not the alternatives**, per one comparison: CLAUDE.md isn't purpose-built for response style and the directive gets diluted on long threads; skills load only when invoked or judged relevant, so there's no always-on formatting layer; `--append-system-prompt` lands at low priority and busts your cache when the value changes. All add roughly the same input tokens, so the win is adherence, not cost.

Two caveats. Styles apply to the main conversation only, so subagents keep their own system prompt (forks are the exception). And there's no per-project output style setting yet, though writing `outputStyle` into `.claude/settings.json` at project root is the workaround people use, and it's an open request upstream.
</agent>
<user>
What about for the pi.dev harness
</user>
<agent>
Pi has no output-styles concept, but it has a cleaner equivalent, because the system prompt is a file you own rather than a menu setting.

**`APPEND_SYSTEM.md` is the one you want.** Put it in `.pi/` for a project or `~/.pi/agent/` globally, and it appends to the default prompt without replacing it. `SYSTEM.md` in the same two locations replaces the default outright. Replacement sounds appealing given Pi's sub-1000-token prompt, but there's a trap: a custom `SYSTEM.md` (or `--system-prompt`) receives some injected context but not the dynamic tools list or tool-specific guidelines the default prompt gets. Someone who tried both landed on `APPEND_SYSTEM.md` as best of both, keeping Pi's logic intact while layering their own adjustments on top. Same conclusion for your case: response formatting is additive, so append.

The pieces you'd carry over from an output style go in as plain markdown. Lead with the result, no preamble, no trailing recap of what was just done, no restating the request, target line counts, and an escape clause for when you actually ask for detail.

**What not to use.** `AGENTS.md` is loaded at startup from `~/.pi/agent/`, parent dirs, and cwd, and is meant for project conventions, commands, safety rules, and preferences. It works, but it's context-file layer rather than system-prompt layer, which is the same dilution problem output styles were built to avoid in Claude Code. Skills load on demand, so they're not an always-on formatting layer either. Worth knowing: `AGENTS.override.md` in a directory replaces `AGENTS.md`/`CLAUDE.md` for that directory while other dirs still layer normally, which is handy if you're sharing a repo with Claude Code users and want different rules.

**The Pi-specific option others don't have:** extensions can inject messages before each turn and filter message history. If append-only text drifts on long sessions, a small extension that re-injects the formatting reminder per turn is roughly what Claude Code does internally when it reminds the model of the active style. That's more work than you probably need, but it's there if the file alone doesn't hold.
</agent>
<user>
What are some prompts that people have found effective for making output more readable? For example:
> make this more readable.
> target Lexile level of 1000L
> consider the W3C Cognitive Accessibility Guidance

Search the web
</user>
<agent>
Research on this is mostly from patient-education and edtech, but the findings transfer. Four patterns, roughly in order of how much they buy you.

**1. The bare "make this more readable" prompt actually works.** LLMs have improved the readability of patient information and radiology reports with prompts that simple. Researchers deliberately study simple prompts because their simplicity is what makes independent use by non-experts realistic. So your first example isn't naive.

**2. Naming a target level beats naming none.** A plain-language prompt targeting a sixth-to-eighth-grade reading level dropped Flesch-Kincaid by about five grade levels and raised reading ease 30 to 38 points, with no measured loss of accuracy, relevance, or depth. Another study compared plain-text instructions against a prompt carrying an explicit readability metric across 2,000 texts, and the metric-guided version cut complexity significantly more.

**3. For Lexile specifically, define the scale in the prompt.** The published zero-shot template does this rather than assuming the model knows:

> A Lexile measure is the numeric representation of a text's readability, where lower scores reflect easier readability and higher scores indicate harder. Rewrite the text below into the target Lexile level, keeping the original meaning and information. Original draft (Lexile = {source}): [TEXT START] … [TEXT END] Rewrite to Lexile = {target}.

The few-shot version adds a worked source/target pair before the task, and repeats only the example block for more shots.

**4. Constraint density is the main lever.** Across five techniques (zero-shot, contextualized, constrained, meta, persona) on 2,655 outputs from twelve models: prompts with fewer constraints yielded less readable output, prompts with more constraints yielded significantly more readable output. No technique or model reliably hit a sixth-grade level by SMOG, though. Treat the target as a direction, not a guarantee, and verify with a checker.

**On your W3C example**, COGA is referenced far less often than WCAG in LLM accessibility work, and cognitive and neurodiverse needs stay underrepresented relative to screen-reader concerns. So the model may have thin grounding for "consider COGA guidance." Naming the specific criteria works better: 3.1.3 unusual words, 3.1.4 abbreviations, 2.4.6 headings that let readers predict section contents. The structural guidance is the part that matters most for agent output. Break content into chunks with whitespace rather than four or five consecutive dense paragraphs, since readers skim. Concrete word swaps too, like "use" instead of "utilize."

Two things that pair well with these, from the anti-slop side of your earlier question: run readability cleanup as a pass separate from the content itself, and get concrete rather than sweeping, since specific names and examples fix the eye-glazing paragraphs faster than simplification alone.
</agent>
