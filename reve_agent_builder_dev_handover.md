# REVE Agent Builder — Developer Handover
**v1** · Developer Handover

Reference agent: EIT Agent V2. Reference wizard: REVE Chat multi-category onboarding.
This handover describes generation logic, not the deployed runtime.

`reference: EIT Agent V2` · `source: onboarding wizard (8 steps)` · `target: node-graph runtime` · `llm: gpt-4o-mini · gpt-4.1-mini`

---

## Turn an onboarding wizard into a working *Agent JSON* — no manual wiring

The onboarding journey collects a business's category, sub-category, and behaviour requirements, then assembles three deliverables automatically: the **Agent JSON** (the node-graph workflow), the **system prompt**, and the **output schema**. This document specifies the orchestration model, the prompt-generation pipeline, and the exact artifacts the generator emits — so the backend can be built fast and correctly.

> **The one-line contract.**
> Onboarding selections → `fetch cached prompt` → `merge free-text via Generator API` → `test & revise (×3)` → `emit nodes[] + edges{} + prompt + responseFormat` → redirect to dashboard with the graph pre-loaded.

## Contents

- [01 · What the onboarding captures](#01)
- [02 · The builder orchestration](#02)
- [03 · Node & edge schema](#03)
- [04 · Prompt-build pipeline](#04)
- [05 · Generator API contract](#05)
- [06 · The 3-round test loop](#06)
- [07 · Final prompt (worked example)](#07)
- [08 · Output schema (worked example)](#08)
- [09 · Onboarding → JSON field mapping](#09)
- [10 · Emitted Agent JSON (skeleton)](#10)

---

<a id="01"></a>
## 01 · What the onboarding captures

Each wizard step writes into a single state object. The generator reads that state — nothing else — to build the agent. The table below is the complete input surface; the "Feeds" column is the JSON target each field drives (detailed in §09).

| Step | Captured | State key | Feeds |
|---|---|---|---|
| 0–1 · Account | Name, work email, OTP | `—` | `accountId`, `creator{}` |
| 2 · Agent mode | ai · hybrid · human | `agentMode` | handover node presence + routing |
| 3 · Industry | ecom / bfsi / telecom / travel / saas / other | `selInd` | prompt key, tool set, sub-agent suggestions |
| 3 · Categories | Multi-select store categories (ecom) | `selSubs[]` | prompt `{cat}`, KB description |
| 4 · Identity | Agent name, greeting | `agentName` · `greeting` | name, designation, `trigger.initialMsg` |
| 4 · Use cases | Chips per industry | `selUCs[]` | prompt "use cases" segment, tools |
| 4 · Custom instructions | Free-text + quick-add pills | `customPrompt` | Generator API `additional_requirements` |
| 5 · Handover | Triggers + schedule + team | `handoverTriggers` · `handoverSchedule` · `teamMembers` | conditional branches, routing |
| 6 · Knowledge | Docs, web crawl, store/API sync | `kbDocs` · `kbCrawl` · `storeOk` · `prodSync` · `kbApi` | `tool_knowledgebase`, `tool_google_sheet`, `node_api_action` |

**Note:** `ucKey()` maps `selInd` to a prompt key (e.g. `ecom → ecom_shopify`). `categoryPromptLabel()` joins `selSubs` into a human phrase for the `{cat}` token.

---

<a id="02"></a>
## 02 · The builder orchestration

A deployed agent is a directed graph of **nodes** joined by **edges**. Below is the EIT reference graph the generator produces for an e-commerce store with a product sheet, a knowledge base, and an image-search path. Colour = node category.

**Legend:** `trigger` (amber) · `conditional` (violet) · `ai_agent` (indigo) · `api_action` (slate) · `tool` (teal) · `response` (green)

![Agent orchestration node graph](diagram_section.png)

Two things make this graph readable at generation time:

1. **The conditional fans out by branch index**: `index 0` is the first declared condition (text) and routes to the text agent; `index 1` (image) routes to the vision API node.
2. **Tools are bound, not chained** — they sit in the edge's `tools[]` array of an `ai_agent` node and are called on demand, never as a sequential downstream step.

---

<a id="03"></a>
## 03 · Node & edge schema

Every node shares one envelope; only `data{}` varies by `type`. Edges are a map keyed by *source* node id.

### Node envelope

```json
{
  "id": "uuid-v4",
  "name": "machine_name",        // referenced in {{local.<name>.output}}
  "displayName": "Human Name",
  "type": "node_ai_agent",       // see type table below
  "category": "node",            // trigger | node | tool
  "executor": "node_ai_agent",   // == type
  "webhookId": null,             // set only on trigger nodes
  "position": { "x": 0, "y": 0 },  // canvas layout only
  "data": { /* type-specific */ },
  "persistOutputInContext": null
}
```

### Node types & their `data{}`

| type | category | Key `data` fields |
|---|---|---|
| `trigger_chat_message` | trigger | `widgetId`, `initialMsg`, `chatSource`, `responseMode`, `postType` |
| `node_conditional` | node | `conditions[]` → `outerBlocks` → `innerBlocks {key, logic, value, valueType}` |
| `node_ai_agent` | node | `systemPrompt`, `inputPrompt`, `maxInteractionCount`, `responseFormat`, `enableResponseFormat`, `llmModel{provider,model}` |
| `node_api_action` | node | `method`, `endpoint`, `headers[]`, `jsonData`, `bodyType`, `curl` |
| `node_response_to_chat` | node | `responseType`, `response` |
| `tool_knowledgebase` | tool | `selectedKBs[]`, `knowledgeBaseIds[]`, `description` |
| `tool_google_sheet` | tool | `documentId`, `sheetName`, `operation`, `credential` |

### Edge map

The `edges` object is keyed by the **source** node id. `nodes[]` are downstream execution targets; `tools[]` are tools bound to an agent. `index` on a target selects the matching conditional branch. Leaf nodes (responses, tools) still get an entry with empty arrays.

```json
"edges": {
  "<on_chat_message.id>": { "nodes": [{ "id":"<condition.id>", "type":"node", "index":0 }], "tools":[] },

  "<condition.id>": { "nodes": [
      { "id":"<ai_agent_1.id>", "type":"node", "index":0 },  // text branch
      { "id":"<api.id>",        "type":"node", "index":1 }   // image branch
    ], "tools":[] },

  "<ai_agent_1.id>": { "nodes": [{ "id":"<response_to_chat.id>", "type":"node", "index":0 }],
    "tools": [ {"id":"<kb.id>","type":"tool","index":0}, {"id":"<sheet.id>","type":"tool","index":0} ] },

  "<api.id>": { "nodes":[{ "id":"<ai_agent.id>", "type":"node", "index":0 }], "tools":[] },
  "<ai_agent.id>": { "nodes":[{ "id":"<response_to_chat_1.id>", "type":"node", "index":0 }], "tools": [/*kb, sheet*/] },

  "<response_to_chat.id>":   { "nodes":[], "tools":[] },  // leaf
  "<kb.id>": { "nodes":[], "tools":[] }, "<sheet.id>": { "nodes":[], "tools":[] }
}
```

> **Wiring rule.** An `inputPrompt` references an upstream node by machine `name`, not id — e.g. the text agent reads `{{local.on_chat_message.body.query.message.text}}`, while the image agent reads `{{local.api.body.output[0].content[0].text}}`. Keep `name` stable when you generate ids.

---

<a id="04"></a>
## 04 · Prompt-build pipeline

The prompt is never hand-authored at runtime. It is composed from a cached base plus the user's free-text, merged by a dedicated Generator API call whose system message defines exactly how the merge happens.

**1. Resolve inputs** — Read `ucKey(selInd)` → prompt key; `categoryPromptLabel(selSubs)` → `{cat}`; collect `selUCs`, `customPrompt`, quick-add pills, and available tools (KB / sheet / API present?).

**2. Fetch cached prompt** — Look up `PROMPTS[key]` and substitute `{cat}`. This is the deterministic base — **no LLM call yet**. If the user added nothing, this (segmented) is the final prompt.

**3. Merge free-text via Generator API** — Only if `customPrompt` or quick-add pills exist: send `{ cached_prompt, metadata, additional_requirements }` to the GPT Prompt Generator. The system message (§05) tells the model how to slot each requirement into the correct segment without dropping safety rules.

**4. Enforce segmentation** — The returned prompt must carry the fixed section headers (below). Reject/re-ask if a segment is missing. Segmentation is what keeps later revisions surgical.

**5. Test & revise (×3)** — Feed test-panel transcript + "Correct this" notes back to the Generator as revision instructions. Max 3 rounds. On round 3 → `goDash()`, graph pre-loaded in the visual builder.

### Mandatory prompt segments

Every generated `systemPrompt` is written in this fixed order. The Generator maps each additional requirement to exactly one segment; nothing goes in "misc."

| Segment | Holds | Source |
|---|---|---|
| `ROLE & GOAL` | Persona, business, conversion objective | cached base + identity |
| `DOMAIN` | In-scope topics; refuse out-of-scope | cached base (industry) |
| `TOOLS` | Which tool for which query; never guess | KB / sheet / API presence |
| `DATA / FIELDS` | Columns, normalization, availability mapping | store/sheet connection |
| `SELECTION LOGIC` | Match tiers, budget parsing, fallbacks | use cases + cached base |
| `RESPONSE RULES` | Brevity, CTA, lead capture triggers | use cases + custom |
| `OUTPUT FORMAT` | The exact reply shape (mirrors output schema) | output schema §08 |
| `SAFETY` | No fabrication, no competitors, escalation | cached base (locked) |

---

<a id="05"></a>
## 05 · Generator API contract

One endpoint does both composition (step 3) and revision (step 5). The system message is fixed and shipped by us; the user's data rides in the user message. This is the "instruction given prior" that decides how requirements attach to the prompt.

### System message (fixed, shipped by backend)

```
You are a Prompt Composer for a no-code support-agent builder.
You receive a CACHED base prompt, agent METADATA, and a list of
ADDITIONAL_REQUIREMENTS (free text + quick-add behaviours).

RULES
1. Output ONE system prompt using EXACTLY these segment headers,
   in order: ROLE & GOAL, DOMAIN, TOOLS, DATA / FIELDS,
   SELECTION LOGIC, RESPONSE RULES, OUTPUT FORMAT, SAFETY.
2. Slot each ADDITIONAL_REQUIREMENT into the single most relevant
   segment. Never invent a segment. Never place a rule in two.
3. Preserve every line of the SAFETY segment from the base prompt
   verbatim. You may ADD safety lines; you may not remove them.
4. Do not reference tools that are not listed in METADATA.tools.
5. Keep instructions imperative and testable. No marketing copy.
6. Also return an OUTPUT_SCHEMA (JSON Schema) that matches the
   OUTPUT FORMAT segment exactly.
7. Reply as JSON: { "system_prompt": string, "output_schema": object,
   "segments_touched": string[] }. No prose outside the JSON.
```

### Request — compose

```json
{
 "cached_prompt": "<PROMPTS[key], {cat} filled>",
 "metadata": {
   "industry": "ecom",
   "categories": ["Fashion & apparel","Electronics"],
   "use_cases": ["Order tracking","Product recommendations"],
   "tools": ["knowledgebase","google_sheet"],
   "agent_mode": "hybrid"
 },
 "additional_requirements": [
   "Offer free shipping on carts over $50.",
   "Always upsell one complementary item."
 ]
}
```

### Request — revise (rounds 1–3)

```json
{
 "current_prompt": "<last generated prompt>",
 "round": 2,
 "revision_instructions": [
   { "user": "where is my order 1042?",
     "agent": "<bot reply>",
     "correction": "Ask for email before sharing order status." }
 ]
}
// same system message; model edits only the segment each
// correction maps to (diff-preserving).
```

Endpoint mirrors the OpenAI Responses API already used by the image path (`POST /v1/responses`, `gpt-4.1-mini`). Keep the key server-side — do not ship it to the client as the reference JSON currently does.

> **Security fix required.** The reference EIT JSON hard-codes an OpenAI bearer token in the `api` node headers. The generator and any runtime API node must read credentials from a server-side vault, never from the exported JSON.

---

<a id="06"></a>
## 06 · The 3-round test loop

The Test panel is where the user validates behaviour and corrects it inline. Each correction is a revision instruction; three rounds is the hard cap before handoff to the dashboard.

| Round | User action | Backend | Exit |
|---|---|---|---|
| **1** | Chats as a customer; clicks "Correct this" on off replies | Buffer transcript + corrections | User clicks "Apply & update" → revise call |
| **2** | Re-tests with revised prompt | Diff-preserving revise; bump `round` | Apply → revise call |
| **3** | Final validation pass | Last revise; freeze prompt + schema | **Auto-redirect → `goDash()`**, graph pre-loaded |

State to track per session: `round ∈ {1,2,3}`, `promptVersions[]` (keep each for rollback), and the running `corrections[]` buffer. When `round === 3` completes, persist the frozen prompt into every `node_ai_agent.data.systemPrompt`, write `responseFormat` + set `enableResponseFormat=true`, then route to the dashboard where `buildDashWF()` renders the same graph for manual extension.

> **Why cap at 3.** Each round is a full prompt rewrite; unbounded loops drift the prompt away from its segmented base and burn tokens. Three rounds is enough to fix tone, missing steps, and one behaviour gap — anything deeper belongs in the visual builder, which is exactly where round 3 lands the user.

---

<a id="07"></a>
## 07 · Final prompt — from the chosen options

Generated for the default onboarding selection: **E-commerce**, categories **Fashion & apparel + Electronics**, use cases **Order tracking + Product recommendations**, mode **hybrid**, tools **knowledgebase + google_sheet**, plus the two quick-add behaviours. This is what lands in `node_ai_agent.data.systemPrompt`.

```
ROLE & GOAL
You are the Support & Sales Agent for an online store selling fashion &
apparel and electronics. Your goal is to resolve customer questions and
convert intent into orders: find the right product, give accurate
stock / price / link, recommend better-fit alternatives, cross-sell one
relevant item, and capture leads when purchase intent is high.

DOMAIN
Stay inside store scope: products, orders, delivery, returns, offers,
sizing, and general store policy. Politely refuse unrelated topics.

TOOLS
- google_sheet (product search): use for ANY product, price, stock,
  availability, size/variant, budget, or comparison query — including
  "do you have X", "is X available", "X in size M?".
- knowledgebase: use ONLY for non-product topics — delivery options,
  return & refund policy, payment methods, store hours, contact.
Never guess stock, price, link, or policy. If a tool is needed, call it.

DATA / FIELDS
Product rows carry: Name, Specification, Size/Variant, Price, In Stock,
Up Coming, Product Link. Availability mapping (always apply):
- In Stock > 0                 → Available
- In Stock = 0, Up Coming > 0  → Upcoming / pre-arrival
- In Stock = 0, Up Coming = 0  → Out of stock; offer alternatives

SELECTION LOGIC
- Order tracking: ask for the order number (and email to verify) before
  sharing status; then return status + ETA in one line.
- Product recommendations: return up to 5 in-stock matches ranked by
  fit, then value. If no exact match, show the closest and name the
  difference ("16GB, you asked for 8GB").
- Budget phrases ("under 50k", "100-120k") → numeric range before filter.

RESPONSE RULES
Answer first when you have enough info. Keep replies short and helpful,
with one clear CTA. Offer free shipping on carts over $50. Always upsell
exactly one complementary item when relevant. Capture name + email +
contact when the customer signals buy / book / visit.

OUTPUT FORMAT
For product replies, one block per product:
1. {Product name}
   - Specification: {clean spec}
   - Price: {price OR "Price not available yet"}
   - Availability: {Available | Limited | Upcoming | Out of stock}
   - Link: {product link if present}
   - Differences: {only if a requested spec differs}

SAFETY
Do not answer out-of-domain questions. Do not discuss competitors except
to suggest an in-store alternative. Never invent product, stock, price,
link, offer, or policy. On sensitive or high-value cases, or after 2
unresolved turns, hand over to a human agent.
```

The SAFETY and DOMAIN segments come verbatim from the cached base; ROLE/RESPONSE were composed with the two quick-add requirements; TOOLS/DATA reflect that the store connected a sheet + KB. Swap categories or tools and only the affected segments change.

---

<a id="08"></a>
## 08 · Output schema — from the chosen options

This JSON Schema is emitted alongside the prompt and written to `node_ai_agent.data.responseFormat` with `enableResponseFormat = true`. It matches the OUTPUT FORMAT segment one-to-one, so the runtime can render cards and the QA harness can assert on fields.

```json
{
  "name": "store_agent_reply",
  "strict": true,
  "schema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "reply_text": { "type":"string", "description":"Natural-language message shown to the customer" },
      "intent": { "type":"string", "enum":["order_tracking","product_reco","policy","smalltalk","handover"] },
      "products": {
        "type":"array", "maxItems":5,
        "items": {
          "type":"object", "additionalProperties":false,
          "properties": {
            "name": {"type":"string"},
            "specification": {"type":"string"},
            "price": {"type":["string","null"]},
            "availability": {"type":"string", "enum":["Available","Limited","Upcoming","Out of stock"]},
            "link": {"type":["string","null"]},
            "differences": {"type":["string","null"], "description":"Set only when a requested spec differs"}
          },
          "required": ["name","specification","price","availability","link","differences"]
        }
      },
      "cross_sell": { "type":["string","null"], "description":"One complementary item, or null" },
      "lead": {
        "type":["object","null"], "additionalProperties":false,
        "properties": { "name":{"type":["string","null"]}, "email":{"type":["string","null"]}, "phone":{"type":["string","null"]} },
        "required": ["name","email","phone"]
      },
      "handover": { "type":"boolean", "description":"true → route to a human per handover triggers" }
    },
    "required": ["reply_text","intent","products","cross_sell","lead","handover"]
  }
}
```

`reply_text` is what `node_response_to_chat` streams; `products`/`cross_sell` drive rich cards; `handover=true` is read by the conditional/routing layer when mode is `hybrid` or `human`.

---

<a id="09"></a>
## 09 · Onboarding → JSON field mapping

The generator's write map. Read left, write right.

| Onboarding input | Target in Agent JSON |
|---|---|
| `agentName` / fallback `agName()` | `name`, `displayName`, `designation` |
| `greeting` | `nodes[trigger].data.initialMsg` |
| `selInd` + `selSubs` | prompt `{cat}`, `tool_knowledgebase.description`, tool selection |
| `selUCs` | prompt SELECTION LOGIC / RESPONSE segments; sub-agent suggestions (`WFC`) |
| `customPrompt` + quick-add (`QA`) | Generator `additional_requirements[]` |
| `agentMode` | presence of handover routing; `handover` field consumption |
| `handoverTriggers` | `node_conditional.conditions[]` branches |
| `kbDocs` / `kbCrawl` | `tool_knowledgebase.knowledgeBaseIds[]` |
| `storeOk` / `prodSync` | `tool_google_sheet` (documentId, sheetName, get_rows) |
| image support (implicit for ecom) | `node_conditional` image branch → `node_api_action` (vision) → second `node_ai_agent` |
| test corrections (`chatMsgs`) | Generator `revision_instructions[]` → all `systemPrompt` |

---

<a id="10"></a>
## 10 · Emitted Agent JSON (skeleton)

The final document the generator persists and the dashboard loads. Prompt (§07) and schema (§08) drop into the two `node_ai_agent` nodes; ids are freshly minted but machine `name`s stay fixed for the `{{local.*}}` references.

```json
{
  "accountId": "<from session>",
  "name": "<agentName>",
  "designation": "<agentName> AI Assistant",
  "status": true,
  "nodes": [
    { "name":"on_chat_message", "type":"trigger_chat_message", "category":"trigger",
      "data":{ "initialMsg":"<greeting>", "chatSource":"website", "responseMode":"LAST_NODE" } },

    { "name":"condition", "type":"node_conditional", "category":"node",
      "data":{ "conditions":[ /* [0]=text, [1]=image on query.type */ ] } },

    { "name":"ai_agent_1", "type":"node_ai_agent", "category":"node",
      "data":{ "systemPrompt":"<§07 FINAL PROMPT>",
             "inputPrompt":"{{local.on_chat_message.body.query.message.text}}",
             "enableResponseFormat":true, "responseFormat":"<§08 OUTPUT SCHEMA>",
             "llmModel":{"provider":"openai","model":"gpt-4o-mini"} } },

    { "name":"api",          "type":"node_api_action" },        // vision: image → product name
    { "name":"ai_agent",     "type":"node_ai_agent" },          // inputPrompt = {{local.api...text}}
    { "name":"response_to_chat",   "type":"node_response_to_chat" }, // {{local.ai_agent_1.output}}
    { "name":"response_to_chat_1", "type":"node_response_to_chat" }, // {{local.ai_agent.output}}
    { "name":"eit_knowledgebase_tool", "type":"tool_knowledgebase", "category":"tool" },
    { "name":"google_sheet",          "type":"tool_google_sheet",  "category":"tool" }
  ],
  "edges": { /* exactly as §03: trigger→condition→{agent_1|api}→response, tools bound to both agents */ }
}
```

> **Build order for the generator.**
> (1) mint node ids + fixed names → (2) fill trigger from `greeting` → (3) fill tools from KB/store state → (4) drop frozen prompt + schema into both agents → (5) build `edges` from the template, binding tools to agents and wiring the conditional branch indexes → (6) attach `accountId`/`creator` → (7) persist & hand to `buildDashWF()`.
