# url-to-wiki-stub

A Claude skill that generates MediaWiki-formatted stub entries from URLs for quick capture into a local MediaWiki instance.

## What it does

Paste one or more URLs with a trigger word and get copyable MediaWiki markup — a linked title, source metadata, synopsis, and suggested tags. Designed for fast link capture and triage, not full article creation.

## Usage

Prefix URLs with a trigger word **at the start of your message** (leading spaces/tabs are fine):

### Full stub (`wiki` or `mediawiki`)

```
wiki https://example.com/some-article
```

Produces up to 8 lines: linked title, metadata (source, capture date, author), up to 5 synopsis lines, and suggested tags.

### Short stub (`wikis`)

```
wikis https://example.com/some-article
```

Produces 4–5 lines: same metadata and tags, but synopsis is condensed to 1 line (3 max).

### Prioritised reading list (`reading-list`)

```
reading-list "impact of AI on software security" https://example.com/article1 https://example.com/article2 https://example.com/article3
```

Fetches all URLs, ranks them by relevance to the quoted topic, and outputs the same stub format with added priority rank, relevance tier (HIGH/MEDIUM/LOW), estimated reading time, and a relevance explanation line. Results are ordered highest priority first.

### Question log (`question-log` or `qlog`)

```
question-log "impact of AI on software security" https://example.com/article1 https://example.com/article2
qlog https://example.com/article1
```

Fetches all URLs and generates a categorised list of open questions grouped into FACTUAL (answerable with further research), METHODOLOGICAL (probing reasoning and validity), and SPECULATIVE (beyond current evidence). When a quoted topic is provided, questions are framed relative to that topic. Can also be used with a topic alone and no URLs.

### Claude Code — slash commands

In Claude Code, each skill folder name becomes a `/name` slash command automatically. The `wiki/` and `wikis/` skill folders provide:

```
/wiki https://example.com/article1 https://example.com/article2
/wikis https://example.com/article1
/wiki all https://example.com/article1 https://example.com/article2 https://example.com/article3
```

The `reading-list` skill is also available as a slash command in Claude Code:

```
/reading-list "my research question" https://example.com/article1 https://example.com/article2
```

The `question-log` skill is also available as a slash command:

```
/question-log "my research question" https://example.com/article1 https://example.com/article2
```

These are self-contained skills with their own SKILL.md — they don't depend on `url-to-wiki-stub` being loaded.

**Note:** Slash commands are a Claude Code feature only. Claude.ai (web) and Claude Desktop use the trigger-word approach (`wiki https://...` typed as a regular message) via the auto-triggered `url-to-wiki-stub` skill. The `/wiki` and `/wikis` folders have no effect in those environments.

