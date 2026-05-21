# LocalLens — Presentation Outline & Talking Points

> Slide-by-slide outline for a ~20–25 min project showcase to a **mixed audience**
> (engineers + non-technical stakeholders). Each slide has **Slide content** (what to
> put on the slide) and **Talking points** (what to say). Keep slides skimmable; let the
> talking points carry the depth.

---

## Slide 1 — Title

**Slide content**
- **LocalLens — AI-Powered Local Discovery Agent**
- One plain-English question → a ranked, reviewed, *explained* shortlist of local places
- Your name · date

**Talking points**
- "LocalLens is a conversational agent that sits between a user and the messy web. You ask one normal-language question, and it does the whole research job for you — understands what you want, figures out where you are, searches several sources, reads the reviews, scores everything, and explains the top picks in plain English."

---

## Slide 2 — The Problem

**Slide content**
- Finding the best local option today means:
  - Wading through Google ads + SEO-gamed results
  - Cross-referencing Maps, Yelp, TripAdvisor by hand
  - Reading dozens of inconsistent reviews
  - Starting over when the first pick is closed / too pricey / irrelevant
- The user is doing all the orchestration manually

**Talking points**
- "Today *you* are the integration layer. You open five tabs, mentally merge them, and form an opinion. That's slow and noisy. LocalLens collapses that into one query."

---

## Slide 3 — What LocalLens Does

**Slide content**
- One query, one shot:
  1. Understands the request (category, count, location, filters)
  2. Resolves the location to coordinates
  3. Searches multiple live sources
  4. Reads & analyzes reviews
  5. Scores and ranks
  6. Writes a grounded 2–3 sentence summary per result
- Output: name · address · score · hours · summary

**Talking points**
- "The promise is "ask once, get the best options ranked and explained." Everything is live — there's no static database; every query re-fetches fresh data."

---

## Slide 4 — Demo / Sample Queries

**Slide content**
- "Find me the best 3 sushi restaurants near me"
- "5 tax filing companies in Irvine CA"
- "Best co-working spaces in Dhanmondi, Dhaka"
- "Top 4 grocery stores near zip code 10001"
- "Affordable yoga studios near downtown Seattle"
- (Live) submit one and watch the agent timeline stream

**Talking points**
- "Let me run one live. Notice the horizontal agent timeline — each stage lights up as it runs and shows its input and output. That transparency is a core UX choice; the user can see *why* they got these results."

---

## Slide 5 — High-Level Architecture

**Slide content**
- 7-stage pipeline orchestrated with **LangGraph**, streamed over **SSE**:
  - Intent → Location → Search → Reviews → Scoring → Summary → Format
- Every stage emits a structured `PipelineEvent` (input, output, duration)
- Diagram of the linear flow

**Talking points**
- "Each layer is an independently testable agent. They're wired into a LangGraph state machine, and after every node we push an event to the browser, which is how the live timeline works."

---

## Slide 6 — Stage Walkthrough (A–D)

**Slide content**
- **A. Intent Parser** — LLM turns the query into a typed `ParsedIntent` (category, count, location, radius, sort, filters)
- **B. Location Resolver** — "near me" via ip-api; named places via Nominatim → bounding box
- **C. Search & Crawl Agent** — multiple sources in parallel (next slide)
- **D. Review Aggregator** — Playwright scrape + HuggingFace distilbert sentiment + theme tags + recency weighting

**Talking points**
- "Intent parsing is one of only two places we use the LLM. Location resolution handles all four input types — near-me, city, zip, neighborhood. The review aggregator runs a local sentiment model over scraped reviews and flags low-data results."

---

## Slide 7 — Stage Walkthrough (E–G)

**Slide content**
- **E. Scoring Engine** — composite 0–100 score, weighted signals, top-N by `intent.count`
- **F. Summary Generator** — grounded 2–3 sentence summary per result (the other LLM use)
- **G. Response Formatter** — final card: name · address · score · hours · summary · low-data badge

**Talking points**
- "Scoring normalizes every signal to 0–100 before weighting. The summary generator is told to use *only* the provided context. The formatter assembles the card the user sees."

---

## Slide 8 — Key Design Principle (the anti-hallucination decision)

**Slide content**
- The LLM is used for **intent parsing** and **final summarization** — and nothing else
- It is **never** used to retrieve facts
- All facts originate from Overpass / scraping / Maps

**Talking points**
- "This is the most important decision in the whole system. If you let an LLM 'find' businesses, it invents plausible-sounding fake ones — wrong addresses, made-up hours. By keeping it strictly to understanding the question and phrasing the answer, the model literally cannot fabricate a venue."

---

## Slide 9 — Multi-Source Search + Fallbacks

**Slide content**
- Sources run **concurrently** (`asyncio.gather`):
  - Overpass (OpenStreetMap) — structured ground truth
  - DuckDuckGo, Bing, Google, **Google Maps** (canonical + raw query) via Playwright
- Each Playwright source self-detects bot blocks → returns empty, never crashes
- Dedup merges the same venue across sources by name + coordinates

**Talking points**
- "OSM is great in Western cities and sparse elsewhere — searching Dhaka returned almost nothing useful at first. Adding Bing and Google Maps fixed regional relevance. Google blocks bots aggressively, so every web source degrades gracefully — if one gets blocked, the others carry the query."

---

## Slide 10 — Scoring & Ranking

