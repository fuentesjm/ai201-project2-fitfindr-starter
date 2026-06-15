# FitFindr 🛍️

FitFindr is an agent that helps you shop secondhand. You describe what you're
looking for in plain language; it searches a listings dataset, styles the top
find against your wardrobe, and writes a shareable "fit card" caption for it.

A complete interaction runs through three tools in sequence, passing state
through a single session dictionary, and branches cleanly when no listing
matches the request.

## Setup

```bash
pip install -r requirements.txt
```

Set your Groq API key in a `.env` file (get a free key at [console.groq.com](https://console.groq.com)):
```
GROQ_API_KEY=your_key_here
```

## Running It

**Web UI (Gradio):**
```bash
python app.py
```
Then open the localhost URL shown in your terminal (usually http://localhost:7860).

**Command line (runs the two built-in test cases):**
```bash
python agent.py
```

**Tests:**
```bash
python -m pytest tests/ -v
```

---

## Tool Inventory

The agent has three tools, defined in [`tools.py`](tools.py).

### 1. `search_listings`

| | |
|---|---|
| **Purpose** | Find listings in the dataset that match the shopper's request. |
| **Inputs** | `description: str` (keywords, e.g. `"vintage graphic tee"`), `size: str \| None` (size filter, case-insensitive substring — `"M"` matches `"S/M"`), `max_price: float \| None` (inclusive price ceiling) |
| **Output** | `list[dict]` — matching listing dicts sorted by relevance (best first). Returns `[]` when nothing matches; never raises. |

It loads the data with `load_listings()`, filters by price and size, scores each
remaining listing by keyword overlap across its title, description, category,
style tags, and colors, drops zero-score listings, and sorts highest-score-first.

### 2. `suggest_outfit`

| | |
|---|---|
| **Purpose** | Style the chosen item into one or two complete outfits. |
| **Inputs** | `new_item: dict` (a listing dict — the item being considered), `wardrobe: dict` (a wardrobe with an `"items"` list; may be empty) |
| **Output** | `str` — a non-empty outfit suggestion. |

This tool calls the LLM (Groq `llama-3.3-70b-versatile`). With a populated
wardrobe it asks for outfits that pair the new item with **named pieces** from
the closet. With an empty wardrobe it falls back to general styling advice for
the item alone, so a new user never gets a crash or an empty response.

### 3. `create_fit_card`

| | |
|---|---|
| **Purpose** | Write a short, shareable social caption ("fit card") for the outfit. |
| **Inputs** | `outfit: str` (the suggestion from `suggest_outfit`), `new_item: dict` (the listing dict) |
| **Output** | `str` — a 2–4 sentence OOTD-style caption, or a descriptive error string if `outfit` is empty. |

Also LLM-backed. It mentions the item name, price, and platform once each, keeps
the tone casual, and uses `temperature=1.0` so repeated runs on the same input
produce different captions.

> Note on argument order: the signature is `create_fit_card(outfit, new_item)` —
> outfit first. The agent calls it in that order.

---

## Planning Loop

The loop lives in `run_agent(query, wardrobe)` in [`agent.py`](agent.py). It is a
fixed, linear sequence rather than a free-form decision loop — each step depends
on the previous step's output:

1. **Initialize** a fresh session dict (`_new_session`).
2. **Parse** the query into `description`, `size`, and `max_price` (see below) →
   `session["parsed"]`.
3. **Search** with those parameters → `session["search_results"]`. **If empty,
   write a helpful message to `session["error"]` and return immediately** — the
   loop does not proceed to the LLM tools with empty input.
4. **Select** the top result → `session["selected_item"]`.
5. **Suggest** an outfit from the selected item + wardrobe →
   `session["outfit_suggestion"]`.
6. **Create** the fit card from the outfit + item → `session["fit_card"]`.
7. **Return** the session.

**Query parsing uses the LLM.** `_parse_query` asks Groq to return strict JSON
(`description`, `size`, `max_price`) with `response_format={"type": "json_object"}`
and `temperature=0.0`. This handles fuzzy phrasing ("looking for a comfy oversized
sweater", "denim jacket below 50 size L") better than hand-written regex. If the
model returns malformed JSON, it falls back to using the whole query as the
description with no filters, so parsing never crashes the run.

---

## State Management

All state for a single interaction lives in **one session dictionary** created by
`_new_session(query, wardrobe)`. It is the single source of truth: each tool reads
the fields it needs and writes its result back, so nothing is passed implicitly
and no step re-prompts the user or uses hardcoded values.

| Field | Set by | Used by |
|-------|--------|---------|
| `query` | initial input | parsing step |
| `parsed` (`description`, `size`, `max_price`) | parsing step | `search_listings` |
| `search_results` | `search_listings` | item selection |
| `selected_item` | item selection (top result) | `suggest_outfit`, `create_fit_card` |
| `wardrobe` | initial input | `suggest_outfit` |
| `outfit_suggestion` | `suggest_outfit` | `create_fit_card` |
| `fit_card` | `create_fit_card` | final output |
| `error` | any step that fails | loop control + final output |

State flow is verifiable by object identity: at the end of a run,
`session["selected_item"]` **is the same dict object** that was passed into both
`suggest_outfit` and `create_fit_card`, and `session["outfit_suggestion"]` **is
the same string** that was passed into `create_fit_card`.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|--------------|----------------|
| `search_listings` | No listings match the query | Returns `[]` (no exception). The loop detects this, writes a message to `session["error"]`, and returns early — `suggest_outfit` and `create_fit_card` are never called. |
| `suggest_outfit` | Wardrobe is empty | Branches on `wardrobe["items"]` and returns general styling advice for the item alone instead of crashing. |
| `create_fit_card` | `outfit` is empty/whitespace | Returns a descriptive error string ("Could not create a fit card: no outfit suggestion was provided…") rather than raising. |

**Concrete example from testing (no-results branch):** running
`python agent.py`, the second built-in test case queries
`"designer ballgown size XXS under $5"`. Nothing in the dataset matches, so the
final session was:

```
search_results    = []        (0 listings)
selected_item     = None
outfit_suggestion = None
fit_card          = None
error             = "No listings found for 'designer ballgown' in size XXS under $5.
                     Try a broader description or a higher price limit."
```

A spy that raised if `suggest_outfit` or `create_fit_card` were called confirmed
**neither tool fired** on this path — the agent short-circuits at step 3 exactly
as specified.

---

## Spec Reflection

What matched the plan and what changed during implementation:

- **The session-dict design held up unchanged.** The loop, error early-return,
  and tool-to-tool hand-off all matched the State Management and Planning Loop
  sections of [`planning.md`](planning.md) without revision.
- **Query parsing changed approach mid-build.** The plan left parsing open
  ("regex, string splitting, or the LLM"). I first implemented regex, which
  worked but leaked stray punctuation into the description and was brittle on
  phrasing it didn't anticipate. I switched to an LLM JSON call, which is more
  robust but adds a third LLM call (and latency) per interaction — a deliberate
  tradeoff for natural-language flexibility.
- **`search_listings` returns *all* matches, not "three."** The original Tool 1
  spec said "returns three matching listings." The implementation returns every
  scored match sorted by relevance and lets the loop pick `[0]`. This keeps the
  tool general and pushes the "pick the top one" decision into the planning loop
  where it belongs.
- **Relevance scoring is simple keyword presence.** Each query keyword counts
  once across the searchable fields. It works well on this dataset but weights a
  common word like "vintage" as heavily as a specific one like "flannel"; a
  weighted scheme (title/tags > description) would be the next improvement.

---

## AI Usage

I used Claude (in Claude Code) as the AI tool throughout. Specific instances:

### Instance 1 — implementing `search_listings`

- **Input I gave it:** the Tool 1 spec block from `planning.md` (purpose, the
  three input parameters with types, the return value, and the no-results failure
  mode), plus the actual listing shape from `listings.json` and the requirement to
  reuse `load_listings()` rather than re-read the file.
- **What it produced:** a filter-then-score implementation — price/size filtering
  followed by keyword-overlap scoring across the listing's text fields, with
  zero-score results dropped and the rest sorted by score.
- **What I changed/verified before trusting it:** I had it match size
  case-insensitively as a substring so `"M"` matches `"S/M"`, and confirmed
  against the spec that it returns `[]` (not an exception) on no match. I ran it
  against three queries (graphic tee, denim jeans size M, and a deliberate
  no-match) before wiring it into the loop, and the `tests/` suite passes.

### Instance 2 — implementing the LLM-backed tools and choosing a model

- **Input I gave it:** the Tool 2 and Tool 3 spec blocks, the wardrobe schema, and
  the constraint to use Groq `llama-3.3-70b-versatile` with the empty-wardrobe and
  empty-outfit failure modes handled.
- **What it produced:** both tools, branching on `wardrobe["items"]` for
  `suggest_outfit` and guarding the empty `outfit` string for `create_fit_card`.
- **What I changed/overrode:** no model was referenced anywhere in the starter, so
  I added a single shared `_MODEL` constant in `tools.py` instead of hardcoding the
  string twice. I set `create_fit_card` to `temperature=1.0` and verified that
  three runs on identical input produced three distinct captions (the spec
  requires variety); `suggest_outfit` stays at `0.7`.

### Instance 3 — the planning loop and switching parsing to the LLM

- **Input I gave it:** the full agent diagram, the Planning Loop and State
  Management sections of `planning.md`, and the numbered TODO steps in
  `run_agent()`.
- **What it produced:** the linear loop with the early-return on empty search
  results and state stored in the session dict at each step.
- **What I changed/overrode:** I first accepted a regex parser, then replaced it
  with an LLM JSON-mode call for robustness, and added a defensive fallback so
  malformed model output can't crash the run. I verified the result with an
  object-identity check — confirming `selected_item` is the *same* object passed
  into both downstream tools — and with a spy proving the LLM tools are never
  called on the no-results branch.
