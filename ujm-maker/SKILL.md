---
name: ujm-maker
description: >
  Builds a fully populated User Journey Map in Figma from source documents —
  interview transcripts, PDFs, research notes, or plain text. Works exclusively
  with the companion UJM Figma template.

  Use this skill whenever the user wants to create, populate, or generate a user
  journey map, customer journey map, experience map, or UJM in Figma — especially
  when they provide research or interview documents alongside a Figma link.
  Also trigger when someone says "turn this into a journey map", "build a UJM",
  "make a CJM from this research", or pastes a Figma link alongside any kind of
  user research document.
---

# UJM Maker

Turns raw research (interview transcripts, PDFs, notes, or a topic description)
into a fully populated User Journey Map inside the companion Figma template.
You handle everything: reading the docs, structuring the phases, presenting the
content plan for approval, then building directly in Figma.

---

## Companion Template

**This skill only works with the official UJM Figma template.**

The template is identified by a frame named exactly **"Journey map template"**.
If that frame is not found in the user's Figma file, stop and ask them to
duplicate the companion template first.

---

## Step 1 — Collect source material

Say exactly this to the user:

> "Drop in any source documents you have for the journey map (PDF, Word,
> Markdown, text — anything goes). If you don't have documents, just describe
> the topic and I'll generate the content from scratch. When you're ready,
> type **done**."

Read all uploaded documents (use the `anthropic-skills:pdf`, `anthropic-skills:docx`,
or `Read` tool as appropriate). Extract:

- Journey phases (explicit or implied)
- User tasks per phase
- Pain points and frustrations per phase
- Opportunities / improvement ideas per phase
- Direct user quotes (or representative ones)
- Emotional tone / sentiment per phase

If no documents are provided, ask for the topic and user type, then generate
realistic content from scratch.

---

## Step 2 — Get the Figma link

Ask:

> "Now paste the link to Figma template file link where you want to build the UJM. "

Extract the `fileKey` from the URL: `figma.com/design/:fileKey/...`

---

## Step 3 — Validate the companion template

**This is a hard gate. Do not scan, clone, or write anything in Figma until
this check passes.**

**Always load the `figma:figma-use` skill before calling `use_figma`.**

Search every page in the file for a frame whose `.name` is exactly
`"Journey map template"` (case-sensitive, no trailing spaces).

```js
// Search all pages
for (const page of figma.root.children) {
  await figma.setCurrentPageAsync(page);
  const tmpl = figma.currentPage.findOne(n => n.name === "Journey map template");
  if (tmpl) return { found: true, id: tmpl.id, pageId: page.id };
}
return { found: false };
```

- **Found** → record `templateId`, `pageId`, and dimensions. Continue to Step 4.
- **Not found** → stop immediately. Tell the user:
  > "I can't find a frame called **'Journey map template'** in this file.
  > This skill only works with the companion UJM template.
  > Please duplicate it from
  > https://www.figma.com/community/file/1628023173002348154/,
  > then paste the new file's link."

Do not proceed past this step until the frame is confirmed present.

---

## Step 4 — Scan the template structure

Run a read-only `use_figma` call to catalogue the template. You need:

- Top-level frame structure: `Header` + `Body`
- `Body` children: a `Headers` row + a `Phases` container
- Inside `Phases`: each phase frame and its sections —
  `Duration` (instance), `Summary` (PhaseIcon + Phase name instance + Phase description TEXT),
  `Quote` (UserQuote instance), `Sentiment` (SentimentCurve instance + SentimentFace instance),
  `User tasks` frame, `Pain points` frame, `Opportunities` frame
- Default item counts per phase (tasks / pain points / opportunities)
- SentimentCurve variant names available: `Type=High`, `Type=Neutral`, `Type=Low`, `Type=Chaotic`

---

## Step 5 — Build the content plan

Organise all extracted content into phases. Each phase needs:

