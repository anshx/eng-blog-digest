# Engineering Blog Digest

A daily learning archive. Every morning, one engineering blog post is selected
for its teaching value — a transferable technique, abstraction, or mental
model — and written up pedagogically.

## Each post captures

- **What's new to learn** — concepts/techniques/mental models the post introduces
- **Prerequisites** — what you need to understand first
- **The core idea** — the central technique, taught in plain language
- **Mechanics** — the how, in detail (algorithm, architecture, code, config)
- **Where it breaks** — limits, tradeoffs, what doesn't generalize
- **Why it works** — the deeper principle ("X is just an instance of Y")
- **Going deeper** — 2-3 follow-up reads

## Sources scanned

- AI/LLM infra: Anthropic, OpenAI, Modal, LangChain, LlamaIndex, Pinecone
- Data/analytics infra: Databricks, Snowflake, ClickHouse, DuckDB, dbt, Redshift
- FAANG-tier eng: Netflix, Uber, Stripe, Cloudflare, Discord, Figma
- Hacker News top engineering posts

Selection bias: prefer transferable concepts, non-obvious decisions, and
"oh, so X is just an instance of Y" reveals. Skip announcements, marketing,
hiring posts, polished retellings of well-known ideas, opinion pieces, and
shallow tutorials.

## Layout

```
digests/
  index.md          # one-line entry per day, links to the deep dive
posts/
  YYYY-MM-DD/
    <slug>.md       # the deep dive itself
```

## Cadence

Updated daily at **06:00 AM IST** by a scheduled remote Claude Code agent
(routine `eng-blog-digest`, model `claude-sonnet-4-6`). Days where nothing
meets the bar produce no commit.
