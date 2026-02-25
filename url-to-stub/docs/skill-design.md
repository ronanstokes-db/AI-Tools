# url-to-wiki-stub — Design Decisions and Considerations

This document captures the design decisions, trade-offs, and considerations made during the development of the `url-to-wiki-stub` skill.

## Purpose

The skill generates MediaWiki-formatted stub entries from URLs for quick capture into a local MediaWiki instance. It is designed as a "clip and sort later" workflow — entries accumulate on a capture page and the user redistributes them to appropriate wiki pages manually.

## Core design principle

Keep it simple. The skill is prompt-only — no scripts, no dependencies, no file generation. It fetches a URL, generates markup, and presents it in a code block for the user to copy. This makes it portable, safe, and easy to maintain.

## Trigger word design

### Problem

"wiki" is a common word. Using it as a trigger risked false positives from messages like "what does the wiki say about X?" or "can you wiki this topic for me?"

### Decision

Require the trigger word (`wiki`, `mediawiki`, or `wikis`) to be the **first word** of the message. Leading spaces and tabs are permitted, but the trigger must not appear mid-sentence. The skill description includes explicit "should not trigger" examples to help Claude disambiguate.

### Alternatives considered

More distinctive trigger words like `stub` or `clip` were considered but rejected in favour of the more natural `wiki` with positional constraint. The trigger set was intentionally kept small — `wiki`, `mediawiki`, and `wikis` — rather than allowing a broad set of synonyms.

## Output format

### Structure (full stub — `wiki` / `mediawiki`)

Each entry is up to 8 lines:

1. Bulleted external link: `* [URL Title]`
2. Metadata: source, capture date, author(s)
3. Synopsis lines 1–5
4. Suggested tags

### Structure (short stub — `wikis`)

Each entry is 4–5 lines:

1. Bulleted external link
2. Metadata
3. Synopsis: 1 line preferred, 3 max
4. Tags

### Key formatting decisions

**MediaWiki `*:` indentation.** Description lines use the `*:` prefix so they render visually indented under the bullet point in MediaWiki. This was chosen over plain indentation or nested bullets for consistent rendering across wiki skins.

**Bold labels in metadata.** The metadata line uses MediaWiki bold (`'''Source:'''`, `'''Captured:'''`, `'''Author(s):'''`) for readability. This is the only wiki markup permitted outside the title line.

**No file generation.** Output is presented inline as a code block. Earlier iterations saved `.wiki` files, but the `.wiki` extension doesn't render in Claude.ai's UI, and the user's workflow is copy-paste anyway. Eliminating file creation simplifies the skill and avoids platform-specific rendering issues.

