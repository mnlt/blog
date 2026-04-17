# How wellread works under the hood

In the [previous post](../2026-04-13-wellread/en.md) I explained why I built wellread. Here's how it works.

## The flow

When you ask Claude something with wellread installed, several things happen before it searches the web:

1. A hook (UserPromptSubmit) intercepts your message before Claude processes it
2. The hook injects a 4-line instruction: "search wellread first, save after"
3. The hook also reads your local JSONL to know your current context baseline
4. Claude sanitizes your question — strips project names, paths, credentials — and calls wellread
5. Wellread searches with embeddings + full text
6. Depending on the result: hit, partial, or miss

### Hit

Wellread returns a ~600 token summary. Claude responds with that. Doesn't touch the web. One turn.

### Partial

Wellread returns what it has. Claude reads the gaps (what wasn't explored) and only researches those. When done, saves the updated version.

### Miss

Claude researches normally — web searches, fetches, reasoning. When done, saves the result to wellread for the next person.

## The search

The query that reaches wellread isn't what you typed. Claude transforms it into a generic technical concept. If you ask "how do I set up auth in my Next.js project at /Users/mnlt/projects/myapp", what reaches wellread is something like "how to set up authentication in Next.js App Router with Auth.js".

Nothing private leaves your machine beyond the generic question.

The search combines two things:
- **Embeddings** (OpenAI, text-embedding-3-small): searches by meaning, not exact words. "configurar autenticación" matches "authentication setup"
- **Full text** (BM25): searches by keywords. Catches what embeddings miss — library names, specific versions

Results are fused with Reciprocal Rank Fusion (70% semantic, 30% text). Returns up to 5 results.

## Freshness

Not all knowledge expires the same way. TCP fundamentals don't change. A library beta might change tomorrow.

Each research is saved with a volatility level:

| Level | Example | Fresh | Check | Stale |
|---|---|---|---|---|
| Timeless | TCP, SQL basics | 1 year | — | after |
| Stable | React, PostgreSQL | 6 months | 1 year | after |
| Evolving | Next.js, Bun | 30 days | 90 days | after |
| Volatile | betas, pre-releases | 7 days | 30 days | after |

The agent that contributes the research decides the volatility. Not perfect, but works reasonably well.

When someone hits an entry in "check" status, Claude doesn't trust it — does a quick web search to verify it still holds. If confirmed, resets the clock for the next person. If outdated, replaces it with a new version.

This way knowledge stays fresh without anyone doing manual maintenance.

## Gaps

Each research is saved with a list of "what wasn't explored." If someone searched "auth in Next.js" but didn't cover OAuth with Google, that's logged as an explicit gap.

When another agent arrives with a question the research partially covers, it knows exactly what to investigate and what not to. Doesn't repeat what's already there, doesn't make up what's missing — only searches the gaps.

## Replaces and versions

If you research something that already existed but with newer info, your research replaces the previous one. The old entry stops appearing in searches but isn't deleted — the version chain is preserved.

This matters because sources and saved tokens accumulate across versions. A research that started with 3 sources and has been replaced twice might have 8 sources and a history of how the knowledge evolved.

## How tokens are measured

The hardest part to solve. I needed to know how many tokens a wellread hit actually saves. Not an estimate — the real number.

### PostToolUse hook

After each save, a hook (PostToolUse) fires automatically. This hook:

1. Reads the JSONL from your current session
2. Finds the span between the wellread search and the wellread save
3. Sums `input_tokens + cache_creation + cache_read` for each turn in that span
4. Subtracts the baseline (what would have been sent anyway)
5. Sends the result to the server via `PATCH /measure`

The result is the real incremental cost of the research — not an estimate, the exact number from the JSONL.

### The personalized badge

When you get a hit, the badge doesn't show what the original research cost. It shows what it would have cost **you**, with **your** current baseline.

The formula: `raw_tokens + (your_baseline × research_turns) - response_tokens`

A research that cost 200K in a fresh session could cost you 3.5M if you're at turn 100. The badge shows that 3.5M.

For this, the UserPromptSubmit hook reads your JSONL and passes your current baseline as `client_stats`. The server does the calculation with the research data (raw_tokens, research_turns) and your baseline.

## The hook

The hook is what makes wellread actually work in Claude Code. Without it, the agent would have to decide on its own whether to search wellread. With the hook, it always searches.

It's a 15-line bash script installed at `~/.wellread/hook.sh`. What it does:

1. Reads your message (if it's very short, it skips — doesn't search for "hey" or "ok")
2. Reads your baseline from the JSONL (last line with usage)
3. Injects 4 lines of instructions into the system prompt

The instructions are deliberately minimal. Every token in the hook gets re-sent on every turn of the session — a 500-token hook costs 500 tokens × N turns. Wellread's is ~106 tokens.

The other thing: everything possible lives on the server. If the search logic, freshness, or badges change, all users get it without doing anything. Only if the hook changes do they need to run `npx wellread@latest`.

## The installer

`npx wellread` detects which tools you have installed and configures everything:

- **Claude Code**: MCP server in settings.json + UserPromptSubmit hook + PostToolUse hook (for measurement)
- **Cursor**: MCP server in mcp.json + rule in rules
- **Windsurf, Gemini CLI, VS Code, OpenCode**: equivalents for each

No manual API keys. The installer registers an anonymous user and saves the key in the client config.

## Privacy

There are six layers between your private context and the shared network:

1. **Hook instruction** - before anything leaves your machine, the hook tells the agent to sanitize the query: strip project names, API keys, file paths, credentials. Only the generic technical concept gets sent.
2. **Search schema** - the search tool parameter reinforces: "Remove project names, API keys, file paths, credentials."
3. **Save schema** - the save tool says: "NEVER include project/repo/company names, internal URLs, file paths, credentials, business logic. Content is PUBLIC."
4. **URL gate (server, hard reject)** - every source must start with `https://` or `http://`. File paths, library identifiers, internal URLs - rejected. The contribution doesn't get saved.
5. **Path detection (server, hard reject)** - the server scans content and search surface for local paths (`/Users/...`, `/home/...`, `file://`, `C:\...`). Found one? Rejected.
6. **By design** - the agent doesn't forward your input. It synthesizes from public sources. What gets saved is a distilled summary of public docs, not your code or conversation.

For something private to reach another user, the agent would have to sneak it past its own instructions, past the URL gate, past the path regex, into a generic summary - and then someone would need to search something similar enough to surface it.

If you want to try: `npx wellread`

GitHub: https://github.com/mnlt/wellread
