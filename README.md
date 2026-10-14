# Yaris — AI-powered humanitarian intelligence

A functional recreation of the GANNET-class humanitarian assistant: a RAG-powered
virtual assistant that retrieves from a verified-source corpus, grounds its answers
in the retrieved evidence, cites every claim inline to the exact source fragment,
answers in English / French / Arabic, and attaches structured deep-dive figures from
the HDX Humanitarian API (HAPI).

> Demonstration project with simulated data. Not affiliated with GANNET.

## Quick start

```bash
npm install
npm run dev          # → http://localhost:3000
```

No API keys required — without `LLM_API_KEY` the pipeline runs a deterministic
template generator, and without HAPI credentials deep dives use the bundled demo
datasets. The whole system works fully offline.

Verify the pipeline without even installing dependencies:

```bash
node scripts/smoke.mjs
```

## Configuration (optional)

Copy `.env.example` to `.env.local`:

| Variable | Purpose |
| --- | --- |
| `LLM_BASE_URL` / `LLM_API_KEY` / `LLM_MODEL` | Any OpenAI-compatible chat API for generative grounded answers |
| `HAPI_APP_NAME` + `HAPI_APP_EMAIL` | HAPI `app_identifier` (base64 of `name:email`) for live deep-dive data |
| `HAPI_BASE_URL` | Defaults to `https://hapi.humdata.org` (API v2) |
| `YARIS_OFFLINE=1` | Skip all HAPI network calls |

## How the pipeline works

```
question ─▶ retrieve ─▶ deep dive ─▶ grounded generation ─▶ citation validation
            (BM25 over               (live HAPI,            (OpenAI-compatible,
             verified                demo fallback)          template fallback)
             fragments)
```

1. **Retrieve** — BM25 over document fragments in `data/sources.json` (title-boosted,
   stopword-filtered, Unicode-aware for Arabic/French), with cross-lingual query
   expansion (FR/AR crisis names → English aliases) while the demo corpus is
   English-only. To swap in a vector store (pgvector, Pinecone…), reimplement
   `retrieve()` in `src/lib/retriever.mjs`.
2. **Deep dive** — `src/lib/hapi.mjs` queries HAPI (`/api/v2/affected-people/idps`,
   `/food-security-nutrition-poverty/food-security`, `/coordination-context/funding`)
   for the detected topic and normalizes the latest reference period into bars/tiles.
   Any failure falls back to `data/hapi-demo.json`.
3. **Generate** — the LLM receives only the top fragments and is instructed to cite
   claims with `{{c:N}}` tokens and to refuse when the sources do not cover the
   question. Invalid citation indices are stripped (`src/lib/citations.mjs`).
4. **Respond** — `POST /api/chat` returns `{ answer, citations, retrieval, deepDive, demo }`;
   the React UI renders streaming-style retrieval cards, clickable citation chips
   that expand the exact source fragment, and the collapsible deep-dive chart.

If the best retrieval score is below threshold, Yaris answers with a
"my verified sources do not cover that yet" message instead of hallucinating.

## Project layout

```
data/               verified-source corpus + demo deep-dive datasets
scripts/smoke.mjs   offline end-to-end pipeline test (bare Node 18+)
src/lib/            retriever, citations, hapi client, llm client, pipeline
src/app/            Next.js App Router page + /api/chat route
src/components/     Chat, Thread, Message, DeepDive, RetrievalCard
```

## Extending

- **More sources**: append documents to `data/sources.json` — no code changes needed.
- **Vector retrieval**: implement the same `retrieve(units, query, k)` signature.
- **Auth / SSO**: put the app behind your identity provider (e.g. NextAuth) — the
  pipeline is stateless and user-agnostic.
- **Deployment**: `npm run build` then deploy to any Node host, or push to Vercel
  for a public URL in one click.