**Capture date vs. publication date.** The metadata records the date the link was captured (today's date), not the article's publication date. This serves as a "when I saved this" timestamp for the user's triage workflow. Publication date was considered but rejected as unreliable to extract programmatically.

## Tags

### Problem

Without a controlled vocabulary, tags will drift over time — "AI", "artificial intelligence", "artificial-intelligence", "machine learning" for overlapping topics. This undermines the sorting purpose.

### Decision

Tags are unconstrained plain-text labels (2–5 per stub). The skill instructs Claude to prefer broader established terms, but does not enforce a taxonomy. This was a deliberate trade-off: a fixed tag list would be too rigid for varied content, and maintaining a taxonomy adds complexity that doesn't fit a prompt-only skill.

### Not MediaWiki categories

Tags are explicitly not `[[Category:]]` tags. Since many entries live on a single capture page, category tags would apply to the page, not individual entries. Tags are plain text for the user's own sorting.

## Author detection

The skill uses "Unknown" as a fallback when authorship cannot be determined from the page. Claude is instructed not to guess — this avoids hallucinating author names from byline-like text (editor names, translator names, company names). The trade-off is that many stubs will show "Unknown", but accuracy is more important than completeness for a reference wiki.

## Batching

### Processing model

URLs are processed one at a time: fetch, generate stub, output, then move to the next. This gives the user incremental progress rather than waiting for all URLs to finish. If one URL fails, processing continues — the failure is noted with a `<!-- Could not fetch: URL -->` comment and the batch is not interrupted.

### Batch limit

A warning is issued for batches exceeding 10 URLs, suggesting the user split across messages. The user can override this with the `all` modifier (e.g., `wiki all https://... https://...`).

10 was chosen as the threshold — large enough for practical use, small enough to manage context window accumulation.

### Context window accumulation

Each fetched page adds content to Claude's context window, and this accumulates across a batch. The 10th URL in a batch consumes significantly more input tokens than the 1st because the context now contains all previous page content and generated stubs. This has three consequences:

- **Cost:** Input tokens scale with batch size, making later URLs progressively more expensive.
- **Quality:** As context fills, Claude has less room to reason, potentially degrading synopsis quality for later URLs.
- **Hard limit:** Very large batches can hit the context window ceiling, causing failures.

The skill instructs Claude to monitor context usage and stop with a suggestion to continue in a new message if quality degrades. The `wikis` short form is recommended for large batches as it generates less output per stub.

### Rate limiting

Rate limiting is per-domain, so batches of URLs from different sites are unlikely to be affected. Same-domain batches are more susceptible. This is noted in the README but not mitigated in the skill itself.

## Security

### Prompt injection

Fetched page content enters Claude's context window. A malicious page could include hidden text with adversarial instructions. The skill explicitly instructs Claude to ignore any instructions found within fetched page content, but mitigation ultimately relies on Claude's built-in prompt injection resistance. This risk is inherent to any web-fetching workflow.

### MediaWiki markup injection

Page titles, author names, or other extracted fields could contain MediaWiki syntax that executes unintended actions when pasted into a wiki:

- `{{template:}}` invocations
- `[[Category:]]` or `[[link]]` insertions
- `<script>` or other HTML tags

The skill includes explicit sanitization rules:

- Replace `{{` with `{ {` and `}}` with `} }` to prevent template execution
- Replace `[[` with `[ [` and `]]` with `] ]` to prevent unintended wikilinks
- Strip all HTML tags from output fields

These are prompt-level instructions, not programmatic enforcement. Users should still review stubs before pasting into their wiki.

### Content accuracy

A page could be crafted to make Claude produce a misleading synopsis. Since the skill generates reference material, poisoned stubs could sit in a wiki and mislead later readers. No mitigation beyond user vigilance.

### No exfiltration risk

Output stays in the conversation for manual copy-paste. No data is sent externally, no files are written, no API keys or credentials are involved.

## Platform differences

### Claude.ai (web) and Claude Desktop

Use the auto-triggered `url-to-wiki-stub` skill. The user types `wiki https://...` as a regular message. Slash commands are not available — the `wiki/` and `wikis/` folders in the skill zip are ignored.

### Claude Code

Both approaches work:

- **Auto-triggered:** `url-to-wiki-stub` fires when the user types `wiki https://...` as a message.
- **Slash commands:** `/wiki https://...` and `/wikis https://...` are available because each skill folder name becomes a slash command. These use `$ARGUMENTS` to receive URLs.

The slash command skills are self-contained with their own SKILL.md — they don't depend on the main skill being loaded.

### `disable-model-invocation` trade-off

In Claude Code, the `/wiki` and `/wikis` slash commands can also auto-trigger based on their description. Adding `disable-model-invocation: true` to their frontmatter would prevent this, but there is a known bug (anthropics/claude-code#26251) where this flag can also block user invocation via slash command. The frontmatter is left at defaults for now.

## Model recommendations

The skill works with any Claude model. Recommendations:

- **Sonnet** — best cost/quality balance for the task (fetch, summarize, format).
- **Haiku** — sufficient for `wikis` short-form stubs.
- **Opus** — works but costs more per stub, especially in batches.

Changing models mid-conversation starts a new session, so the model should be selected before the first `wiki` message.

## Question log skill

### Purpose

The question-log skill maps what you don't know. Given one or more URLs and/or a research topic, it produces a categorised list of open questions — things the articles don't answer, assumptions they don't examine, and tensions between sources. Where wiki stubs capture *what was said* and reading lists prioritise *what to read*, the question log identifies *what is missing*.

### Why three categories

Questions are grouped into FACTUAL, METHODOLOGICAL, and SPECULATIVE. This categorisation was chosen because each type demands a different response from the researcher:

- **FACTUAL** questions have answers that exist somewhere — the user needs to find more data, another source, or do a specific lookup. These are actionable immediately.
- **METHODOLOGICAL** questions probe the reasoning and validity of what's been read. They help the researcher evaluate source quality and identify where conclusions may not follow from evidence.
- **SPECULATIVE** questions go beyond current evidence into implications and possibilities. They're useful for framing a research agenda or identifying where original thinking is needed.

A simpler two-category split (answerable/unanswerable) was considered but loses the methodological middle ground, which is often the most valuable for critical reading.

### Output format

The format deliberately differs from wiki stubs and reading lists. Questions use `== Section ==` headings to separate categories (because they're independent groups, not a single ranked list) and a lighter per-entry structure (question, source, context — no tags, no metadata line). This keeps the output scannable when there are 10+ questions.

Each question has a stable ID (F1, M1, S1) within its category. These provide reference anchors for discussion — "let's investigate F3 first" — without implying priority ordering within a category.

### Flexible input modes

The skill accepts three input patterns:

- **Topic + URLs**: Questions framed relative to the topic, drawing on article content. This is the primary use case — "given what I'm researching, what don't these articles tell me?"
- **URLs only**: Questions derived purely from the articles' own claims, gaps, and tensions. Useful for critical reading without a specific research angle.
- **Topic only**: Questions a researcher would need to investigate to address the topic, generated from the model's knowledge. Useful for scoping a new research area before finding sources.

This flexibility was chosen because question generation is useful at multiple stages of research — from initial scoping (topic only) through active reading (topic + URLs) to critical evaluation (URLs only).

### Trigger design

Two trigger words: `question-log` (explicit) and `qlog` (shorthand for frequent use). The shorthand was added because this skill is likely to be used repeatedly during a research session, and typing `question-log` each time is cumbersome. Both are distinctive enough to avoid false positives.

The same first-word-of-message constraint applies. Natural requests like "what questions does this article raise?" should not trigger the skill — they use conversational phrasing that suggests the user wants a prose response, not structured MediaWiki output.

### Batching and context

Like reading-list, question-log cannot stream incrementally. All URLs must be fetched before questions can be generated because:

- Questions often arise from *tensions between* sources, not just individual articles
- Deduplication requires seeing all content (two articles might leave the same gap)
- Categorisation benefits from seeing the full picture

This means the same context window trade-offs as reading-list: higher peak usage, longer time to first output, and graceful degradation on overflow.

### Quality over quantity

The skill targets 5–15 questions total. This was a deliberate constraint because:

- Too many questions (20+) overwhelm the researcher and dilute signal
- The model tends to pad with vague questions ("What are the broader implications?") when pushed for quantity
- Each question must be specific enough to act on — search for an answer, design a study, or reason through it
- Questions must identify what is *missing*, not restate what the articles already answer

### What the skill does not do

- **No answer suggestions.** The skill identifies questions but does not attempt to answer them. Adding provisional answers would blur the line between question generation and research synthesis.
- **No priority ranking within categories.** Unlike reading-list, questions within a category are not ranked. Prioritisation depends on the researcher's specific needs, which the model cannot reliably infer.
- **No cross-session accumulation.** Each invocation is independent. There is no mechanism to merge question logs from multiple sessions into a unified "what I still don't know" list.
- **No source quality assessment.** The skill does not flag whether a source is reliable. Methodological questions probe reasoning but do not render a verdict on source credibility.

## Security implications of adding scripts

All skills in this repo are currently prompt-only. This is a deliberate design choice with significant security implications. If scripts were ever added (e.g., a concurrent URL fetcher, a sanitization validator, or a MediaWiki API uploader), the attack surface changes fundamentally.

### Prompt-only skills (current state)

The worst case is bad output. A fetched page containing adversarial content could influence Claude to produce malformed wiki markup, a misleading synopsis, or escape the intended format. But all consequences are downstream and manual — the user sees the output, reviews it, and copies it to their wiki. The failure mode is *visible*.

### Script-enabled skills (hypothetical)

Adding Python or shell scripts introduces the following risks:

**Arbitrary code execution.** Claude Code runs scripts locally with the user's permissions. A fetched page containing adversarial prompt injection could influence Claude to modify script arguments, pass malicious input, or — in the worst case — alter the script itself if the skill directory is writable. This is not theoretical: prompt injection via fetched web content is a known attack vector, and the combination of prompt injection + local code execution is the most dangerous pairing in agentic AI systems.

**File system access.** A Python script can read and write anywhere the user can. A compromised invocation could exfiltrate sensitive files, modify source code, install persistence mechanisms, or tamper with other skills.

**Network access.** A script doing concurrent URL fetches has outbound network access. Combined with prompt injection, this could be steered toward data exfiltration (sending local file contents to an attacker-controlled URL) or server-side request forgery (SSRF) against internal network resources.

**Supply chain risk.** Any `pip install` dependency is a vector. Even common libraries like `requests` or `aiohttp` pull transitive dependencies that could be compromised. A prompt-only skill has zero dependencies.

**Silent failure.** Script errors may be swallowed or produce partial output that looks correct but isn't, with no indication to the user. Prompt-only skills fail visibly — bad markup is obvious when you look at it.

### Design principle

Prompt-only skills fail *visibly* (bad output you can see before pasting). Script-enabled skills can fail *silently* with side effects the user never sees. This asymmetry is why this repo avoids scripts despite the performance benefits they could offer (e.g., concurrent fetching).

### If scripts are added in the future

If a future version introduces scripts, the following mitigations should be considered:

- **Read-only operations only.** Scripts should fetch and transform data but never write to the file system outside of explicitly designated output paths.
- **No network access beyond URL fetching.** Scripts should not make arbitrary outbound connections. If possible, use an allowlist of target domains.
- **No dependencies.** Prefer Python standard library only. If external packages are required, pin exact versions and audit transitive dependencies.
- **Input validation.** Scripts should validate all inputs (URLs, file paths, arguments) before processing. Never pass unvalidated content from fetched pages to shell commands or file operations.
- **Sandboxing.** Where possible, run scripts in a restricted environment (e.g., a container, a subprocess with limited permissions, or Claude Code's `allowed-tools` restriction).
- **User confirmation.** Any script with side effects (writing files, making network requests beyond fetching) should require explicit user confirmation before execution.

## Cross-platform compatibility

### Agent Skills open standard

The SKILL.md format is an open standard maintained by Anthropic at agentskills.io. It has been adopted by Claude Code, OpenAI Codex CLI, Cursor, GitHub Copilot (VS Code), Goose, Windsurf, Gemini CLI, Roo Code, Trae, Amp, OpenCode, and others. The core principle is "write once, use everywhere" — a skill folder with a SKILL.md file is the universal unit of packaging.

### Design choices that aid portability

The skill is prompt-only: no scripts, no Python dependencies, no Claude-specific APIs. The frontmatter uses only the universally supported `name` and `description` fields. This maximises portability across platforms.

### Design choices that limit portability

**Tool name coupling.** The skill references `web_fetch` by name. This is Claude's tool for retrieving URL content. Other agents may expose this capability under different names (`fetch`, `browse`, `http_get`, etc.). This is the single most likely point of failure on non-Claude platforms. A more portable approach would be to describe the action generically ("fetch the URL content") rather than naming a specific tool, but this risks Claude not using the right tool. The decision was to optimise for Claude's reliability at the cost of requiring minor edits for other platforms.

**`$ARGUMENTS` placeholder.** The slash command skills use `$ARGUMENTS`, which is a Claude Code convention for passing user input to a skill. Other platforms may use different placeholders or not support command arguments at all.

**Trigger word instruction.** The "must be the first word of the message" constraint is a natural-language instruction that relies on the model understanding positional context. More capable models handle this well; weaker models may misinterpret it, leading to false triggers or missed invocations.

**Sanitization instructions.** The MediaWiki markup escaping rules (`{{` → `{ {`, etc.) are prompt-level instructions. Their effectiveness depends entirely on the model following them. Less capable models may apply them inconsistently or not at all, which has security implications for wiki injection.

**Output formatting consistency.** The 8-line stub format with specific `*:` prefixes, bold labels, and pipe-delimited metadata requires precise formatting adherence. Claude follows this reliably; other models may drift from the format, especially over long batches.

### Recommendation

The skill is designed primarily for Claude and tested only on Claude. It will likely work on other Agent Skills-compatible platforms with minor adjustments (primarily the `web_fetch` tool name), but output quality and formatting consistency will depend on the underlying model. Users on non-Claude platforms should test with a few URLs before relying on batch processing.

## Reading list skill

### Purpose

The reading-list skill extends the wiki stub workflow for research triage. Given a research topic/question and a set of URLs, it fetches all articles, ranks them by relevance, and outputs prioritised stubs with relevance explanations and estimated reading times. The core value proposition is answering "what should I read first and why?"

### Why a separate skill rather than extending wiki stubs

The reading-list skill has a fundamentally different processing model — it requires all URLs to be fetched before any output can be produced (because ranking requires comparison). Wiki stubs stream incrementally. Combining them would add conditional logic that makes both harder to follow as prompt instructions. Keeping them separate also means each SKILL.md stays well under the 500-line guideline.

### Output format

The output format reuses the wiki stub structure with two additions:

- **Rank and relevance tier on the title line**: `'''[1]''' [HIGH] [URL Title]`. This provides at-a-glance priority without needing to read the full entry. The three-tier system (HIGH/MEDIUM/LOW) was chosen over a numeric score because it's easier for a model to produce consistently — asking Claude to distinguish between a 7/10 and 8/10 relevance score is unreliable, but HIGH vs. MEDIUM is a natural-language judgment it handles well.
- **Relevance explanation line**: A single sentence explaining why this article is ranked at this position relative to the stated topic. This is the key differentiator from a plain wiki stub — without it, the user has to read every synopsis to understand the ranking.
- **Estimated reading time**: Added to the metadata line. Based on a rough heuristic of 250 words per minute, rounded to the nearest minute. This is imprecise but useful for planning.

### Ranking approach

Articles are sorted by relevance tier first (HIGH → MEDIUM → LOW), then by specificity within each tier — articles that directly address the topic rank above those that are tangentially related. This was chosen over a pure numeric ranking because:

- The model produces tier judgments more consistently than fine-grained ordinal rankings
- The user can quickly scan to the HIGH section and ignore LOW entries
- Within-tier ordering is a softer judgment where minor inconsistencies are acceptable

### Trigger design

The trigger `reading-list "topic" URL URL...` uses a quoted string for the topic. This was chosen because:

- The topic may be a multi-word phrase or question — quotes provide an unambiguous delimiter
- URLs following the quoted topic are clearly separated from the topic text
- The trigger word `reading-list` is distinctive enough to avoid false positives (unlike `wiki`)

The same first-word-of-message constraint applies. General requests like "make me a reading list about X" should not trigger the skill — they lack the structured format with URLs.

### Batching trade-offs

Unlike wiki stubs, reading-list stubs cannot be streamed incrementally. The skill must fetch all URLs, assess each against the topic, rank them, and then output the full ranked list. This means:

- **Higher peak context usage**: All fetched content must be held in context simultaneously for comparison.
- **Longer time to first output**: The user sees nothing until all URLs are processed.
- **Greater risk of context overflow**: For the same number of URLs, reading-list uses more context than wiki stubs.

The 10-URL warning threshold and `all` override carry over from the wiki stub skill. The skill also instructs Claude to stop, rank, and output partial results if context limits are approached — graceful degradation rather than failure.

### What the skill does not do

- **No weighting by source quality.** All sources are treated equally — a blog post and a peer-reviewed paper get the same ranking criteria. Source authority assessment is too subjective for a prompt-only skill.
- **No de-duplication across topics.** If the same URL appears in multiple reading-list invocations with different topics, it gets re-fetched and re-ranked each time.
- **No persistent reading queue.** Each invocation is independent. There is no mechanism to append to or update a previous reading list.
- **No "read next" sequencing.** The skill ranks by relevance, not by optimal reading order (e.g., read the overview before the deep dive). This would require understanding dependencies between articles, which is unreliable.

## What the skill does not do

- **No deduplication.** The skill has no memory across sessions. The same URL processed twice produces duplicate stubs with no warning.
- **No tag taxonomy.** Tags are freeform, leading to potential inconsistency over time.
- **No publication date extraction.** Only capture date is recorded.
- **No target page specification.** The user cannot inline a wiki page name (e.g., `wiki AI-News https://...`).
- **No concurrent fetching.** All URLs are fetched sequentially. A scripted concurrent fetcher was considered but rejected to keep the skill prompt-only.

These are potential enhancements for a future version.