| Field | Notes |
|---|---|
| **Name** | Short label — "Preparation", "Booking", etc. |
| **Timeline** | When this phase occurs — "1–2 weeks before", "Day of booking", etc. |
| **Story** | 2–3 sentence narrative for the Phase description field |
| **User quote** | One direct or representative quote |
| **Sentiment** | 🙁 / 😐 / 🙂 — maps to Chaotic / Low+Neutral / High curve |
| **Tasks** | 6–10 user actions in this phase |
| **Pain points** | 3–6 frustrations or blockers |
| **Opportunities** | 3–6 improvement ideas |

Present the complete plan to the user, clearly organised by phase. End with:

> "Does this content look right? You can ask me to adjust any phase, task,
> wording, or structure before I write to Figma.
>
> When you're happy, type **yes** to create the UJM in Figma, or **no** to
> keep editing."

Wait for approval before touching Figma.

---

## Step 6 — Build in Figma

**Always load the `figma:figma-use` skill before each `use_figma` call.**
Work incrementally — one `use_figma` call per logical step.

### 6a — Clone the template

Clone the "Journey map template" frame. Name the clone after the topic
(e.g. `"Travel Insurance — User Journey Map"`). Position it to the right of
all existing content: scan `figma.currentPage.children` to find `max(x + width)`,
then place the clone at `maxX + 200`. Return the clone ID.

### 6b — Map the clone's structure

