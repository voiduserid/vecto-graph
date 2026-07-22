# vecto-graph — Build Plan

## Context

The repo currently contains only a README.md describing the concept of a "Graph-RAG Recommender": vector search finds a seed node, graph traversal expands multi-hop relationships from it, and an LLM turns the resulting subgraph into a natural-language recommendation with an explicit, auditable reasoning path. There is no code yet. The goal now is to build this as a real, running system (not a notebook) — a service you can send a query to and get back a recommendation plus the graph path that justified it.

Stack decisions made with the user:
- **Language**: Python
- **Graph DB**: Neo4j, using its native vector index (5.11+) so embeddings and graph structure live in one database — no separate vector store
- **Dataset**: MovieLens (`ml-latest-small` — 9,742 movies, 610 users, 100k ratings; matches the README's Interstellar/Inception example, fast to iterate on)
- **LLM**: OpenAI (`gpt-4o-mini` for generation, `text-embedding-3-small` for embeddings)
- **Enrichment**: MovieLens has no director/composer/cast data, so we enrich via the free TMDb API (director, composer, top-5 cast per movie), cached to disk, one-time ~30-40 min run — this is what makes the README's exact "Interstellar → directed by Nolan → Inception" example real and demoable, not just illustrative.

v1 scope is intentionally minimal: no auth, no multi-tenancy, no scaling concerns — just a correct, working end-to-end pipeline.

---

## 1. Project structure (src layout)

```
vecto-graph/
├── pyproject.toml
├── .env.example
├── .gitignore
├── docker-compose.yml
├── Dockerfile
├── data/                        # gitignored
│   ├── raw/ml-latest-small/
│   └── cache/tmdb_credits.json  # cached TMDb responses, resumable
├── src/vecto_graph/
│   ├── config.py                 # pydantic-settings Settings
│   ├── db/neo4j_client.py        # driver singleton + session helpers
│   ├── ingestion/
│   │   ├── download.py           # fetch + unzip ml-latest-small (idempotent)
│   │   ├── tmdb_enrich.py        # director/composer/cast via TMDb, cached
│   │   ├── embed.py              # build embedding text, call OpenAI embeddings
│   │   ├── schema.py             # constraints + vector index Cypher
│   │   ├── load_neo4j.py         # batched UNWIND/MERGE writes
│   │   └── run_ingestion.py      # CLI entrypoint chaining all steps
│   ├── retrieval/
│   │   ├── vector_search.py      # embed query, db.index.vector.queryNodes
│   │   ├── graph_traversal.py    # multi-hop Cypher from seed node(s)
│   │   └── subgraph.py           # serialize Path objects -> JSON + audit trail
│   ├── generation/
│   │   ├── prompt.py             # prompt template
│   │   └── llm.py                # OpenAI chat completion call
│   ├── pipeline.py                # recommend(query) orchestrates retrieval+generation
│   └── api/
│       ├── main.py                # FastAPI: POST /recommend, GET /health
│       └── schemas.py
└── scripts/
    ├── sanity_check.py            # Cypher count checks post-ingestion
    └── test_retrieval.py          # manual retrieval+generation test, no HTTP
```

Core deps: `neo4j` (official driver), `openai`, `fastapi`, `uvicorn[standard]`, `pydantic-settings`, `httpx`, `pandas`, `tenacity`, `tqdm`. `.gitignore` covers `.env`, `data/`, `.venv/`, `__pycache__/`.

## 2. Neo4j schema

**Nodes**: `Movie {movieId, title, year, tmdbId, imdbId, genres[], embeddingText, embedding}`, `Genre {name}`, `User {userId}`, `Person {personId, name}` (directors/composers/cast share this label).

**Relationships**: `(User)-[:RATED {rating, timestamp}]->(Movie)`, `(User)-[:TAGGED {tag, timestamp}]->(Movie)`, `(Movie)-[:IN_GENRE]->(Genre)`, `(Person)-[:DIRECTED]->(Movie)`, `(Person)-[:COMPOSED]->(Movie)`, `(Person)-[:ACTED_IN {character}]->(Movie)` (top-5 billed only).

```cypher
CREATE CONSTRAINT movie_id IF NOT EXISTS FOR (m:Movie) REQUIRE m.movieId IS UNIQUE;
CREATE CONSTRAINT user_id IF NOT EXISTS FOR (u:User) REQUIRE u.userId IS UNIQUE;
CREATE CONSTRAINT genre_name IF NOT EXISTS FOR (g:Genre) REQUIRE g.name IS UNIQUE;
CREATE CONSTRAINT person_id IF NOT EXISTS FOR (p:Person) REQUIRE p.personId IS UNIQUE;

CREATE VECTOR INDEX movieEmbeddingIndex IF NOT EXISTS
FOR (m:Movie) ON (m.embedding)
OPTIONS { indexConfig: { `vector.dimensions`: 1536, `vector.similarity_function`: 'cosine' } };
```

`embeddingText` per movie = `"{title} ({year}). Genres: {genres}. Tags: {top user tags from tags.csv}."` — self-contained, no extra API needed beyond calling OpenAI to embed it.

## 3. Ingestion pipeline (repeatable CLI, not a notebook)

1. `download.py` — fetch `ml-latest-small.zip` from GroupLens, unzip to `data/raw/`, skip if already present.
2. Parse `movies.csv`, `ratings.csv`, `tags.csv`, `links.csv` with pandas.
3. `tmdb_enrich.py` — for each `tmdbId` in `links.csv`, call TMDb `GET /movie/{tmdbId}/credits`. Extract `job == "Director"`, `job == "Original Music Composer"`, cast with `order < 5`. Cache every response to `data/cache/tmdb_credits.json` keyed by `tmdbId` (resumable — reruns skip cached ids). `tenacity` retry on transient errors; skip gracefully on missing/null tmdbId or 404.
4. `embed.py` — build `embeddingText`, call `openai.embeddings.create(model="text-embedding-3-small", input=[...])` in batches of ~500.
5. `schema.py` — run constraints + vector index Cypher once before any writes.
6. `load_neo4j.py` — batched `UNWIND`/`MERGE` writes via `driver.execute_query(...)`, batch size ~1000, idempotent (safe to rerun).
7. `run_ingestion.py` — `python -m vecto_graph.ingestion.run_ingestion` chains all of the above with `tqdm` progress and a final node/relationship count summary.

## 4. Retrieval pipeline

- **Embed query** with the same model as ingestion.
- **Vector search** for seed node(s): `CALL db.index.vector.queryNodes('movieEmbeddingIndex', $topK, $queryEmbedding) YIELD node, score`.
- **Multi-hop traversal** from the seed: separate, explicit Cypher queries (not one giant query) for director-overlap, composer-overlap, genre-overlap, and co-rated-highly-by-similar-users, each returning real `Path` objects. Merge and rank candidates in Python: +2 shared director, +2 shared composer, +1 per shared genre, +1 per distinct co-rating user (capped). Take top 3-5.
- **Serialize** into `{seed, candidates: [{movie, reasons[], path[]}]}` — used both as LLM context and as the API's audit trail.

## 5. Generation

Fixed prompt template instructing the model to recommend exactly one candidate from the provided list and explain why, citing only the given relationships (no invented facts). `openai.chat.completions.create(model="gpt-4o-mini", temperature=0.4)`. `pipeline.py` exposes `recommend(query) -> RecommendationResult` as the single entrypoint used by both the test script and the API.

## 6. Serving layer

FastAPI (`api/main.py`): `POST /recommend {"query": str, "top_k": int=5}` → runs `pipeline.recommend`, returns recommendation text + seed movie + candidates + graph path. `GET /health` checks `driver.verify_connectivity()`.

`docker-compose.yml`: `neo4j` service (`neo4j:5.26-community`, APOC enabled, ports 7474/7687) + `app` service (builds from repo `Dockerfile`, runs uvicorn, depends on neo4j, reads `.env`). Ingestion runs as a one-off (`docker compose run app python -m vecto_graph.ingestion.run_ingestion`), not baked into the always-on container.

## 7. Config/secrets

`config.py` — `pydantic-settings` `BaseSettings` reading `.env`: `neo4j_uri`, `neo4j_user`, `neo4j_password`, `openai_api_key`, `openai_embedding_model`, `openai_chat_model`, `tmdb_api_key`. `.env.example` committed with placeholders only; `.env` gitignored; never hardcoded.

## 8. Milestones (each independently verifiable)

- **Phase 0** — Scaffold (`pyproject.toml`, `src/` layout, `.env.example`, `.gitignore`) + `docker-compose.yml` with just `neo4j`. Verify: `docker compose up -d neo4j`, reach `http://localhost:7474`.
- **Phase 1** — Full `ingestion/` + run it. Verify: counts + vector index exist + Interstellar has `DIRECTED`-by-Nolan and `COMPOSED`-by-Zimmer edges (queries below).
- **Phase 2** — `retrieval/` + `scripts/test_retrieval.py`. Verify: query "sci-fi thriller like Interstellar" returns Interstellar as seed, Inception as top candidate with Nolan/Zimmer reasons.
- **Phase 3** — `generation/` + `pipeline.py`. Verify: generated text recommends Inception citing Nolan/Zimmer.
- **Phase 4** — `api/` FastAPI service running locally against the already-running Neo4j. Verify via curl.
- **Phase 5** — Add `app` service + `Dockerfile` to compose, full stack up. Verify same curl test end-to-end containerized.

## 9. Verification

Phase 1 sanity checks (`scripts/sanity_check.py`):
```cypher
MATCH (m:Movie) RETURN count(m);
MATCH (u:User) RETURN count(u);
MATCH ()-[r:RATED]->() RETURN count(r);
MATCH ()-[r:DIRECTED]->() RETURN count(r);
SHOW INDEXES YIELD name, type WHERE type = 'VECTOR';
MATCH (:Movie {title: 'Interstellar (2014)'})<-[:DIRECTED]-(p:Person) RETURN p.name;
MATCH (:Movie {title: 'Interstellar (2014)'})<-[:COMPOSED]-(p:Person) RETURN p.name;
```

Phase 2/3: `python scripts/test_retrieval.py "Recommend me sci-fi thrillers like Interstellar"` — assert seed contains "Interstellar", Inception appears with a Nolan/Zimmer reason, generated text reproduces the README's example.

Phase 4/5 end-to-end:
```bash
curl -X POST http://localhost:8000/recommend \
  -H "Content-Type: application/json" \
  -d '{"query": "Recommend me sci-fi thrillers like Interstellar"}'
```
Confirm response mentions Inception + Nolan/Zimmer and `graph_path` lists the explicit traversal edges — the README's "auditable logic" made concrete.

### Critical files
- `src/vecto_graph/ingestion/run_ingestion.py`
- `src/vecto_graph/ingestion/tmdb_enrich.py`
- `src/vecto_graph/ingestion/schema.py`
- `src/vecto_graph/retrieval/graph_traversal.py`
- `src/vecto_graph/pipeline.py`
- `docker-compose.yml`
