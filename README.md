# Story Log

A personal reading tracker for fiction with AI-powered recommendations.
Built as a single HTML file hosted on GitHub Pages, backed by Supabase.

## Features

- **Add stories** — paste any URL and the title and author are auto-filled
  via Microlink; manually edit if needed
- **Search** — filter your collection by title, author or source site
  in real time
- **Shuffle** — the list order randomises on every page load so no story
  stays buried
- **Embed** — a built-in batch tool fetches story content via Jina Reader,
  converts it to semantic vectors via Jina Embeddings, and stores both in
  Supabase; run 10 stories at a time at a respectful pace
- **Find similar** — once stories are embedded, pgvector's cosine
  similarity finds stories with matching themes, tone and style across
  English, Hindi and Telugu

## How it works

```
Add story
  └─ Microlink API ──────────────────────────────► Supabase
                                                   (title, author, link)

Embed stories
  └─ Jina Reader ─► Jina Embeddings ────────────► Supabase
     (page text)     (768-dim vector)              (content, embedding)

Find similar
  └─ Supabase pgvector (match_stories RPC) ──────► ranked results
```

Series pages (e.g. `/series/se/`) are detected automatically
by URL pattern — the embedder follows the chapter links and combines
content from the first three parts.

## Tech stack

| Layer | Tool | Cost |
|---|---|---|
| Hosting | GitHub Pages | Free |
| Database | Supabase (PostgreSQL + pgvector) | Free tier |
| Content extraction | Jina Reader API | Free |
| Semantic embeddings | Jina Embeddings v3 (89 languages, 768 dims) | 1M tokens free |
| Metadata auto-fill | Microlink API | Free tier |

## Setup

### 1. Supabase

Create a project at [supabase.com](https://supabase.com). In **SQL Editor**, run:

```sql
-- Enable vector support
create extension if not exists vector;

-- Add content and embedding columns
alter table stories
  add column if not exists content text,
  add column if not exists embedding vector(768);

-- Similarity search function
create or replace function match_stories(
  query_embedding vector(768),
  match_threshold float,
  match_count     int,
  exclude_id      bigint
)
returns table (id bigint, name text, author text, link text, score float)
language sql stable as $$
  select id, name, author, link,
    1 - (embedding <=> query_embedding) as score
  from stories
  where embedding is not null
    and id != exclude_id
    and 1 - (embedding <=> query_embedding) > match_threshold
  order by score desc
  limit match_count;
$$;
```

In **Authentication → Policies**, add public RLS policies for SELECT,
INSERT, UPDATE and DELETE on the `stories` table.

### 2. API keys

- **Supabase** — Project URL and anon key from Settings → API
- **Jina** — Free key from [jina.ai](https://jina.ai) (1M tokens included,
  no credit card required)

### 3. Configure and deploy

Open `index.html` and replace the two placeholder values near the top of
the `<script>` block:

```js
const SUPABASE_URL  = "https://your-project-id.supabase.co";
const SUPABASE_ANON = "your-anon-public-key";
```

Push to GitHub, then enable GitHub Pages in **Settings → Pages → Source:
main branch**. The site is live in about 60 seconds.

> The anon key is safe to commit — access is controlled entirely by
> Supabase RLS policies, not by the key itself.

## Embedding stories

1. Open your live site and scroll to **⚙ Embedding Tools**
2. Paste your Jina API key (saved in localStorage after first use)
3. Click **Embed next 10 stories**
4. Repeat daily or in one session — built-in 2-second delays between
   requests keep usage well within the free rate limits

## Notes

- Recommendations improve significantly once 30–40 stories are embedded;
  the full collection gives the best results
- Series detection targets Literotica URL patterns by default; additional
  site patterns can be added incrementally to `isSeriesUrl()`
- The embedding model (Jina v3) handles transliterated Indic text
  reasonably well alongside native-script Hindi and Telugu

## License

MIT
EOF
