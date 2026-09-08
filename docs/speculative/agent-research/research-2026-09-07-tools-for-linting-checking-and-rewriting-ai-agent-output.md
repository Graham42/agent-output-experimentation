# The Prose De-Slopping Toolchain: Tools for Linting, Checking, and Rewriting AI Agent Output

## TL;DR
- **The category has bifurcated into two layers you should combine**: mature, deterministic prose linters (Vale is the clear anchor, with proselint, textlint, retext, alex, and LanguageTool around it) that catch mechanical tells and gate CI with machine-readable output; and a fast-moving crop of AI-slop-specific tools (slop-cop, the textlint AI-writing preset, Vale AI-tells packages, EQ-Bench's Slop Score, agent skills like unsloppify/SlopMonster) built since ~2024 specifically for em-dashes, "delve," and "it's not X, it's Y."
- **For a coding-agent workflow, the highest-leverage architecture is lint → LLM-rewrite → re-lint**, with a deterministic linter (Vale + an AI-tells style package) as the non-negotiable gate and an agent skill doing the rewriting; a cross-model rewrite (one model family cleans another's draft) measurably outperforms self-revision.
- **Recommended minimal stack**: Vale + the `tbhb/vale-ai-tells` or `JMill/deslop` style package for deterministic gating; `textstat` (Python) or the `papa` CLI for readability thresholds; and one agent skill (unsloppify or SlopMonster) wired as a pre-reply self-revision step. This is free, offline, MIT-licensed, and CI-friendly.

## Key Findings

1. **Vale is the de facto standard** for deterministic prose linting: markup-aware (12 formats via real parsers, skips code spans/URLs), Go binary with no runtime dependencies, ~6.1k stars (per github.com/vale-cli/vale), actively maintained (v3.13.1, Feb 2026, MIT), and developed solely by @jdkato, who describes himself as "the sole developer of Vale." It emits JSON (`--output=JSON`), integrates with every major editor via LSP, GitHub Actions, and pre-commit, and imports style packages (Microsoft, Google, write-good, proselint, alex ports).

2. **The AI-slop-specific tooling exploded in 2024-2026** but is mostly young and low-adoption. The most useful deterministic entrants are `slop-cop` (Go, agent-oriented JSON output), the `textlint-ja/textlint-rule-preset-ai-writing` preset (~1.1k stars, actively developed, now works as an MCP server), and Vale style packages for AI-isms (`tbhb/vale-ai-tells`, `JMill/deslop`, `Syntaf/vale-llm-slop`, `t0ddharris/slopster`).

3. **Readability scoring is a solved, commodity problem** in Python (`textstat`, `py-readability-metrics`) and JS, computing Flesch-Kincaid, Flesch Reading Ease, SMOG, Gunning Fog, Coleman-Liau, Dale-Chall, ARI. **Lexile is the exception** — it is proprietary to MetaMetrics, available only via a paid partner API; open tools use approximations correlated with public formulas.

4. **Plain-language / accessibility enforcement is thinner and more manual.** plainlanguage.gov provides guidance (not a linter); WCAG SC 3.1.3/3.1.4/2.4.6 are operationalized mostly by proprietary suites (Siteimprove) or new agent projects (`Community-Access/accessibility-agents`). Deterministic jargon/passive/heading checks come from retext plugins, Vale rules, and readability CLIs like `papa`.

5. **Hybrid pipelines (lint → LLM rewrite → re-lint) are the emerging best practice**, exemplified by SlopMonster (rival-model cleanse), slopster (Vale + agent skill + diff gate), and Sam Paech's Antislop framework (sequence-level sampler + FTPO fine-tuning). Research confirms metric-guided prompts beat plain "make it readable" instructions.

6. **Voice-matching / style-profile tools** exist commercially (WRITER's Voice, Noren) and as agent-skill patterns (voice-profile.md files fed to the model); research shows profile-based prompting beats RAG for style, but LLMs still imitate everyday authors imperfectly — the Stony Brook/Penn State study *Catch Me If You Can? Not Yet* (arXiv:2509.14543), spanning 40,000+ generations per model across 400+ authors, concludes that "while LLMs can approximate user styles in structured formats like news and email, they struggle with nuanced, informal writing in blogs and forums."

## Details

### Why this problem is real and getting worse

The "delve" phenomenon is now quantified. Kobak, González-Márquez, Horvát & Lause (*Science Advances* 2025;11(27):eadt3813), analyzing 15.1M PubMed abstracts, found "delves" had the strongest excess-frequency ratio (r = 28.0) of any word, ahead of "underscores" (r = 13.8) and "showcasing" (r = 10.7). The same paper estimates that **at least 13.5% of 2024 abstracts were processed with LLMs, reaching 40% for some subcorpora.** Undisclosed use compounds the signal: a 2025 Springer-Nature survey found around half of all LLM use was undisclosed, and more than three-quarters among early-career researchers and PhD students (self-reported, cited in arXiv:2512.01560). The tells are pervasive, statistically measurable, and largely invisible to the authors producing them — which is exactly why deterministic tooling helps.

### 1. Deterministic, general-purpose prose linters

**Vale** (`vale-cli/vale`, formerly errata-ai) — Go binary, ~6.1k stars, v3.13.1 (Feb 2026), MIT, maintained solely by @jdkato. Detects style/terminology/usage issues via configurable YAML rules (existence, occurrence, substitution, readability). Markup-aware across Markdown, reST, AsciiDoc, HTML, plus comments in 19 languages. Imports style packages: Microsoft Writing Style Guide, Google developer docs, write-good, proselint, alex, Joblint ports. **Machine-readable**: yes — JSON output plus line/level diagnostics, designed for CI. Integrations: VS Code/Cursor/Windsurf, Neovim, Sublime, Zed, Emacs, JetBrains, Obsidian; GitHub Actions (`vale-cli/vale-action`), pre-commit, LSP. There is a dedicated Pragmatic Bookshelf book (*Write Better with Vale*, Brian P. Hogan). **Limitations**: it lints existing prose only; regex-based rules can false-positive on product names (mitigated via vocabularies); single-maintainer risk.

**proselint** (`amperser/proselint`) — Python, ~4.5k stars, v0.16.0 (Nov 2025), BSD-3-Clause, ~3,000 PyPI downloads/week (Snyk flagged it "Inactive" mid-2025, but a release shipped Nov 2025). Aggregates rules from famous editors (Garner, DFW, Orwell, etc.) focused on usage/style, not grammar. **Machine-readable**: yes — `--output-format json` with a documented stable wire schema (check, message, source, line, column). pre-commit hook available. **Limitations**: opinionated, some checks vague; slower than write-good on large docs.

**write-good** (`btford/write-good`) — Node, ~5.1k stars, MIT, but **effectively unmaintained** (no published releases, open issues dating to 2017-2023, issue creation restricted). Naive checks for passive voice, weasel words, "so"-openers, clichés, adverbs. Returns JS objects (programmatic). Best consumed *through* Vale or textlint rather than standalone now.

**textlint** (`textlint/textlint`) — Node, ~3.1k stars, v15.5.0 (Dec 2025), MIT, requires Node 20+. The pluggable NL linter: nothing enabled by default; configure via `.textlintrc`. Huge rule/preset ecosystem, supports Markdown/plain text by default and other formats via plugins. Inline disable comments (`<!-- textlint-disable -->`). **Machine-readable**: yes — kernel API returns structured messages; JSON formatter; as of v14.8.0 runs as an **MCP server** for direct AI-tool integration. Can wrap write-good and proselint-style rules.

**alex** (`get-alex/alex`) — Node, ~5.1k stars, npm v11.0.1 (Aug 2026), MIT, by @wooorm (retext-based). Single-purpose: catches insensitive/inconsiderate language (gender, race, religion, ability). Built on retext-equality. VS Code extension, CLI, LSP-capable. **Limitations**: narrow scope by design; known false positives on legitimate technical terms.

**retext / unified / remark ecosystem** (`retextjs/retext`) — Node, ESM, MIT. Natural-language processing over concrete syntax trees; compose plugins: `retext-passive` (passive voice), `retext-simplify` (complex phrases), `retext-equality` (alex's engine), `retext-readability` (flags overly complex sentences via multiple formulas), `retext-intensify`/`retext-profanities`. `remark-retext` and `rehype-retext` bridge Markdown/HTML. **Machine-readable**: VFileMessage objects with source/ruleId/position; vfile-reporter for output. Best for engineers who want to build a custom, composable pipeline in Node.

**LanguageTool** (`languagetool-org/languagetool`) — Java, ~15k stars, LGPL 2.1+ core (open-core; premium is proprietary), 25+ languages, 2,000+ English rules. Self-hostable via Docker; REST API; Python wrapper (`language-tool-python`) runs a local JAR or hits the public API. Grammar + style + spelling, deeper than the prose linters but heavier. **Machine-readable**: yes — JSON REST responses. **Limitations**: JVM footprint, public API rate-limited (self-host to avoid).

**Redpen** (`redpen-cc/redpen`) — Java, ~597 stars, Apache-2.0, latest 1.10.4. **Largely dormant/abandoned** — commit activity is old (2017-2024), site still advertises v1.9.0. Was a multi-language document validator (sentence length, invalid symbols, etc.) with JSON output. Not recommended for new adoption.

**Newer general entrant**: `LintMe` (academic prototype, arXiv 2603.00331) combines deterministic and LLM-based README linting with 21 malleable operators — research-stage.

### 2. AI-slop-specific linters and detectors

This category is young (most tools <1 year old, low star counts) but directly targets the user's problem. Group by mechanism:

**Deterministic CLI / package linters:**
- **slop-cop** (`yasyf/slop-cop`) — Go, ~8 stars, MIT. Agent-first design: prints JSON on stdout, diagnostics on stderr, no TUI. Uses tree-sitter to mask non-prose bytes (flags only comments, string literals, JSX text). 57 rules across layers (slop, base, google); can invoke `claude` CLI to rewrite. Flags, doesn't auto-fix (rewrite is a separate opt-in command).
- **slop-lint** (`eric-sabe/slop-lint`, npm 0.8.0) — Node 18+, zero-dependency, single file. **Fails** on em-dash (treated as the near-decisive typographic tell), **warns** on words/clichés/constructions. Has a `--discover` mode that compares word/bigram frequency against a baseline to find new tells statistically (the method that surfaced "delve").
- **slop-gate** — zero-dependency CLI, em-dash + ~40 English tells plus translationese packs (Korean, Russian, Vietnamese, Chinese, Filipino).
- **blocklint** — simple banned-phrase linter, wrapped by the community MCP server alongside proselint/alex/write-good/textstat.
- **SlopSift** (`slopsift.dev`) — a custom-trained compact dependency parser (quantized ONNX, runs locally in Node/browser/VS Code), not a word list. Detects corrective-antithesis, unsupported-certainty, vague-attribution, AI-vocabulary via grammatical relationships. Errors vs. warnings; VS Code Problems-panel integration; agent skill available.

**textlint presets:**
- **`@textlint-ja/textlint-rule-preset-ai-writing`** — ~1.1k stars, v1.7.0 (May 2026), MIT, by azu. Despite the `-ja` org, it includes structure-based rules (bold-first list items, hype expressions, emphasis patterns, colon-continuation, list formatting) usable beyond Japanese. Optimized for the "AI writes → AI checks → AI improves" loop; runs via textlint's MCP server. Configurable per-rule with `allows` lists. Flags only.
- **slopless** (`preset-slopless` / `npx slopless`) — deterministic textlint preset + zero-config CLI, no API key; flags hollow framing, fake contrasts, hedging, em-dash tics, vacuous closers so an agent rewrites until clean.

**Vale style packages for AI-isms:**
- **`JMill/deslop`** — Vale package, MIT, v0.3.0 ships ~34 rule files; installs via `.vale.ini` release URL + `vale sync`; tested so clean technical prose produces zero alerts and no two rules overlap. Flags. (Note: "deslop" is a crowded name — the Vale one is `JMill/deslop`.)
- **`tbhb/vale-ai-tells`** — Vale package (self-parodying description). Rules derived from published taxonomies (Charlie Guo, Pangram Labs, Wikipedia Signs of AI Writing). Flags.
- **`Syntaf/vale-llm-slop`** — two styles: `Slop` (AI-sounding writing: comments echoing code, buzzwords like "robust"/"delve", fake enthusiasm) and `STE` (opt-in ASD-STE100 simplified-English discipline). 28 rules, 0 false positives on clean fixtures. Ships plain-English substitutions (the ASD dictionary is copyrighted).
- **`krishnasunkam/vale-ai-tells`** — 17 rules, 6 gate as errors (mechanical tells that should never ship), rest are suggestions; paired agent skill.
- **`t0ddharris/slopster`** — five Vale YAML rules + a "Tagore" agent skill (29-pattern catalog, 8-dimension scoring) + a `slop-diff` PR gate; explicit division of labor (Vale = mechanical, skill = structural/voice).

**Detectors / benchmarks (measure rather than gate):**
- **EQ-Bench Slop Score** (`sam-paech/slop-score`, ~27 stars, dual MIT/Apache/CC-BY-SA) — runs in-browser; scores over-used words, "not X but Y" constructions, and over-represented trigrams against a human baseline. Explicitly *not* an AI detector; a compass, not a verdict.
- **slop-forensics** and **auto-antislop** (Sam Paech) — fingerprinting and pipeline tooling.
- Research: the *Science Advances* excess-vocabulary study (above); Shaib et al. (2025) *Measuring AI Slop in Text* builds a taxonomy from expert interviews and span-level annotation.

**Generation-time suppression:**
- **antislop-sampler** (`sam-paech/antislop-sampler`, ~347 stars, MIT) — backtracks during generation to suppress thousands of known slop phrases (phrase-level, not per-token logit biasing). Integrated into koboldcpp. The broader **Antislop framework** (arXiv:2510.15061, ICLR 2026) adds FTPO fine-tuning to make suppression permanent while preserving benchmark capability.

**AI-code slop (adjacent, for coding agents):** `dmmulroy/anti-slop` (Oxlint rules, Rust, 50-100× faster than ESLint — matters for tight agent loops), `skew202/antislop`, `Nutlope/hallmark` (~5.3k stars, 58 deterministic gates, `npx skills add nutlope/hallmark`). These target generated *code*, not prose, but share the deterministic-gate philosophy.

**Comparison — key AI-slop tools:**

| Tool | Runtime | Output | Auto-fix? | Maturity | Best for |
|---|---|---|---|---|---|
| slop-cop | Go | JSON (agent-first) | Rewrite via `claude` CLI (opt-in) | New (~8★) | Agent-driven CI |
| slop-lint | Node, 0-dep | Text/fail | No | New | Zero-config em-dash gate |
| textlint AI-writing preset | Node | JSON/MCP | No | Active (~1.1k★) | textlint users, MCP |
| JMill/deslop (Vale) | Vale | JSON | No | New | Drop-in Vale gate |
| Syntaf/vale-llm-slop | Vale | JSON | No | New | Vale + STE discipline |
| SlopSift | ONNX/Node | Structured | No | New | Lower false positives |
| EQ-Bench Slop Score | Browser/Python | Score | No | Established | Measuring, not gating |
| antislop-sampler | Python | (generation) | Prevents at source | ~347★ | Local model inference |

### 3. Readability scoring and enforcement

**Python:** `textstat` (~1.4k stars, 0.7.13, MIT) computes Flesch Reading Ease, Flesch-Kincaid Grade, SMOG, Coleman-Liau, ARI, Dale-Chall, Linsear Write, Gunning Fog, plus a `text_standard` consensus and several non-English formulas. Its adoption dwarfs everything else in this survey: **1,424,494 downloads in the last month (351,573 last week; 47,573 last day), per pypistats.org.** `py-readability-metrics` (`cdimascio`, MIT) covers the same set plus Spache; requires ≥100 words and NLTK punkt. Both are trivial to wrap in a CI gate (compute grade, exit non-zero above threshold).

**Node/JS:** `retext-readability` flags complex sentences via multiple formulas; `textlens` and various "Hemingway clone" libraries compute 8 formulas; `text-readability` npm port of textstat.

**CLI with CI gating:** `papa` (`bharadwaj-pendyala/papa`, pipx) — Hemingway-grade: scores ARI/FK/Gunning Fog, highlights hard sentences/passive/adverbs/complex phrases, ignores frontmatter and code, emits JSON, and **gates CI via `--max-grade` and exit code**. This is the closest thing to a purpose-built "readability threshold gate" for docs pipelines.

**Vale readability:** Vale historically lacked a built-in readability check (long-standing feature request #47); the underlying `prose` Go library supports FK/Gunning-Fog/Coleman-Liau, and community style rules approximate it, but it is not a first-class extension point. Use textstat/papa alongside Vale for hard readability gates.

**Hemingway:** the canonical tool ($19.99 desktop, closed, manual, no CI/API). Open alternatives: `papa`, `Qerbz/hemingway-skill` (Python CLI + Claude Code skill, academic-tuned), miniwebtool/nexotext (browser). Hemingway targets ~grade 6; plain-language guidance recommends grade 6-9.

**Lexile:** proprietary to MetaMetrics. The only official programmatic access is the paid **Lexile partner API** (e.g., `/text/difficulty` returns an oral readability measure like "1060L") — oriented to education partners, not general dev use. The specification equation (using a ~1.4-billion-word frequency corpus) is not public. Open approximations (e.g., readabilit.com's calculator) use log-mean-sentence-length + syllable/letter proxies calibrated to correlate with public formulas — treat these as estimates, not true Lexiles. **Recommendation: use Flesch-Kincaid or a consensus grade as your enforceable metric and only convert to an approximate Lexile band for communication.**

### 4. Plain-language and accessibility checkers

- **plainlanguage.gov** (`GSA/plainlanguage.gov`, CC0 public domain) — guidance and the 237-item simplification list, *not* a linter. Content now mirrored in Digital.gov's plain-language guide series.
- **UK GDS / GOV.UK, ONS, Australian Style Manual** — provide "words to avoid" A-Z lists and plain-language rules that can be encoded as Vale/textlint substitution rules, but ship no first-party linter.
- **WCAG cognitive criteria** — SC 3.1.3 (unusual words), 3.1.4 (abbreviations), 3.1.5 (reading level), 2.4.6 (headings). Operationalized by: (a) **Siteimprove** and similar commercial suites; (b) `Community-Access/accessibility-agents` — 11 specialist agents for Claude Code/Copilot/Claude Desktop with a `cognitive-accessibility` agent covering COGA guidance, plain-language analysis, and reading-level scoring against WCAG 2.2 SC 3.3.7/3.3.8/3.3.9; (c) readability CLIs (papa, nexotext's plain-language checker) that flag passive voice, jargon, complex words, and grade level.
- **Deterministic jargon/passive/heading checks:** retext-passive, retext-simplify, Vale's Microsoft/Google styles (which flag passive voice and vague language), and custom Vale existence rules for undefined abbreviations.
- **WordRake** (commercial, $) — "Simplicity mode" claims to cover 90% of the plainlanguage.gov simplifications.
- Research/standards: the CLEARS-2025 shared task and ISO 24495-1:2023 (plain language) / Easy-to-Read standards frame the LLM plain-language rewriting space.

### 5. LLM-based rewriters and hybrid pipelines

The dominant architecture is **detect deterministically → rewrite with a model → re-lint**:
- **SlopMonster** (`ItsssssJack/SlopMonster`) — three-stage: score (`deslop.py`, stdlib, exits red below 5/5), rewrite (three passes: kill vocabulary → kill shapes → restore a person), re-lint. Key design choice: the **cleanse runs on a different model family than the one that wrote the draft** ("a model is poor at hearing its own accent"), and it refuses to route a draft back to its own family. Ships as a Claude Code skill, a Codex-compatible SKILL.md, and a `.github/workflows/slop.yml` build gate.
- **slopster** — Vale (mechanical, deterministic) + Tagore skill (structural/voice, 8-dimension scoring) + slop-diff PR gate; explicitly sequences the deterministic net after the LLM rewrite.
- **unsloppify** (`woerndl/unsloppify`, ~17 stars, MIT) — tiered agent skill: always-on baseline ruleset pasted into CLAUDE.md/AGENTS.md, plus banned-vocabulary tiers (importance inflation, manufactured drama, performative register, false precision, template filling). Preserves facts; warns about overcorrection.
- **deslopify** (`glaforge/deslopify`) — Gemini CLI skill grounded in the tropes.fyi trope list.
- **de-slopify** (Dicklesworthstone) and **avoid-ai-writing** — Claude Code skills that rewrite reader-facing prose, emphasizing that the fix is "reinstating the concrete detail the slop displaced," not just deleting tics.
- **Two-pass / separation-of-concerns pattern**: keep style evidence out of the substance layer — develop content first, then run a dedicated prose-cleanup pass (Noren, tropes.fyi, and the voice-profile literature all advocate this).
- **Evals**: EQ-Bench Slop Score for measuring; blind LLM-judge + rule-based AI-ism counters (the unslop project reports "100% preference / 92% reduction" from its harness — vendor-reported, treat with caution).

**Research backing metric-guided rewriting**: a study (N=2,000) found a metric-guided prompt (embedding the readability formula) significantly reduced FK grade vs. a plain-text "make it simpler" instruction; another study warns of a "brevity bias" where LLMs shorten rather than genuinely simplify, and sometimes introduce misinformation — so re-linting and fact-checking after rewrite is essential.

### 6. Style-profile / voice-matching tools

- **WRITER Voice** (commercial) — upload sample copy; two specialized LLMs (a voice-extraction model and a voice-generation model) build a profile and generate matching text; per-product/channel profiles.
- **Noren** (`usenoren.ai`) — exports a portable `voice-profile.md` (frequency targets, anti-patterns, triggers) usable across ChatGPT/Claude/Gemini/Ollama.
- **Agent-skill voice profiles** — `unslop`, slopster's `voice.md`, and `avoid-ai-writing` all ship a "build-your-own voice profile" method so output reads as its author, not just generically de-slopped. This directly addresses the user's implicit goal.
- **Research**: profile-based prompting beats within-user and cross-user RAG for readability/style alignment (ReLay); the *Catch Me If You Can? Not Yet* study (arXiv:2509.14543) found LLMs approximate structured styles (news, email) but struggle with informal blog/forum voice; instruction-tuning makes output *less* human on Biber grammatical features (PNAS, *Do LLMs write like humans?*). Practical implication: a compact, explicit rules + gold-sample corpus outperforms tone adjectives.

### 7. Practical integration

- **pre-commit hooks**: Vale, proselint, textlint, alex all ship or document pre-commit configs. Best for committed files (docs, READMEs, commit bodies).
- **CI (GitHub Actions)**: `vale-cli/vale-action`, textlint/proselint runners, papa's `--max-grade` gate, SlopMonster's `slop.yml`, slopster's `slop-diff`. Gate on error-level rules only to avoid blocking on suggestions.
- **Editor LSP**: Vale (VS Code/Cursor/Windsurf/Neovim/Zed/JetBrains/etc.), alex, SlopSift, proselint (via diagnostic-languageserver) give as-you-type diagnostics.
- **Agent-invoked self-revision (the key pattern for the user's problem)**: the agent lints and silently revises its own draft before replying. Enabled by (a) MCP servers — textlint v14.8.0+ and the community "5 linters + AI-tell" MCP server expose linting as tools the agent calls; (b) SKILL.md skills (SlopMonster, unsloppify, slopster/Tagore, deslopify) that instruct the agent to run the linter, read findings, and recast; (c) rules files (CLAUDE.md/AGENTS.md/.cursorrules/GEMINI.md, tropes.md) that suppress tells at generation time.
- **Chat-turn vs. committed-file**: most deterministic linters assume files, but the **CLI-with-stdin** tools (proselint `-`, slop-cop stdin, slop-lint, textstat as a library) and **MCP/skill** approaches work on turn-level responses. This is the crucial distinction for a coding agent: to clean chat output rather than commits, you need the stdin/library/MCP path, not the pre-commit path.

## Recommendations

**Stage 1 — Deterministic gate (do this first, ~30 min):** Install Vale (`brew install vale`), add a `.vale.ini` with the Microsoft or Google style package plus an AI-tells package (`tbhb/vale-ai-tells` or `JMill/deslop`, or `Syntaf/vale-llm-slop`'s `Slop` style). Run it in your editor (LSP) and as a pre-commit hook / GitHub Action. This catches em-dash pileups, "delve"/"leverage"/"robust," negation pivots, and filler openers with zero model calls and JSON output an agent can consume. Set only the mechanical tells to error level; leave the rest as suggestions.

**Stage 2 — Readability threshold (~15 min):** Add `textstat` (Python) or the `papa` CLI to compute Flesch-Kincaid grade and gate CI at your target (grade 8-10 for technical docs; papa's `--max-grade`). Do **not** chase Lexile — use FK grade as the enforceable number and only approximate Lexile for stakeholder communication.

**Stage 3 — Agent self-revision (the biggest quality win):** Wire one agent skill as a pre-reply pass. Best choices: **unsloppify** (tiered, fact-preserving, installs into CLAUDE.md/AGENTS.md) for a lightweight always-on layer, or **SlopMonster** if you want the rival-model cleanse + build gate. Add `tropes.md` (from tropes.fyi) to your system prompt / AGENTS.md to suppress tells at generation time. If you use textlint, run it as an MCP server so the agent lints itself.

**Stage 4 — Voice matching (optional, if generic de-slopping isn't enough):** Build a `voice-profile.md` from a corpus of your own writing (Noren export, or the templates in unslop/slopster) and feed it to the rewrite step so output sounds like you, not like generic clean prose. Temper expectations: the imitation research shows LLMs match informal personal voice imperfectly, so treat the profile as a strong nudge, not a guarantee.

**What would change these recommendations:**
- If you write in **languages other than English**, prioritize LanguageTool (25+ languages) and the translationese packs in slop-gate.
- If you need **accessibility compliance** (WCAG/Section 508), add the `Community-Access/accessibility-agents` cognitive-accessibility agent and a plain-language checker; readability alone won't satisfy SC 3.1.3/3.1.4/2.4.6.
- If linter **false positives** frustrate the team, move from word-list tools (slop-lint) toward parser-based ones (SlopSift) and lean on Vale vocabularies to whitelist terms of art.
- If you're gating **generated code** rather than prose, use `dmmulroy/anti-slop` (Oxlint) or `Nutlope/hallmark` instead — different problem, same deterministic-gate philosophy.

## Caveats

- **Adoption is low and churn is high** for AI-slop-specific tools: many key repos have single-digit-to-low-hundreds stars (slop-cop ~8, unsloppify ~17, JMill/deslop ~0, slop-score ~27) and are months old. Expect abandonment, renames, and breaking changes; vendor these into your repo rather than depending on them long-term. The mature anchors (Vale ~6.1k★, textstat ~1.4M downloads/month, textlint ~3.1k★, LanguageTool ~15k★, alex ~5.1k★) are the safe bets.
- **The em-dash-as-decisive-tell heuristic is contested.** Several tools (slop-lint, SlopMonster) treat em-dashes as near-decisive, but human writers use them legitimately; blanket failing on em-dashes will false-positive. Restrict to prose-punctuation em-dashes (not code/CLI/math/ranges) and treat density, not any single occurrence, as the signal.
- **AI-detector accuracy is poor.** Slop *linters* (pattern counters) differ from AI *detectors* (classifiers); the latter false-positive constantly. Use these tools to improve readability, not to reliably prove text was machine-written.
- **The cat-and-mouse problem**: as tells like "delve" get suppressed, models shift to new patterns; word lists decay. Statistical-discovery tools (slop-lint `--discover`, slop-forensics) that re-derive tells per model are more durable than static lists.
- **Vendor-reported efficacy numbers** (e.g., unslop's "100% preference / 92% reduction") come from the projects' own harnesses and are not independently verified.
- **LLM rewriting can degrade content**: research documents a "brevity bias" (shortening instead of simplifying) and occasional misinformation introduction. Always re-lint and fact-check after a model rewrite; separate content development from prose cleanup.
- **Single-maintainer risk** applies to several anchors (Vale/@jdkato, the self-described sole developer; alex/@wooorm; the textlint AI preset/azu). They are healthy today but concentrate bus-factor risk.
- Some data points (star counts, download stats) are rounded and current as of the September 2026 research window; treat them as directional.