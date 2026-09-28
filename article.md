# Building the Frontend for Hindsight Before the Backend Existed

The first version of our interface could not talk to anything. It had a text box, a button, and a function that returned `"Backend not connected yet."` Three other people were still working on the memory layer, the prompts, and the wiring, and I had to hand them a screen they could plug into without asking me for changes.

That constraint ended up shaping the whole design. This is a write-up of how we built a procurement briefing tool around [Hindsight, the open-source agent memory system from Vectorize](https://github.com/vectorize-io/hindsight), and why the most useful thing I did was decide what the UI would refuse to claim.

## What the system does

Procurement teams negotiate with the same vendors again and again, and the history lives in email threads, spreadsheets and people's heads. The price a vendor accepted last spring, the delivery that slipped by two weeks, the discount that only appeared after someone escalated: all of it gets lost between negotiations.

You type a vendor name and get a briefing. The briefing says what happened last time, which tactic worked on this particular vendor, what to ask for, and what to be wary of. When a draft email makes sense, it comes with one.

There are three pieces:

1. **Memory.** Every vendor interaction is retained in Hindsight as a plain-language record: what was quoted, what was agreed, what went wrong, and what moved the vendor.
2. **Briefing service.** On each request it recalls the relevant memories for that vendor and asks an LLM to write the briefing from them, and only from them.
3. **The interface.** A single HTML file, no framework, no build step. A command bar for the vendor name, a history sidebar, and one card per briefing with edit, delete and copy.

I owned the third piece. The interesting part is that it isn't a thin shell. Most of the honesty of the product lives there.

## Why memory instead of a bigger prompt

Negotiation history is awkward data. It is small per vendor but spread across many months, and the useful signal is often a pattern rather than a fact. One of our records says a chemicals supplier quoted a 9% increase and refused to move on "best and final", and only capped it at 4% after we escalated to their VP of Sales with a competing quote. A retrieval system that returns the top few similar snippets will give you that record if you ask about that vendor. What we wanted was for the *lesson* to survive: this vendor ignores soft pushback.

That is the reason I wanted a memory system with a real model of retention and reasoning, not just a vector store behind a search box. Hindsight splits the work into [retain, recall and reflect operations](https://hindsight.vectorize.io/), and that split matched the way we already thought about the product. Retaining a negotiation outcome is a write. Recalling what we know about a vendor is a read that returns evidence. Writing a briefing is a separate step that sits on top of the evidence. If you want the broader background, Vectorize has a good explainer on [what agent memory actually is and how it differs from retrieval](https://vectorize.io/what-is-agent-memory).

I'll come back to why keeping evidence and inference apart mattered so much for the UI.

## The through-line: one function as the whole contract

The rule I set on day one was that the interface knows about exactly five backend functions and nothing else. Everything the browser needs from the outside world is behind them:

```js
const api = {
  async loadProfile()          { /* display name */ },
  async saveProfile(name)      { /* ... */ },
  async loadChats()            { /* history sidebar */ },
  async saveChats(chats)       { /* ... */ },
  async getBriefing(vendorName) {
    const res = await fetch('/api/briefing', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ vendor: vendorName })
    });
    if (!res.ok) throw new Error('Briefing service returned ' + res.status);
    const data = await res.json();
    return { text: data.briefing, memoryRetrieved: data.memory_used, draftEmail: data.draft_email };
  }
};
```

`getBriefing` returns `{ text, memoryRetrieved, draftEmail }`. That's the entire interface between the front end and the memory-backed service. It sounds trivial, but agreeing on it before anyone wrote backend code meant the prompt work, the Hindsight setup and the UI could all move at once. Nobody waited for anyone. For the first stretch, the function read from a local file of vendor records. Swapping in the real call changed one function body and touched nothing else.

I'd recommend this to anyone building a UI on top of something that doesn't exist yet. Don't mock the backend broadly. Mock one function, with a return shape you're willing to keep.

## The field that matters most: `memoryRetrieved`

I want to spend the most time on the boolean, because it's the piece I got wrong first.

My initial layout had a green badge on every AI reply reading "Hindsight Memory Retrieved". It looked good. It was also a lie whenever the service answered without recalling anything, and it would have stayed a lie in exactly the situations where a user most needs to know. A briefing built from zero memories reads just as fluent as one built from six.

So the badge is now driven by the response, and only by the response:

```js
if (e.result) {
  const r = e.result;
  body = (r.memoryRetrieved ? '<span class="badge"><i></i>Hindsight Memory Retrieved</span>' : '')
       + `<p>${esc(r.text)}</p>`
       + (r.draftEmail ? `<div class="email">...</div>` : '');
}
```

On the server side, `memory_used` is true only when recall returned at least one memory for that vendor. If it returns nothing, the card renders with no badge, and the text says there is no history for that vendor yet. That absence is information. A reader can look at a card and tell in a glance whether the advice is grounded in what the team has seen or is generic.

This is where Hindsight's design paid off for me. Because recall returns raw memories as its own step, separate from the step that writes the answer, the service always knows whether it had evidence. A system that fuses retrieval and generation into one opaque call would have made that flag much harder to compute honestly.

## The bug that taught me to check copy against data

The mock conversation I wrote at the start showed a briefing about a logistics vendor that "agreed to $3.96 per unit after a four-day delay." It read well. It also contradicted the records we were about to load, which said that vendor had a 98.6% on-time rate across 42 shipments, zero quality disputes, and had offered a 5% volume discount without being asked.

Nothing crashed. If a reviewer had asked why the sample and the data disagreed, I'd have had no good answer. I rewrote the sample from the actual record: ask for the 5% discount up front, address the email to the named account manager, and turn a $4.80 quote into $4.56.

The lesson is dull and worth stating anyway. Plausible generated text is not the same as text that is consistent with your data. Every number in an interface should trace back to something in the store.

## Vendor names are not questions

A smaller failure worth mentioning: my first lookup compared whatever was typed in the box against vendor names. Then someone pasted a whole sentence into it and got "No history found." The input was labelled as a vendor name but behaved like a search field, and users treat a text box like a search field no matter what the label says.

I made the matching more forgiving in the front end, but the real fix was on the service side, where recall takes a natural-language query and doesn't care whether it's a name or a sentence. Hindsight's recall runs several retrieval strategies in parallel, including semantic, keyword, graph and temporal, so a question like "how did we handle the delay with the fasteners supplier" and the bare name both find the same records. My job in the UI was to stop fighting that and pass the text through.

## History is not memory

The sidebar of past briefings looks like chat history, and that resemblance is a trap. I made a deliberate decision to keep the two apart.

The sidebar is a record of what *you asked and saw*. Hindsight's memory bank is what *the organization knows*. Deleting a card removes it from your history and does nothing to the memories behind it. Forgetting a fact about a vendor is a separate, explicit action, because the alternative is that tidying up your sidebar quietly degrades what the whole team's briefings are based on.

That separation shows up in what I persist. Transient UI state never gets saved:

```js
const persist = () => api.saveChats(
  chats.map(c => ({
    ...c,
    entries: c.entries.map(({ loading, editing, ...e }) => e)
  }))
);
```

`loading` and `editing` are stripped before saving, so reopening the page never shows a spinner that will never resolve.

Editing follows the same idea. Editing a card changes the vendor name and re-runs the briefing:

```js
async function run(entry) {
  entry.loading = true; entry.error = null; renderChat();
  try { entry.result = await api.getBriefing(entry.vendor); }
  catch (e) { entry.error = e.message || 'Unknown error'; entry.result = null; }
  entry.loading = false; persist(); renderAll();
}
```

An edited card can legitimately come back with a different answer than before, because memory may have changed in between. I preferred that to keeping a stale response under a new question.

## What it looks like in use

Here is a typical interaction, using a record for a solvents supplier.

I type `Solvex`. The card renders a skeleton for a moment, then a briefing with the memory badge:

> Don't open with "best and final". Last cycle Solvex refused to move until we escalated to their VP of Sales with a documented competing quote (BlueRock at $4.20/L against their $4.65/L). That capped a 9% increase at 4%. Bring a written competing quote and ask for the VP again.

A second vendor shows why per-vendor memory beats a general playbook. For a different supplier, the record says it ignored competitor quotes and bulk commitments for over a year, then offered 15% and a dedicated account manager the moment we sent a formal 30-day termination notice. The briefing for that vendor says to skip the soft tactics, and the one for the chemicals supplier says the opposite. A generic negotiation assistant would give both the same advice.

A third case is a vendor we have never dealt with. There's no badge, and the card says there is no history yet. Once the outcome of that negotiation is retained, the next briefing has something to work from.

## What I'd tell another engineer

1. **Define the seam before the system behind it.** One function with a fixed return shape let four people work in parallel. Keep the shape small and stable, and make it carry the facts the UI needs.
2. **Make the UI report provenance, not just answers.** A boolean like `memoryRetrieved` costs almost nothing and turns a fluent paragraph into something you can judge. Only show it when it is true.
3. **Keep evidence and inference separate.** Recall returning memories independently of the step that writes the answer is what made the provenance flag honest. Design for that separation early.
4. **Don't let history pose as memory.** If your interface has a list of past interactions, decide what deleting one means for the underlying store, and write it down.
5. **Check every number in generated copy against your data.** The sample text I wrote first was well written and wrong. Nothing in the tooling would have caught it.

None of this needed a framework. The whole interface is one file, and most of its usefulness comes from a handful of decisions about what it will and won't say. If you're building on top of a memory layer, I'd start by asking what your UI should refuse to claim, and then build the memory around that answer.