**Note:** Claude Code may also auto-trigger these skills based on their description. To prevent this, you can add `disable-model-invocation: true` to the frontmatter in `wiki/SKILL.md` and `wikis/SKILL.md`, though there is a [known issue](https://github.com/anthropics/claude-code/issues/26251) where this can also block user invocation via slash command.

### Unlimited batch (`all` modifier)

Add `all` after the trigger word to skip the 10-URL warning and process everything:

```
wiki all https://example.com/article1 https://example.com/article2 https://example.com/article3 ...
wikis all https://example.com/article1 https://example.com/article2 ...
```

URLs can also be separated by newlines.

**Rate limiting note:** Fetching many URLs in sequence may hit rate limits depending on your Claude plan. In practice, batches of URLs from different sites are unlikely to be affected since rate limiting is per-domain. Batches with many URLs from the same domain are more likely to encounter issues.

**Token usage note:** Each fetched page adds content to Claude's context window, and this accumulates across a batch. The 10th URL in a batch costs significantly more input tokens than the 1st because Claude is carrying all previous page content and generated stubs in context. Very large batches may hit the context window limit entirely, causing failures on later URLs. For large batches, prefer `wikis` (short-form) to reduce context accumulation, or split across separate messages.

The skill will **not** trigger if "wiki" appears mid-sentence (e.g., "what does the wiki say about X").

## Example output

### Full

```
* [https://example.com/post/rust-1-80 Rust 1.80 Introduces LazyCell and LazyLock]
*: '''Source:''' example.com | '''Captured:''' 2026-02-24 | '''Author(s):''' Jane Smith
*: Covers the stabilization of LazyCell and LazyLock in Rust 1.80 for lazy initialization.
*: These replace common patterns that previously required the once_cell crate.
*: LazyCell is for single-threaded contexts while LazyLock is its thread-safe counterpart.
*: The post walks through migration examples from once_cell to the new standard library types.
*: Also mentions several other smaller stabilizations included in this release.
*: '''Tags:''' Rust, programming languages, standard library, lazy initialization
```

### Short

```
* [https://example.com/post/rust-1-80 Rust 1.80 Introduces LazyCell and LazyLock]
*: '''Source:''' example.com | '''Captured:''' 2026-02-24 | '''Author(s):''' Jane Smith
*: Rust 1.80 stabilizes LazyCell and LazyLock for lazy initialization, replacing the need for the once_cell crate.
*: '''Tags:''' Rust, programming languages, standard library
```

## Building

Requires `make` and `zip`.

```bash
cd url-to-stub
make
```

This produces `dist/url-to-wiki-stub.skill` — a zip file ready for upload.

To clean build artifacts:

```bash
make clean
```

## Installation

### Claude.ai (web)

1. Run `make` to build the `.skill` file
2. Go to **Settings → Capabilities**
3. Scroll to **Skills** and click **Upload skill**
4. Upload `dist/url-to-wiki-stub.skill`
5. Ensure the skill toggle is **on**

The skill is private to your account. Team and Enterprise admins can provision it organization-wide from the same settings page.

The uploaded zip includes all skills, but only `url-to-wiki-stub`, `reading-list`, and `question-log` (the auto-triggered skills) are used in the web interface. The `wiki/` and `wikis/` slash command folders are ignored — use the trigger-word approach instead (`wiki https://...`, `reading-list "topic" https://...`, or `question-log "topic" https://...` as a regular message).

### Claude Desktop

Installation is the same as Claude.ai web:

1. Run `make` to build the `.skill` file
2. Open **Settings → Capabilities**
3. Under **Skills**, click **Upload skill**
4. Upload `dist/url-to-wiki-stub.skill`
5. Toggle the skill **on**

As with the web version, only the auto-triggered skills (`url-to-wiki-stub`, `reading-list`, and `question-log`) are used. Slash commands (`/wiki`, `/wikis`, `/reading-list`, `/question-log`) are not available in Desktop.

### Claude Code (CLI)

Claude Code uses directory-based skills rather than zip files. Copy the skill folders to one of the supported locations:

**Personal skills** (available across all projects):

```bash
mkdir -p ~/.claude/skills
cp -r skills/url-to-wiki-stub ~/.claude/skills/   # auto-triggered skill
cp -r skills/wiki ~/.claude/skills/                # /wiki slash command
cp -r skills/wikis ~/.claude/skills/               # /wikis slash command
cp -r skills/reading-list ~/.claude/skills/        # /reading-list slash command
cp -r skills/question-log ~/.claude/skills/        # /question-log slash command
```

**Project skills** (available only in a specific project, shared via git):

```bash
mkdir -p /path/to/your/project/.claude/skills
cp -r skills/url-to-wiki-stub /path/to/your/project/.claude/skills/
cp -r skills/wiki /path/to/your/project/.claude/skills/
cp -r skills/wikis /path/to/your/project/.claude/skills/
cp -r skills/reading-list /path/to/your/project/.claude/skills/
cp -r skills/question-log /path/to/your/project/.claude/skills/
```

No restart required — Claude Code discovers skills automatically.

In Claude Code you can use either approach: type `wiki https://...` as a message (auto-triggered via `url-to-wiki-stub`) or use the slash commands (`/wiki https://...`, `/wikis https://...`). Install whichever suits your workflow — you don't need all three.

## Repo structure

```
url-to-stub/
├── README.md
├── Makefile
├── docs/
│   └── skill-design.md            # design decisions and considerations
├── skills/
│   ├── url-to-wiki-stub/
│   │   └── SKILL.md              # main skill (auto-triggered)
│   ├── wiki/
│   │   └── SKILL.md              # /wiki slash command (Claude Code)
│   ├── wikis/
│   │   └── SKILL.md              # /wikis slash command (Claude Code)
│   ├── reading-list/
│   │   └── SKILL.md              # /reading-list slash command (Claude Code)
│   └── question-log/
│       └── SKILL.md              # /question-log slash command (Claude Code)
└── dist/                          # created by make
    └── url-to-wiki-stub.skill     # zip file for upload
```

## Compatibility

This skill follows the [Agent Skills open standard](https://agentskills.io), so the `SKILL.md` format is portable across any compatible agent. Confirmed adopters of the standard include Claude Code, OpenAI Codex CLI, Cursor, GitHub Copilot (VS Code), Goose, Windsurf, Gemini CLI, Roo Code, Trae, Amp, and OpenCode.

### What works everywhere

The core `url-to-wiki-stub/SKILL.md` — the frontmatter (`name`, `description`) and markdown instructions are the universal standard. Any skills-compatible agent will discover and load it.

### What may need adaptation

- **`web_fetch` tool reference.** The skill instructs Claude to use `web_fetch` to retrieve URLs. Other agents may expose this capability under a different tool name (e.g., `fetch`, `browse`, `http_get`). You may need to adjust the tool name in the SKILL.md for your platform.
- **`$ARGUMENTS` in slash commands.** The `/wiki` and `/wikis` slash command skills use `$ARGUMENTS`, which is a Claude Code convention. Other platforms may handle command arguments differently or not support them at all.
- **Trigger word detection.** The "first word of message" trigger pattern relies on the model understanding and following that instruction. More capable models (Claude, GPT-4) handle this reliably; smaller models may not.
- **Output quality.** Synopsis quality depends on the model's ability to paraphrase, summarize, and follow formatting rules consistently. Weaker models may produce inconsistent formatting or lower-quality summaries.
- **Sanitization.** The markup injection mitigations are prompt-level instructions. Their effectiveness varies with model capability.

### Platform-specific notes

| Platform | Auto-trigger | Slash commands | Notes |
|---|---|---|---|
| Claude.ai (web) | Yes | No | Upload `.skill` zip via Settings → Capabilities |
| Claude Desktop | Yes | No | Same zip upload as web |
| Claude Code | Yes | Yes (`/wiki`, `/wikis`) | Copy skill folders to `~/.claude/skills/` |
| OpenAI Codex CLI | Yes | Varies | May need tool name adjustment for URL fetching |
| Cursor | Yes | Yes | Supports Agent Skills natively |
| VS Code (Copilot) | Yes | Yes | Via `chatSkills` extension point |
| Other adopters | Likely | Varies | Check platform docs for skill discovery paths |

## Security considerations

This skill fetches arbitrary URLs and processes their content. There are inherent risks to be aware of:

**Prompt injection via fetched pages.** When Claude fetches a URL, the page content enters its context window. A malicious page could include hidden text with adversarial instructions that attempt to override Claude's behavior. This risk is inherent to any web-fetching workflow and relies on Claude's built-in prompt injection resistance for mitigation. The skill explicitly instructs Claude to ignore any instructions found within fetched page content.

**MediaWiki markup injection.** If a page title, author name, or other extracted field contains MediaWiki syntax (e.g., `{{delete:}}`, `[[Category:Malware]]`, or `<script>` tags), it could execute templates, create unwanted links, or attempt XSS when pasted into your wiki. The skill includes sanitization rules that strip or escape MediaWiki special characters (`{{ }}`, `[[ ]]`, `< >`, HTML tags) in all output fields. However, you should still review generated stubs before pasting them into your wiki.

**Content accuracy.** A page could be crafted to produce a misleading synopsis — for example, malware disguised as a legitimate tool. Since this skill generates reference material, always verify stubs for content that seems unusual or suspicious.

**No exfiltration risk.** Output stays in the conversation for manual copy-paste. No data is sent externally, no files are written, and no API keys or credentials are involved.

## Notes

- Output is presented inline as a code block for copying — no files are generated
- Capture date is the date you clip the link, not the article's publication date
- Tags are plain-text labels for your own sorting, not MediaWiki `[[Category:]]` tags
- Multiple URLs can follow a single trigger word, one per line
- URLs are processed and output one at a time so you see progress incrementally
- For best results, keep batches to 10 or fewer URLs per message — use `all` to override this limit
- Requires a paid Claude plan (Pro, Max, Team, or Enterprise)

## Recommended models

This skill works with any Claude model. For cost-effective use:

- **Sonnet** — recommended for general use. The task (fetch, summarize, format) doesn't require Opus-level reasoning.
- **Haiku** — sufficient for `wikis` short-form stubs where the output is a single synopsis line.
- **Opus** — works fine but costs more per stub, especially in batches.

Select your preferred model before starting the conversation — changing models mid-conversation starts a new session.
