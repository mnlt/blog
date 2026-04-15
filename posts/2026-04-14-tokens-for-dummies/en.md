# Claude tokens for dummies

Every time Claude does anything, it rereads your entire conversation from message one. That's why your usage limit hits faster than you expect.

# The ingredients

* **Token** — a chunk of text (~4 characters). Everything Claude reads or writes is measured in tokens.
* **Turn** — one action Claude takes. Reading your message and responding = 1 turn. A web search = 1 turn. Opening a doc = 1 turn.
* **Context** — your full conversation right now. What you see on screen. Every message you sent, every response Claude gave, every search result, every doc. It grows every turn.
* **Baseline** — what's loaded before you type anything. Your settings, tools, MCP servers, custom instructions, rule files. Usually 15-20K tokens. This gets reread on every single turn.
* **Input** — what Claude reads on a given turn. This is your full context (everything so far) + whatever new thing it's processing.
* **Output** — what Claude writes on a given turn. The response, the tool call, the thinking step. This gets added to context for the next turn.
* **Usage (cap)** — the total tokens Claude has read + written across all turns. Anthropic limits this over a rolling window. When you hit it, you wait.

# The key insight

Context and usage are not the same thing.

**Usage** is how much total reading Claude has done. It grows fast — because Claude rereads the full context every turn. Anthropic limits this over a rolling 5-hour window. The exact limit isn't published.

**Context** is how long your conversation is. It grows slowly. It has a limit too (the context window — 200K or 1M depending on your plan). When it gets close to the limit, Claude compresses the conversation to make room (more on that later). You generally don't want to get there — compacted conversations are blurry. Claude doesn't remember the details well.

# Your first turn

You start Claude and before you say anything, you're already burning tokens. Which ones? The baseline: your settings, tools, MCP servers, custom instructions, rule files.

So we start with (for example) 18K between all of that.

| Step | What happens | Context | Usage |
|:-|:-|:-|:-|
| Start | Baseline loads | 18K | 0 |
| You | "explain this function" | 19K | 0 |
| Claude | reads 19K, writes 1K | 20K | 20K |

Let's break this down:

- You typed a message. Context grew from 18K to 19K (your message added 1K).
- Claude read the full 19K (baseline + your message), then wrote 1K.
- Context is now 20K (your message + Claude's response).
- Usage is 20K (everything Claude read + wrote this turn).

So far so good. Now you ask a follow-up:

| Step | What happens | Context | Usage |
|:-|:-|:-|:-|
| Before | | 20K | 20K |
| You | "now refactor it" | 21K | 20K |
| Claude | reads 21K, writes 2K | 23K | 43K |

Claude didn't just read your new message. It reread the entire conversation (21K) and then wrote 2K. Usage jumped from 20K to 43K. Context only grew by 3K.

That's the pattern. Context grows a little each turn. Usage grows by the **full context size** each turn.

# Why it gets more expensive over time

The same question costs more the later you ask it. Not because the question is different — because there's more context to reread.

| When you ask "fix this bug" | Context | Usage for this one question |
|:-|:-|:-|
| Turn 1 (fresh session) | 19K | 20K |
| Turn 30 | 60K | 62K |
| Turn 100 | 150K | 152K |
| Turn 250 | 480K | 482K |

Same question. Same answer. 24x more expensive at turn 250.

# Turns you don't see

When you ask Claude something short — "what does this error mean" — that's 1 turn. Claude reads your context once, responds, done.

But when you ask Claude to research something — "how do I set up auth in Next.js" — it doesn't do 1 turn. Under the hood it does many: web searches, opening docs, thinking, searching again, opening more docs, then finally responding.

You only see the answer. You don't see the 5-10 turns that happened in between.

And every single one of those invisible turns rereads your full conversation.

# Example: research at turn 30

Your context is 60K. You ask a research question. Claude does 3 rounds of work under the hood before responding:

| Step | What Claude does | Context | Usage (this research) |
|:-|:-|:-|:-|
| Before | | 60K → | 0 |
| Round 1 | searches web, opens 2 docs | → 67K | 196K |
| Round 2 | thinks, searches again, opens more | → 72K | 352K |
| Round 3 | writes your answer | → 74K | 426K |

Context: 60K → 74K. **14K of new knowledge.**

Usage: **426K.** That's 30x the new knowledge.

You asked one question. Claude reread your conversation multiple times to get you those 14K of useful information.

# Same research, different moment

Same question. Same answer. Same 14K of new stuff. But the cost depends entirely on how big your context is when you ask:

| When | Context | Usage cost | New knowledge |
|:-|:-|:-|:-|
| Turn 1 (fresh) | 18K | ~200K | 14K |
| Turn 30 | 60K | ~480K | 14K |
| Turn 100 | 150K | ~1.2M | 14K |
| Turn 250 | 480K | ~3.5M | 14K |

17x more expensive at turn 250 than at turn 1. Same answer.

# Compact

Remember: context has a limit. When you get close (~750K on a 1M window, ~150K on a 200K window), Claude compresses the conversation into a shorter summary.

Sounds helpful, and it does keep you working. But:

* **It costs usage.** To compress, Claude has to read the full context one more time — that's another full reread on the usage meter. A compact at 750K context adds 750K+ to your usage.
* **You lose detail.** The summary is shorter but blurrier. Exact code, specific outputs, nuances — some of that disappears.
* **Usage doesn't reset.** Your 5h meter keeps running. Compact only shrinks context, not usage. In fact, it increases usage because of the reread needed to compress.

Compact keeps you going, but it's not free and it's not lossless. Better to not need it.

# What you can do

* **Research early.** Before your context grows. Same knowledge, fraction of the cost.
* **Start fresh.** Switching topics? New session. Resets your context to baseline.
* **Slim your baseline.** Fewer MCPs, shorter CLAUDE.md, less stuff in rules. Every token there gets reread on every turn for the entire session.
* **Plan before executing.** Every wrong turn = a full reread wasted. Fewer mistakes = fewer turns = less usage.
* **Web search costs more, but it's real.** Claude can guess from training data (cheap, sometimes wrong) or search the web (expensive, accurate). Now you know what "expensive" actually means — and why it's still worth it.