Scan the clone to collect node IDs for:
- Header title TEXT node
- Phases container frame
- Each phase and its key sections (all IDs you'll need to write to)

### 6c — Handle phase count

The template has **4 phases**. If the content plan has **more than 4**:

1. Clone an existing phase frame for each extra phase needed
2. Append each clone to the Phases container
3. Re-scan to capture the new node IDs

If the plan has **fewer than 4** phases, leave extra phases with placeholder
text — don't delete them (the user can clean up manually).

### 6d — Populate each phase

Handle each phase in a single `use_figma` call. In each call:

1. **Load all fonts first** — `clone.findAll(n => n.type === 'TEXT')`, collect
   unique `fontName` values, `await figma.loadFontAsync()` for each. Do this
   before touching any text.
2. **Duration** — `instance.findOne(n => n.name === "Label" && n.type === "TEXT")`
3. **Phase name** — same pattern on the Phase name instance
4. **Phase description** — set `.characters` on the TEXT node directly
5. **User quote** — find `"Quote"` and `"User name and role"` TEXT nodes inside
   the UserQuote instance
6. **Sentiment face** — find `"Face"` TEXT node inside SentimentFace; set emoji
   (🙁 / 😐 / 🙂)
7. **Adjust list counts** (tasks / pain points / opportunities):
   - Too few items → clone last child, `parent.appendChild(clone)`, then set text
   - Too many items → `node.remove()` from the end
   - Then iterate `frame.children` and set each `"Title"` TEXT node
8. **Return all mutated node IDs**

### 6e — Swap phase icons

Each phase has a `PhaseIcons` instance inside its `Summary` frame. Replace the
default icon with one that fits the phase's theme.

**Step 1 — Discover available icons (read-only call)**

Scan the file for the `PhaseIcons` component set to get all available variant names and IDs:

```js
await figma.setCurrentPageAsync(figma.root.children.find(p => p.id === "<pageId>"));
const iconSet = figma.currentPage.findOne(n => n.type === "COMPONENT_SET" && n.name === "PhaseIcons");
if (!iconSet) return { found: false };
return iconSet.children.map(c => ({ name: c.name, id: c.id }));
```

**Step 2 — Match semantically**

For each phase, pick the most contextually fitting variant. Use the phase name, its story, and key tasks as context. Some heuristics:

| Phase theme | Good icon choices |
|---|---|
| Research / preparation | `ID=Read`, `ID=Document`, `ID=Question` |
| Purchase / payment | `ID=Buy`, `ID=Bill`, `ID=Exchange` |
| Travel / departure | `ID=Travel`, `ID=Navigate`, `ID=Map` |
| Arrival / check-in | `ID=Home`, `ID=Key`, `ID=Done` |
| Digital / app use | `ID=PC`, `ID=Code`, `ID=Settings` |
| People / contact | `ID=User`, `ID=Call`, `ID=Chat` |
| Problem / stress | `ID=Stress`, `ID=Warning`, `ID=Wait` |
| Medical / health | `ID=Security`, `ID=Fill`, `ID=Process` |
| Paperwork / claim | `ID=Document`, `ID=Edit`, `ID=Bill` |
| Completion | `ID=Done`, `ID=Success`, `ID=Star` |
| Onboarding / start | `ID=Launch`, `ID=Add`, `ID=Goal` |

Pick a different icon for each phase — don't reuse the same one across phases.

**Step 3 — Swap all icons in one call**

```js
await figma.setCurrentPageAsync(figma.root.children.find(p => p.id === "<pageId>"));
const cloneFrame = await figma.getNodeByIdAsync("<cloneId>");

// iconMap: array of { phaseId, componentId } in phase order
const iconMap = [
  { phaseId: "<phase1Id>", componentId: "<iconComponentId>" },
  // ...
];

const results = [];
for (const { phaseId, componentId } of iconMap) {
  const phase = await figma.getNodeByIdAsync(phaseId);
  const iconInst = phase.findOne(n => n.name === "PhaseIcons" && n.type === "INSTANCE");
  if (!iconInst) { results.push({ phaseId, error: "no PhaseIcons instance" }); continue; }
  const comp = await figma.getNodeByIdAsync(componentId);
  await iconInst.swapComponent(comp);
  results.push({ phaseId, componentId, name: comp.name });
}
return results;
```

### 6f — Set the header title

Update the header title TEXT node to the map title.

### 6g — Screenshot and verify

Take a screenshot of the full clone frame. Check that:
- All phases are visible with real content (not placeholders)
- Each phase has a distinct, contextually appropriate icon (not the default placeholder)
- No obvious text clipping

Fix anything that looks wrong before reporting back.

---

## Step 7 — Deliver

Return:
- A direct Figma link:
  `https://www.figma.com/design/{fileKey}?node-id={nodeId with : replaced by -}`
- A one-line summary of what was built

---

## Technical reference

**Finding text inside instances**
```js
instance.findOne(n => n.name === "Label" && n.type === "TEXT")
```
Never assume direct child order inside an instance.

**Cloning list items**
```js
// Always: clone → appendChild → THEN set sizing/text
const newItem = lastChild.clone();
parent.appendChild(newItem);
const t = newItem.findOne(n => n.name === "Title" && n.type === "TEXT");
t.characters = "New text";
```

**Font loading (do once per call)**
```js
const fonts = new Set();
clone.findAll(n => n.type === 'TEXT').forEach(t => {
  try { fonts.add(JSON.stringify(t.fontName)); } catch(e) {}
});
for (const f of fonts) {
  const { family, style } = JSON.parse(f);
  await figma.loadFontAsync({ family, style });
}
```

**Page context resets between calls**
Always start each `use_figma` call with:
```js
await figma.setCurrentPageAsync(figma.root.children.find(p => p.id === "<pageId>"));
```

**Swapping a PhaseIcon instance**
```js
// Find the instance inside the phase's Summary frame
const iconInst = phaseFrame.findOne(n => n.name === "PhaseIcons" && n.type === "INSTANCE");
// Load the target component by its node ID
const comp = await figma.getNodeByIdAsync("106:1149"); // e.g. ID=Travel
await iconInst.swapComponent(comp);
```
Discover available variants first:
```js
const iconSet = figma.currentPage.findOne(n => n.type === "COMPONENT_SET" && n.name === "PhaseIcons");
return iconSet.children.map(c => ({ name: c.name, id: c.id }));
```

**Sentiment curve → emoji mapping**

| Sentiment | Curve variant | Emoji |
|---|---|---|
| Positive / easy | Type=High | 🙂 |
| Mixed / cautious | Type=Neutral | 😐 |
| Negative / stressed | Type=Low | 😐 or 🙁 |
| Highly negative / chaotic | Type=Chaotic | 🙁 or 😤 |
