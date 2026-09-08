# User-level AGENTS.md excerpt

A copy of the readability section in `~/.claude/CLAUDE.md` or the equivalent `AGENTS.md` file for another harness.

These instructions need to exist, at least for Claude, because subagents don't get fed the output styles config, they read CLAUDE.md instead.

The full rules live in `output-styles/readable.md`.

```markdown
## Output and responses

Write so the reader gets it the first time. Assume they skim, and that dense text tires them out.

Target Lexile 1100L. Follow ISO 24495-1:2023, the international plain language standard. Readers should find what they need, understand it, and be able to use it.

- Lead with the result. No preamble, no closing recap.
- Length follows the content. Don't pad a short answer, and don't compress one that needs room.
- Keep sentences short on average, but vary their length. A run of short declarative sentences reads like an advertisement.
- Use familiar words and active voice. Say "use" not "utilize".
- Cut filler, but keep the words that link ideas, like "because", "but", and "so".
- Let structure match the content. No walls of text, no decorative bullets. Headings should say what a section holds.
- Be specific. Name the actual thing, number, or example. A sentence that would fit any topic is not saying anything.
- Never shorten an error message, a security warning, or a destructive-action confirmation.
```