**Slide content**
- Composite score = **0.35** star + **0.25** review count + **0.20** sentiment + **0.10** recency + **0.10** semantic match
- All weights live in YAML + a runtime slider overlay — **no magic numbers in code**
- Category/filter overrides (e.g. "affordable" tilts the weights)
- Semantic match = ChromaDB + MiniLM cosine similarity to the query intent

**Talking points**
- "Weights are fully externalized — you can retune ranking from the UI sliders without touching code. The semantic-match signal helps fuzzy queries like 'cozy date-night spots' where keyword tags fail."

---

## Slide 11 — Trust: Grounding & Relevance Guards

**Slide content**
- `verify_grounding` — flags any summary claim not present in the source text
- Relevance filter — heuristic + LLM, **strict mode** on High/Max
- Final **place validator** — drops blog / listicle / article entries from the top-N
- Low-confidence ("limited data") badge when reviews < 5 / no rating

**Talking points**
- "Layered defense. Heuristics catch obvious junk cheaply, the LLM catches the subtle 'Top 10 sushi in Dhaka' blog post, and a final validator double-checks that what reaches the user is a real, visitable venue — not an article about venues."

---

## Slide 12 — Effort Tiers (the latency vs. quality dial)

**Slide content**
- **Low** (~3–7s): Overpass + DuckDuckGo only, no reviews
- **Medium** (~17–35s): + Bing, listicle expansion, reviews, sentiment, semantic match
- **High** (~30–60s): Maps-focused (canonical + raw), strict relevance, place validator
- **Max** (~55–100s): every source, largest caps — the kitchen sink
- User picks per query in the UI

**Talking points**
- "Latency is a product decision, not a constant. Low is for speed; Max is for thoroughness. High is deliberately Google-Maps-focused and clean. Same pipeline, different knobs — all runtime-switchable, no restart."

---

## Slide 13 — The UI

**Slide content**
- 3-column chat layout: history · conversation · scoring sliders
- Live **horizontal agent timeline** per message (each agent's input + output)
- Effort selector · model switcher · voice input (Whisper) · retry on failure
- Server-side chat history with conversational memory ("show me cheaper ones")

**Talking points**
- "The UI's job is transparency. You don't just get an answer — you watch the agents work, can tune the ranking live, and follow-up queries remember context within a chat."

---

## Slide 14 — Tech Stack (100% free & open-source)

**Slide content**
- Python 3.11 · FastAPI + uvicorn · LangGraph · SSE
- LLM: Ollama (local) **or** Groq (hosted free tier) — env switch
- Data: Overpass · Nominatim · ip-api · DuckDuckGo · Playwright + BeautifulSoup
- ML: HuggingFace distilbert (sentiment) · MiniLM (semantic) · whisper-tiny (voice) · ChromaDB
- Ops: diskcache · Langfuse · Docker Compose

**Talking points**
- "Zero cost to run and no API keys required by default. The LLM provider is a one-line env switch between a local Ollama box and Groq's free tier."

---

## Slide 15 — Observability & Testing

**Slide content**
- **Langfuse** traces every agent + LLM call (no silent calls)
- SSE timeline = live in-product observability
- **22 unit-test files** + integration tests
- **4 eval notebooks**: intent accuracy, scoring formula, summary grounding, intent classifier
- Full stack via `docker compose up`

**Talking points**
- "Production-grade: every model call is traced and reproducible, the pipeline is covered by tests, and quality is measured in notebooks — not vibes."

---

## Slide 16 — Stretch Goals: all 11 shipped

**Slide content**
- Infra: async parallel crawling · docker-compose stack · semantic search · SSE frontend
- Data quality: low-data detection · cross-source duplicate merging · category auto-tagging
- Advanced AI: conversational memory · voice input (Whisper) · BERT intent classifier · clarifying questions

**Talking points**
- "Every stretch goal from the original scope doc is implemented — from async crawling to a voice composer to a small BERT-family intent classifier."

---

## Slide 17 — Lessons Learned

**Slide content**
- Keep the LLM **out of factual retrieval** → kills hallucination at the source
- **Bot-blocking is real** — Google blocks Playwright hard; answer is graceful degradation + stealth + multi-source redundancy
- **Regional data gaps** demand multiple sources (the Dhaka problem)
- **Latency is a product decision** → effort tiers, parallelism, deferring expensive work to survivors
- **Small training sets overfit** — a 31-example kNN "classifier" is a cross-check, not a replacement; be honest about model limits
- **Config over code** — YAML weights + runtime overlays make tuning safe
- *(Optional)* What's next: price-aware ranking · residential-proxy Maps · fine-tuned intent model

**Talking points**
- "The biggest lesson is architectural discipline: the LLM is powerful but the moment you let it touch facts, it lies confidently. The second is that the open web fights back — bot detection, sparse data in some regions — so resilience and redundancy matter more than any single clever source."

---

## Slide 18 — Closing / Q&A

**Slide content**
- LocalLens: one question → ranked, grounded, explained local results — built on 100% free tooling
- Thank you · Q&A
- (Optional) repo / demo link

**Talking points**
- "To recap: a 7-stage agent pipeline, multiple resilient data sources, grounded summaries with hallucination guards, a tunable latency/quality dial, and full observability — all open-source. Happy to dive into any stage."

---

### Presenter notes
- For a **technical** crowd, expand slides 8–12. For a **non-technical** crowd, lead with 2–4 and 13, compress 6–7 into one.
- If converting to a real deck later: one diagram on slide 5, one screenshot on slide 13, keep the rest bullets.
