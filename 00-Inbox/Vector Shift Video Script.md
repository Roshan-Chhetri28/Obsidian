# Screen Recording Script — VectorShift Technical Assessment

**Target length: ~5:30.** Their ask, verbatim: *"focus on walking through the final
functionality and design, along with a brief discussion of your code."*
So demo first, code second. Don't invert it.

---

## Before you hit record

- [ ] Backend running: `cd backend && python main.py`
- [ ] Frontend running: `cd frontend && npm start`
- [ ] **Canvas empty** — refresh so you start clean
- [ ] Editor open with these tabs pre-loaded, in this order:
      `Node.jsx` → `FilterNode.jsx` → `registry.js` → `services/graph.py`
- [ ] Browser zoom ~110% (small text is unreadable in compressed video)
- [ ] Close unrelated tabs, silence notifications
- [ ] Do one full dry run first — the demo has a rhythm, and knowing it lets you
      talk instead of hunt for the mouse

---

## 0:00 – 0:20 — Open

> "Hi, I'm Roshan. This is my submission for the VectorShift frontend
> assessment. I'll show the working product first, then walk through how the
> code is organised — starting with the node abstraction, which is the piece
> the rest of the design hangs off."

Have the app on screen, canvas empty, toolbar visible.

---

## 0:20 – 1:00 — The nodes (Part 1 + 2)

**Do:** point at the toolbar. Drag **Input**, then **LLM**, then **Output** onto
the canvas. Connect Input → LLM → Output.

> "There are nine node types here — the original four, plus five I added to
> demonstrate the abstraction: Math, Filter, API, Timer, and Model."
>
> "Each node is a white card with a navy header and a colour-coded accent dot.
> The dot colour is shared between the toolbar chip and the node itself, from a
> single source, so they can't drift apart."

**Don't** dwell. This is scene-setting.

**Do:** drag a **Model** node out. Change **Provider** from Anthropic to OpenAI —
the Model dropdown re-populates.

> "The Model node is worth a second. Picking a provider re-filters the model list
> — Anthropic gives Claude models, OpenAI gives GPT models. What I like about it
> is that the base node needed no support for dependent fields: the config is
> just a prop, so this node derives its fields from its own state on every
> render. And if you switch provider while a model from the other one is
> selected, it falls back rather than showing an invalid option."

---

## 1:00 – 2:00 — Text node (Part 3) — *your strongest visual*

**Do:** drag a **Text** node out.

> "The Text node is where most of the logic lives."

**Do:** click into it, type: `Hello {{name}}, welcome to {{city}}`
— slowly enough that the handles appearing is visible.

> "As I type valid variable names in double curly braces, a handle appears on
> the left for each one. It validates against JavaScript identifier rules — so
> `{{2bad}}` is rejected because a variable can't start with a digit —"

**Do:** type `{{2bad}}` — show no handle appears. Delete it.

> "— and repeats are deduped, so `{{name}}` twice is still one handle."

**Do:** press Enter a few times / type a long line.

> "The node also grows in both width and height as I type — height from the
> textarea's content, width measured from the longest line."

**Do:** drag-select some text inside the textarea.

> "And selecting text inside a field doesn't drag the node — ReactFlow would
> normally hijack that mousedown."

---

## 2:00 – 2:45 — Submit + DAG (Part 4)

**Do:** refresh to clear the canvas, then click **Submit** with nothing on it.

> "Submitting an empty canvas is caught on the frontend — it never hits the
> backend."

Toast appears, auto-dismisses after 5s (don't wait for it — move on).

**Do:** rebuild Input → LLM → Output, connect, **Submit**.

> "With a real pipeline, it posts the nodes and edges to the FastAPI backend,
> which returns the counts and whether it's a valid DAG. Three nodes, two edges,
> and it is a DAG."

**Do:** close modal. Drag **two Filter** nodes out. Wire
`Filter1.out → Filter2.in`, then `Filter2.out → Filter1.in`. **Submit**.

> "And if I create a cycle — here, two filters pointing at each other — it
> correctly reports that it's not a DAG. Which matters, because a pipeline with
> a cycle can't be executed in any order."

---

## 2:45 – 4:15 — Code: the architecture argument

**This is the part they're grading. Slow down here.**

### `Node.jsx` (~40s)

> "The core of my design is this one file. Rather than each node re-implementing
> its own markup, handles and state, there's a single config-driven base node.
> It takes a title, an array of handles, and an array of fields, and renders any
> node from that description."
>
> "Handles are spaced evenly down whichever side they're on — one sits at 50%,
> two at 33 and 66 — so no node hardcodes handle positions."

### `FilterNode.jsx` (~25s)

> "Which means a concrete node is just a config. This is the entire Filter node —
> about ten lines. Adding a node is writing this, not copying a file."

**The line to land:**

> "The payoff is maintenance. When I styled the app, I restyled all nine nodes by
> editing one file. Same when I fixed the text-selection bug — one constant,
> every node fixed."

### `registry.js` (~20s)

> "Nodes are registered once here. The canvas and the toolbar both read from this
> list, so there's no second place to update and no way for them to fall out of
> sync."

### `services/graph.py` (~25s)

> "On the backend, the DAG check is Kahn's algorithm — repeatedly remove nodes
> with no incoming edges; if any are left over, there's a cycle. I liked it
> because the edge cases fall out for free: an empty graph is vacuously a DAG, a
> self-loop never reaches in-degree zero, and a diamond — where two branches
> rejoin — is correctly *not* a cycle, which a naive visited-check gets wrong."

---

## 4:15 – 5:15 — Structure

**Do:** show the folder tree in your editor sidebar.

> "Both sides are organised the same way. On the frontend: components, nodes,
> hooks, utils, services, store, constants, styles. The logic is extracted out of
> the components — the variable regex and the width measurement are pure
> functions in utils, the API call is in services, the textarea resize is a hook."
>
> "The backend mirrors it: routers for the HTTP layer, schemas for the models,
> services for the logic, core for config. Dependencies only point one way, so
> the DAG code doesn't import FastAPI at all and can be tested on its own."

> "It builds with zero warnings, and there are tests covering the variable
> parsing and that the app mounts."

---

## 5:15 – 5:30 — Close

> "That's the submission — the abstraction was the decision everything else
> followed from, and it's what made the styling and the bug fixes cheap.
> Thanks for watching."

---

## Delivery notes

- **Don't read code aloud line by line.** Say what it does and why you chose it.
  They can read.
- **Say "why" at least three times.** "I chose X because Y" is what separates a
  senior-sounding walkthrough from a tour.
- **If you fumble, keep going.** One retake of a bad section beats 6 takes of the
  whole thing.
- **Best single sentence you have:** *"restyling all nine nodes was one change."*
  Make sure it lands.
- If asked later what you'd harden: a Text node variable named `output` would
  collide with its own output handle's id. Knowing your own edge case is a good
  look.