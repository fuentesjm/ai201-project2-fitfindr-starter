# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Searches through a designated listings dataset in which returns matching items based on the input parameters.
**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `description` (str): represents what the user is specifically looking for in terms of clothings
- `size` (str): represents the size of what the user wears and looking for based on description
- `max_price` (float): represents how much the user is willing to pay.

**What it returns:**
<!-- Describe the return value — what fields does a result contain? -->
Returns three matching listings sorted by relevance
**What happens if it fails or returns nothing:**
<!-- What should the agent do if no listings match? -->
Output should let the user know there is no matching suggestion for the specific requirement

---

### Tool 2: suggest_outfit

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Obtain the specific item such as the 1 to 3 results return by step 1 and given the user's current wardrobe description. It will suggest 1 or more complete outfit combinations.
**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `new_item` (dict): based on the search_listings() results such as "band tee".
- `wardrobe` (dict): based on the query description of the user's wardrobe.

**What it returns:**
<!-- Describe the return value -->
return a string of outfit combinations
**What happens if it fails or returns nothing:**
<!-- What should the agent do if the wardrobe is empty or no outfit can be suggested? -->
Output should let the user know there is no matching outfit combination for the specific requirement
---

### Tool 3: create_fit_card

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Outputs a small description of a complete outfit based on step 1-2. Most produce something different each time and similar to a caption.
**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `outfit` (...): represents the complete outfit combination that have resulted from step 2 tool

**What it returns:**
<!-- Describe the return value -->
Returns a description in string format which will be a shareable description for a caption related post.
**What happens if it fails or returns nothing:**
<!-- What should the agent do if the outfit data is incomplete? -->
Should return error on why it failed or why the caption is unable to be produce.
---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?**
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->
the agent will start by reading the user's query. The agent will then strip out description of what the user's wants, their size, and max price the user is willing to pay. Based on those three information, the agent will call search_listings() and fill in the parameters for description, size, and max_price. The agent will then wait untill search_listings() outputs three matching listings and will use the top result (search_listings()[0]) and use that for new_item parameter for the next tool suggest_outfit(). The agent will also input the user's wardobe such as what they mostly wear as the wardrobe parameter for suggest_outfit() tool. The agent will then wait for the tool to return one or more complete outfit combinations and use the first complete outfit combination as the outfit parameter for create_fit_card() tool. Reusing the same new_item parameter as before and the outfit parameter,  the tool will generate a short, shareable description of the complete outfit.
---

## State Management

**How does information from one tool get passed to the next?**
<!-- Describe how your agent stores and accesses state within a session. What data is tracked? How is it passed between tool calls? -->

All state for a single interaction lives in one session dictionary created by _new_session(query, wardrobe) at the start of run_agent(). Each tool reads the fields it needs and writes its result back, so nothing is passed implicitly. The loop calls search_listings and stores its output in session["search_results"], selects the top result into session["selected_item"], passes that as new_item to suggest_outfit (storing session["outfit_suggestion"]), then feeds both into create_fit_card (storing session["fit_card"]).

---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Stop the loop. Write a message to `session["error"]` (e.g. "No listings found for '{description}' in size {size} under ${max_price}"), and return the session early without calling `suggest_outfit` or `create_fit_card`. |
| suggest_outfit | Wardrobe is empty | Don't crash — suggest an outfit built around the new item alone (or note that the user has no wardrobe items to pair with). If no combination can be produced, set `session["error"]` and skip `create_fit_card`. |
| create_fit_card | Outfit input is missing or incomplete | Validate that `selected_item` and `outfit_suggestion` are present first. If either is missing, set `session["error"]` explaining the card couldn't be generated and return without a fit card. |

---

## Architecture

<!-- Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     ASCII art, a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html), or an embedded
     sketch are all fine. You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool. -->

```
  User query + wardrobe
          |
          v
  +--------------------------+
  | Session dict             |  <--- single source of truth
  | (_new_session)           |        every tool reads/writes here
  +--------------------------+
          |
          v
  Parse query -> description, size, max_price
          |
          v
  +--------------------------+      no match
  | search_listings()        |---------------> session.error = "no listings found"
  +--------------------------+                            |
          | results found                                |
          v                                              |
  Select top result -> selected_item                     |
          |                                              |
          v                                              |
  +--------------------------+   none possible           |
  | suggest_outfit()         |-------------> session.error = "no outfit combination"
  +--------------------------+                            |
          | outfit produced                              |
          v                                              |
  +--------------------------+  missing input            |
  | create_fit_card()        |-------------> session.error = "card not generated"
  +--------------------------+                            |
          | card created                                 |
          v                                              v
  Return session ->                          Return session early ->
  show fit_card                              show error message

  (every tool above reads its inputs from, and writes its
   results back to, the shared Session dict)
```

---

## AI Tool Plan

<!-- For each part of the implementation below, describe:
     - Which AI tool you plan to use (Claude, Copilot, ChatGPT, etc.)
     - What you'll give it as input (which sections of this planning.md, your agent diagram)
     - What you expect it to produce
     - How you'll verify the output matches your spec before moving on

     "I'll use AI to help me code" is not a plan.
     "I'll give Claude my Tool 1 spec (inputs, return value, failure mode) and ask it to implement
     search_listings() using load_listings() from the data loader — then test it against 3 queries
     before trusting it" is a plan. -->

**Milestone 3 — Individual tool implementations:**

**Milestone 4 — Planning loop and state management:**

---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
<!-- What does the agent do first? Which tool is called? With what input? -->
the agent will call search_listings() tool and will use the "vintage graphic tee" as the description parameter. $30 in the query will be the max_price in the tool parameter. search_listings() will then return the top result
**Step 2:**
<!-- What happens next? What was returned from step 1? What tool is called now? -->
the agent will then call suggest_outfit(). The new_item parameter will be the top result of search_listings() tool and the wardrobe paramter will be the user's wardrobe such as from the original query "baggy jeans and chunky sneakers". This tool will return suggestion.
**Step 3:**
<!-- Continue until the full interaction is complete -->
the agent will then call create_fit_card() tool and outfit parameter will be the result of step 2 and new_item will be from step 1. 
**Final output to user:**
<!-- What does the user actually see at the end? -->
the output would be a result of a fit. An example will be "thrifted this faded band tee off depop for $22 and honestly it was made for my wide-legs 🖤 full look in my stories".
