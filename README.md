# realm-zep-memory

[Zep Cloud](https://www.getzep.com/)'s temporal knowledge graph under the shared `Memory` label.
Zep extracts facts from a user's conversations and data and keeps, for each, the window in which
it held. This realm recalls those facts through Zep's own graph search, so one Virtual Cypher
query answers from Zep alongside the assistant's own memory and every other memory system.

```cypher
MATCH (:Concept {value:'coffee'})-[r:RELEVANT_TO]->(m:Memory)     -- every memory system
RETURN m.source, m.text, m.validFrom, m.validTo, r.score ORDER BY r.rank

MATCH (:Concept {value:'coffee'})-[r:RELEVANT_TO]->(m:ZepMemory)  -- Zep alone
RETURN m.text, m.name, m.validFrom, m.validTo, r.score ORDER BY r.rank

MATCH (:Concept {value:'employer'})-[:RELEVANT_TO]->(m:ZepMemory)  -- what was true in March
WHERE m.validFrom <= '2026-03-01' AND (m.validTo IS NULL OR m.validTo > '2026-03-01')
RETURN m.text
```

Every node carries `:Memory`, `:ZepMemory` and the realm label `:Zep`.

## What it needs

A key, declared in `keys.yml` and bound in the world (`set ZEP_API_KEY = …` in chat, or the
realm's settings page), and a user id, set in the realm:

| Where | What |
|---|---|
| `ZEP_API_KEY` (key) | A Zep Cloud API key. Sent as `Authorization: Api-Key <key>`. |
| `user_id` in `producers/zep.yml` | The Zep user whose graph this world reads. Zep addresses a user's graph by `user_id`; this realm reads one. A request-body value is not yet interpolated from the credential store the way a header's `${VAR}` is, so it is edited in place. |

Without the key the API is skipped at world load and the edge answers empty, reported.

## The contract

| Memory property | From Zep |
|---|---|
| `id` | the edge's `uuid` |
| `text` | `fact` |
| `name` | the relation name |
| `status` | `ACTIVE` while Zep holds the fact; `SUPERSEDED` once `invalid_at` or `expired_at` is set |
| `validFrom` / `validTo` | `valid_at` / `invalid_at` |
| `created` / `revised` | `created_at` / `expired_at` |
| `r.score` | Zep's reranked score, unrescaled (`recordScoreField`) |

A plain read returns ACTIVE facts; pin `{status:'SUPERSEDED'}` on the node to read what Zep no
longer holds. That is the `Memory` lifecycle, declared once in the default world and inherited here.

## Graphiti

Zep's graph engine is [Graphiti](https://github.com/getzep/graphiti), and a self-hosted Graphiti
server is the same contract without the key: see `realm-graphiti-memory`.

## Status

Built from Zep's v2 API reference. Not yet exercised against a live Zep Cloud account; the
Graphiti sibling, which shares the fact shape, is.
