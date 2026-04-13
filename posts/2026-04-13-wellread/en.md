# Someone already searched that

I spend more and more time building with Claude. And doing research has become one of my most expensive hobbies.

If you don't want to burn through your context windows at lightning speed, the playbook is clear: get clarity and make decisions before executing. You need to save turns fixing errors later. (1)

You need to escape the training data trap.

The problem with using research for everything is that these processes consume a huge amount of tokens.

A research process easily takes 20 turns between web_search, fetch_url, reasoning and outputs.

And worse, Claude compounds all these turns:

```
turn 1 = fetch 1 = baseline (e.g. 10k) + 5k
turn 2 = fetch 2 = bl 10k + fetch 1 5k + fetch 2 5k
turn 3 = fetch 3 = bl 10k + f1 5k + f2 5k + f3 5k
total = 60k (not the 15k I got the first time when I forgot about compounding)
```

This gets even worse depending on when you do the research. A research in a fresh session burns 200K tokens (which is a lot). But if you do it at turn 100 (which honestly isn't that many), it easily eats 3.5M tokens.

Here's where it gets interesting. As I said above (1) "You need to save turns fixing errors later." The later you fix problems, the more expensive they are.

For context, in March I hit my 5h window cap at ~4M (still investigating — there are no official numbers and the ones out there are outdated).

But tokens are the consequence, not the problem.

If I didn't have to do research, but still had the knowledge without burning tokens, my sessions would last longer. I'd get further, faster, with fewer hallucinations and errors.

Two interesting data points:

- Between 15 and 40% of the searches we do individually are redundant (re-finding info you already found)
- Between 61 and 68% of the searches we do collectively are about the same thing

So I can't avoid research, but:

What if I could just skip the ones I already did?
What if someone else already did the research I need today?

So I built wellread.

An MCP that, before your agent searches the web, checks if someone (or you) already researched it before.

Beyond not hallucinating, not starting from scratch every time, and getting further with fewer turns: with wellread it doesn't matter when in the session you search — on a hit, what comes back to context is 1 turn and about 600 tokens. Solving (1):

| Session depth | Without wellread | With wellread |
|---|---|---|
| Turn 1 (fresh session) | 200K tokens · 10 turns · 67s | 600 tokens · 1 turn · 28s |
| Turn 30 | 1.2M tokens | 600 tokens |
| Turn 100 | 3.5M tokens | 600 tokens |
| Turn 250 | 11M tokens | 600 tokens |

I've been using it for 2 weeks. I've only hit my rate limit once (used to be 2 or 3 times a week). I've saved 60M tokens. Donated 20M tokens to the network. Ratio 3:1. For every token I contributed, wellread gives me back 3 right now.

It's not perfect. But it's free, it works, and the numbers are real.

`npx wellread`

GitHub: https://github.com/mnlt/wellread
