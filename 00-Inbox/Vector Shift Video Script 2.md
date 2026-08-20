# Screen Recording Script — VectorShift Assessment

**Target: ~4:00.** Their word was *"brief."* Demo first, code second.
The spoken lines below are short on purpose — dense prose is what pushed the
last version to 10 minutes. Say these roughly, don't recite them.

---

## Before you record

- [ ] Backend up: `cd backend && uv run python main.py`
- [ ] Frontend up: `cd frontend && npm start`
- [ ] Canvas empty (refresh)
- [ ] Two editor tabs open: **`Node.jsx`** and **`FilterNode.jsx`**
- [ ] Browser zoom ~110%, notifications off
- [ ] One dry run so you know the click order

---

## 0:00–0:15 — Open

> "Hi, I'm Roshan — this is my VectorShift assessment. I'll demo it, then show
> the node abstraction the rest of the design is built on."

App on screen, canvas empty.

---

## 0:15–1:15 — Demo the nodes

**Input → LLM → Output**, connect them.

> "Nine node types — the original four plus five I added. Each is one config on a
> shared base node."

**Drag Model. Switch provider Anthropic → OpenAI** (list re-populates).

> "The Model node's model list depends on the provider — and the base node needed
> no special support for that. The config is just a prop."

**Drag Text. Type `Hello {{name}}, {{city}}`** slowly.

> "In the Text node, each valid variable in double braces adds a handle on the
> left."

**Type `{{2bad}}`** (no handle), delete it.

> "It follows JS naming, so `2bad` is rejected. Repeats dedupe."

**Press Enter a couple times; drag-select inside the box.**

> "It grows with the text, and selecting inside it doesn't drag the node."

---

## 1:15–2:00 — Submit + DAG

**Submit on empty canvas** → toast (don't wait for it).

> "Empty pipeline is caught before it hits the backend."

**Rebuild Input → LLM → Output, Submit** → modal.

> "A real pipeline posts to the FastAPI backend — node count, edge count, and
> whether it's a valid DAG. This one is."

**Two Filter nodes wired into a loop, Submit** → not a DAG.

> "A cycle is correctly flagged — a pipeline with a cycle can't run in order."

---

## 2:00–3:15 — The code (the part they grade)

**Tab to `Node.jsx`.**

> "This is the core. One base node renders any node from a config — a title, an
> array of handles, an array of fields. Handles space themselves evenly down each
> side, so nodes never hardcode positions."

**Tab to `FilterNode.jsx`.**

> "So a real node is just this — about ten lines. Adding a node is writing a
> config, not copying a file."

**The one line that must land:**

> "That's the whole point: I styled all nine nodes, and fixed a drag bug across
> all nine, each by editing one file."

**Stay on the tab, say (no need to open graph.py):**

> "On the backend, the DAG check is Kahn's algorithm, in its own service that
> doesn't import FastAPI — so it's testable on its own. Empty graphs, self-loops,
> and diamonds all fall out correctly without special-casing."

---

## 3:15–3:50 — Structure

**Pan the editor sidebar.**

> "Both sides use the same layout — components, hooks, utils, services on the
> front end; routers, schemas, services on the back. Logic lives in utils and
> services, not inside components. Zero build warnings, with tests on the app and
> the variable parsing."

---

## 3:50–4:00 — Close

> "That's it — the abstraction was the one decision everything else followed from.
> Thanks for watching."

---

## Delivery notes

- Don't read code line by line. Say what it does and why.
- If you fumble, keep going — retake one section, not the whole thing.
- The sentence that sells it: **"styled all nine, fixed a bug across all nine,
  one file each."**
- If asked what you'd harden: mention you namespaced the Text node's variable
  handles so a variable named `output` can't collide with the output handle —
  a small edge case you caught and closed.
- 