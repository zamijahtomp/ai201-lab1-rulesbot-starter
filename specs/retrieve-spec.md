# Spec: `retrieve()`

**File:** `retriever.py`
**Status:** Spec incomplete — fill in all blank fields before implementing

---

## Purpose

Given a user's natural language query, find the most relevant chunks from the vector store using semantic similarity search. Return them ranked by relevance so that `generate_response()` can use them as context.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | `str` | The user's natural language question |
| `n_results` | `int` | Maximum number of chunks to return (default: `N_RESULTS` from `config.py`) |

**Output:** `list[dict]`

Each dict in the returned list must contain exactly these keys:

| Key | Type | Description |
|-----|------|-------------|
| `"text"` | `str` | The chunk text |
| `"game"` | `str` | The game name this chunk came from |
| `"distance"` | `float` | Cosine distance score — lower means more similar to the query |

Results should be ordered from most to least relevant (lowest to highest distance). Returns an empty list `[]` if the collection contains no documents.

---

## Design Decisions

*Complete the fields below before writing any code. Use your AI tool in Plan or Ask mode to help you reason through what belongs here — but the decisions are yours.*

---

### Query approach

*Describe how you will use `_collection.query()` to find relevant chunks. What arguments will you pass, and why?*

```
[your answer here]
```

---

### Return structure

```txt
One item looks like:

{
    "text": "The longest road award goes to the player with at least 5 connected roads...",
    "game": "Catan",
    "distance": 0.31,
}

- "text" comes from results["documents"][0][i]
- "game" comes from results["metadatas"][0][i]["game"]
- "distance" comes from results["distances"][0][i]

where i is the rank of the result (0 = most similar). Chroma already
returns results ordered by ascending distance, so building the list by
iterating i in order preserves "most to least relevant" without any
extra sorting.
```

---

### Handling the nested result structure

```txt
results["documents"], results["metadatas"], and results["distances"] are
each a list-of-lists: the outer list has one entry per query string
passed in query_texts, and the inner list has one entry per result for
that query. Since retrieve() only ever passes a single query string
(query_texts=[query]), the results I want are always at index [0] of
the outer list, e.g. results["documents"][0] is the list of chunk texts
for my one query. I then zip/iterate the three [0] lists together (by
index i) to build one dict per result.

The nesting exists because query_texts accepts multiple queries in one
call, batching several searches in a single round-trip to the
embedding model and the index.
```

---

### Relevance threshold

```txt
No threshold — return all n_results as-is, ranked by distance.

Rationale: with only 8 rule books and a few hundred chunks total, even
the "worst" of n_results=3-5 results is usually still topically
related, and generate_response() (the LLM) is well-suited to judge
relevance and say "I don't see that rule" if the retrieved context
doesn't actually answer the question. A fixed distance cutoff is risky
to tune without much data: too strict and it silently drops the one
useful chunk for a valid question (false negative → app looks broken
for no reason); too loose and it does nothing.

Tradeoff: no threshold means low-quality/no-match queries still get
n_results context stuffed into the prompt, which could nudge the LLM
toward a plausible-but-wrong answer instead of "I don't know." If that
becomes a problem in testing, the cheap fix is to add a cutoff (e.g.
distance > 1.0 gets dropped) rather than building it in upfront.
```

---

### Edge cases

*How does your implementation behave when: (a) the collection is empty, (b) the query matches no chunks well, (c) the query matches chunks from multiple games?*

```
[your answer here]
```

---

## Implementation Notes

**Test query and top result returned:**

```txt
Query: "How many points to win?"
Top result game: Uno
Distance score: 0.323
Does it make sense? Yes — top 3 results all came from Uno's "WINNING
THE GAME" section (first to 500 points across rounds). Makes sense
since "points to win" maps almost literally onto Uno's scoring rule,
and no other rule book in the set uses a points-based win condition.
```

**One thing about the query results that surprised you:**

```txt
"What happens when you roll a 7?" returned Catan's actual "ROLLING A 7"
rule (discard half, move robber, steal a card) at distance 0.466 — much
higher than the "points to win" example (0.323), even though it's a
clean, correct top-1 match ahead of unrelated Risk chunks at 0.597+.
Distance isn't a fixed pass/fail threshold — it depends on how closely
the query's wording overlaps with the rule book's phrasing (e.g. "roll
a 7" vs. the heading "ROLLING A 7" plus prose describing it). What
matters is relative ranking within a query, not an absolute cutoff
across queries.

"How do you win?" returned chunks from four different games (Catan,
Ticket to Ride, Pandemic, Monopoly) — see the multi-game analysis
above. This isn't a bug: the query never named a game, so surfacing
each game's genuinely relevant winning-conditions chunk is the correct
behavior for an ambiguous, cross-game question.
```
