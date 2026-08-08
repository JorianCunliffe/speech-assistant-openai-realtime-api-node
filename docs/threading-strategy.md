# Threading

How a day's worth of messy human communication — twenty calls, fifty emails, ten
texts, three meetings in a room and three on a screen — becomes a set of threads
that a project, a task, or an agent can be handed.

This is a design note, not a built thing. It exists because threading is the
piece that decides whether the orchestration engine is a filing system or a
coworker, and because getting the data model wrong here is expensive to undo
later.

## Contents

- [What already exists](#what-already-exists)
- [The one decision everything else hangs off](#the-one-decision-everything-else-hangs-off)
- [Five layers, and what must not be merged](#five-layers-and-what-must-not-be-merged)
- [The signal ladder](#the-signal-ladder)
- [Domain and thread are different questions](#domain-and-thread-are-different-questions)
- [Temporal proximity, done properly](#temporal-proximity-done-properly)
- [The calendar is the anchor](#the-calendar-is-the-anchor)
- [The model ranks, it does not label](#the-model-ranks-it-does-not-label)
- [Confidence buys autonomy](#confidence-buys-autonomy)
- [Threads and ProjectFlow](#threads-and-projectflow)
- [What to store](#what-to-store)
- [Where this lands in this codebase](#where-this-lands-in-this-codebase)
- [The human is not an approver](#the-human-is-not-an-approver)
- [How you will know it works](#how-you-will-know-it-works)
- [What will go wrong](#what-will-go-wrong)

## What already exists

More of the substrate is built than it might feel like.

`public.communications` is already the right shape: one row per communication
whatever channel produced it, with `channel`, `contact_id`, `occurred_at`,
`subject`, `body`, `summary`, a weighted `tsvector`, and a `(source_table,
source_id)` unique key so re-projecting a source row updates rather than
duplicates. Triggers project calls, SMS and recordings into it. Adding email and
calendar is another two projections, not a new architecture.

`public.projects` carries `aliases`, which is the thing that lets *"the
subdivision"* and *"lot 42"* resolve to a project without anyone typing an ID.

`attach_communication_project()` already does a primitive version of what this
note describes: three rules in priority order, and — importantly — it records
**which rule caught it** in `project_link_reason`. That instinct is correct and
under-exploited. The whole strategy below is that idea taken seriously: every
association carries its reason, its confidence, and its evidence, and can be
revised.

`context.js` already merges channels into wall-clock turns and already groups
calls and recordings into bounded conversations. It reads by *contact*. That is
the thing that has to change, and the change is the payoff.

## The one decision everything else hangs off

**A thread is an assignment, not a property of a message.**

The tempting move is `communications.thread_id`, set once at write time. Don't.
Threading is probabilistic and it will be wrong, and the cost of being wrong has
to be a cheap operation rather than a data repair.

Store it as its own record:

```
thread_assignments
  communication_id  -> communications
  thread_id         -> threads
  method            hard_id | calendar | participants | entity | temporal | model | human
  confidence        0..1
  evidence          jsonb   -- what actually caught it
  assigned_at
  superseded_by     -> thread_assignments   -- nullable
```

Three properties fall out, and all three matter:

- **Revisable.** New evidence arrives constantly — an email lands two days later
  that proves the call and the text were about different things. Re-threading is
  inserting a new assignment, not rewriting history.
- **Explainable.** When the system files a personal call under a work project,
  you can ask *why* and get an answer. A threading system you cannot interrogate
  is one you stop trusting after the second mistake, and one you stop trusting is
  one you stop using.
- **Trainable.** `method = human` rows are labelled data. Every correction the
  human makes is a training example for the priors, acquired for free, in the
  course of work they were doing anyway.

Threads themselves must merge and split cheaply. A merge writes new assignments
and leaves the dead thread as an alias so old references still resolve; a split
is the same operation in reverse. Design for this on day one — a system that
cannot merge threads will grow one enormous thread per contact, which is exactly
the per-contact model you already rejected, arrived at by accident.

## Five layers, and what must not be merged

Keep these strictly separate. Every failure mode I can think of comes from
collapsing two of them.

1. **Capture** — a communication happened, normalised to a common shape.
   *Already built.*
2. **Identity** — which contacts were involved. Not one contact: a *set*.
3. **Domain** — work / personal / health / spiritual. A property of the thread,
   mostly inherited from the relationship.
4. **Thread** — which ongoing conversation this belongs to.
5. **Work** — which project, and which task. Threads link to work; they are not
   work.

The specific collapse to avoid is **3 into 4**. Classifying a message as
"work" tells you almost nothing about *which* work, and threading a message
correctly does not require knowing its domain first. They run in parallel, and
each is evidence for the other, but neither is a gate on the other. Building
domain classification as step one and threading as step two means every
misclassification becomes a misthread, and you have coupled your two error rates
together for no gain.

The second collapse to avoid is **4 into 5**. Threads and projects are different
objects with different lifetimes. Most threads have no project. New work arrives
*as* a thread before the project exists — that is the normal case, not an edge
case, and a model that requires a project to file a conversation cannot represent
the moment work begins.

## The signal ladder

Run signals cheapest-and-strongest first. The expensive semantic step should
only ever see the residue.

**Rung 1 — hard identifiers. Certain. Never overridden.**

- Email `Message-ID` / `In-Reply-To` / `References`. RFC 5322 threading is a
  solved problem and Nylas hands you a `thread_id` outright. Use it. An email
  reply is not a 0.9-confidence guess.
- Calendar event ID / iCal UID on anything derived from a meeting.
- `sms_threads.id`, `twilio_call_sid`, WhatsApp quoted-message id.
- Anything a human explicitly filed.

These do not go to a model. Ever. A model asked to re-derive what a
`References` header already states will occasionally disagree with it, and that
is a pure loss.

**Rung 2 — structural.** Participant-set overlap and calendar containment.
Cheap, deterministic, and strong. Details below.

**Rung 3 — temporal.** Proximity as a decay, not a window. Details below.

**Rung 4 — entity mentions.** Underrated, and probably the highest
value-per-effort item on this list. Extract hard tokens from every
communication into a normalised table: project names and aliases, lot numbers,
addresses, invoice and PO numbers, quote references, street names, unit numbers.

These are *join keys*. A call transcript saying *"the retaining wall on
forty-two"* and an email subject reading *"Lot 42 — retaining wall quote,
revised"* have no shared thread id, may have no shared participant, and may be
four days apart. The shared entity threads them with certainty that no
similarity score will ever match. In a construction and property context, the
entity vocabulary is small, closed, and enormously discriminative. Build this
before you build embeddings.

**Rung 5 — semantic.** Full-text (`communications.search` is already there,
already weighted) and then, only for what is left, a model.

The ordering is the point. On a normal day, rungs 1–4 should place the large
majority of communications. Rung 5 exists for the genuinely ambiguous residue,
which is where the model is actually worth its latency and cost.

## Domain and thread are different questions

You framed domain classification — work, personal, health, spiritual — as the
first step. I'd move it, and change what it attaches to.

**Domain is a property of the relationship first, the thread second, and the
message barely at all.** Bruce is a work contact. He is a work contact in the
morning and a work contact at 9pm. The prior on any given communication from
Bruce being work-related is not 25% across four categories; it is something like
0.95, and it was 0.95 before you read a word of the message.

So model it as a distribution on the edge:

```
contact_domains
  contact_id
  domain          work | personal | health | spiritual | admin | ...
  weight          0..1, sums to 1 across domains for a contact
  observations    int
  updated_at
```

Seed it from the CRM — a contact on a project team is work; a contact tagged
family is personal — then update it from confirmed threads. Now classification
of a new message is a cheap Bayesian update rather than a from-scratch judgement:
the prior does most of the work, and the message content only has to move it.

Two things this buys you beyond accuracy:

- **The awkward cases become visible instead of wrong.** The contacts whose
  distribution is genuinely mixed — your accountant who is also a friend, the
  builder who is also in your congregation — are exactly the contacts where
  `weight` is spread. That is a computed signal that this contact needs the
  message read, and it is available before you read it.
- **Domain becomes an access boundary, not a tag.** Once a thread is labelled
  personal or health, that label should *do something*: keep it out of project
  prompts, out of a work agent's retrieval, out of a report. Given the
  orchestration engine will have agents acting with your identity across email
  and voice, the ability to say *"this class of thread is not visible to work
  automation"* is a safety property, and it is much easier to enforce if the
  label exists from the moment the thread does. `{{history}}` in this codebase
  already carries caller text into a system prompt; the same mechanism carrying
  a health conversation into a subcontractor's call is a different order of
  mistake.

## Temporal proximity, done properly

Your instinct — a call, then a text half an hour later, is probably the same
thing — is right, and it deserves to be a function rather than a rule.

Model it as a decay:

    P(same thread | Δt) ∝ exp(−Δt / τ)

with **τ set per channel pair**, because the natural rhythm of a follow-up
depends entirely on which two channels are involved:

| From → To | τ (rough) | Why |
|---|---|---|
| call → SMS | ~1 hour | *"as discussed, here's the address"* — minutes, usually |
| meeting → email | ~4 hours | minutes, actions, the thing you promised in the room |
| call → call | ~1 day | called back, or they missed you |
| email → email | ~3 days | email tolerates latency; people reply on Tuesday to a Friday mail |
| SMS → SMS | ~2 days | a thread that runs for months, punctuated |
| any → any (baseline) | ~7 days | the floor, so nothing decays to zero unfairly |

Do not hardcode these as constants buried in a function. Put them in a table.
They are hypotheses, they are wrong, and once `method = human` assignments
accumulate you can fit them from your own data instead of my guesses. That is
the single cheapest accuracy win available after the first month of use.

Two guards:

- **Temporal proximity is a multiplier on a candidate, never a candidate
  generator on its own.** Two communications being close in time is meaningless
  without a shared participant, entity, or project. Otherwise every busy Tuesday
  afternoon becomes one thread.
- **Bound the total span.** A thread that has been silent for sixty days should
  go dormant, and a new communication should prefer to open a fresh thread over
  resurrecting it. Dormancy prevents the slow slide into one thread per contact
  that eats every threading system that lacks it.

## The calendar is the anchor

Of everything on the list, calendar integration is the highest-leverage item,
and it is worth understanding why it is structurally different from email.

Three in-person meetings and three online meetings a day are the communications
with the *least* metadata. There is no message id, no sender, no transcript
unless someone recorded it. In a purely message-based model they are invisible,
and they are frequently the most consequential thing that happened.

A calendar event is not a message. **It is a thread seed with a declared
participant set, a declared subject, and a time box** — the richest single
object in the entire pipeline, and it exists *before* the communication happens.
That makes it uniquely useful:

- Anything from those participants inside the event window, or within a few
  hours after it, is presumptively that thread. This is rung 2, and it is strong.
- An event title containing a project alias links the meeting *and everything
  that clusters around it* to ProjectFlow in one hop, with a reason of
  `calendar_alias` rather than a guess.
- A conferencing link on the event ties an uploaded recording to the event, and
  therefore to the thread, with no matching required at all. `recordings`
  already accepts external sources; the event id is the join.
- The attendee list is a **participant set you did not have to infer**, which is
  what lets a group meeting's follow-ups thread correctly when two of the six
  attendees email each other afterwards.

Nylas giving you mail and calendar behind one auth is convenient, but calendar
is the part that changes what is possible. If you sequence the integration work,
do calendar first, even though email is the larger volume.

The corollary: get in the habit of the in-person meeting having a calendar
entry, even retrospectively. A one-line event created after the fact is enough
metadata to thread a conversation that otherwise has none — which makes
*"create the calendar entry for the meeting that just happened"* a genuinely
valuable thing for the voice assistant to do, and a nice small first
coworking loop.

## The model ranks, it does not label

For the residue that rungs 1–4 leave unplaced, the shape of the model call
matters more than the model.

**Do not** ask a model to read a message and emit a category and a topic. That
is unbounded generation, unevaluable, and it will invent thread names that drift.

**Do** generate candidates deterministically and ask the model to choose:

```
Here is a communication: <channel, participants, time, subject, body/transcript>.
Here are the candidate threads it may belong to:
  A. "Lot 42 retaining wall" — 3 participants, last activity 2h ago, project Arkendeith
  B. "Council DA submission" — 5 participants, last activity 2 days ago, project Arkendeith
  C. "Bruce — personal" — 1 participant, last activity 3 weeks ago
Or: NEW thread, or UNSURE.
Return the choice, a confidence, and the evidence you used.
```

Retrieval-then-rank. It is bounded, it is cheap, it is auditable, the output
space is enumerable, and `UNSURE` is a first-class answer rather than something
the model has to be tricked into. Candidate generation comes from participant
overlap, entity mentions, project membership and temporal decay — the rungs
above — capped at five or so candidates ranked by prior score.

Two hard rules:

- **The model may not override a hard identifier.** If `In-Reply-To` says which
  thread it is, the model is not consulted.
- **The model's output is text derived from untrusted input, and it now drives
  routing.** An email containing *"this relates to the Arkendeith project"* can
  misfile things, deliberately or accidentally. This codebase already documents
  prompt injection as a live concern for `{{history}}`; threading raises the
  stakes, because a thread is becoming an access boundary. Text-derived signals
  must never outrank identifier-derived ones, and no extracted text should ever
  grant someone access to a thread they are not a participant of.

Run this asynchronously. **It must not go in the trigger.** The migration
already notes that `calls_to_communications` fires while a call is live; putting
a model call behind that would put multi-second latency on the voice path, which
is the one thing this project has spent real effort protecting. The trigger does
rungs 1–3 inline (all cheap SQL), marks the row `needs_threading` when it cannot
place it, and a worker does the rest.

## Confidence buys autonomy

One threshold is not enough, because the cost of being wrong is not constant.

Band the behaviour:

| Confidence | Behaviour |
|---|---|
| ≥ 0.90 | File silently. It is in the thread; nobody is told. |
| 0.60–0.90 | File, and show the assumption where the work appears — *"filed under Lot 42 · move"*. Reversible in one click. |
| < 0.60 | Unplaced queue. Visible, counted, not guessed at. |

And gate **actions** separately from **filing**. Filing a note into the wrong
thread is a nuisance. Sending a reply, or handing an AI agent work, on the wrong
thread is a real-world mistake in front of a client. The confidence required to
*act* on a thread should be higher than the confidence required to *file* into
it — and that difference should be a property of the action in the orchestration
engine, not a constant.

The metric that matters is therefore **precision on the silent band**, not
overall accuracy. An unplaced item costs a few seconds of attention. A silently
misfiled item costs trust, and it costs it permanently.

## Threads and ProjectFlow

```
communication ──< thread_assignment >── thread ──< thread_project_link >── project
                                          │                                  │
                                          └── domain, participants[]         └── task
```

- **Many threads per project.** A project accumulates threads; a thread has at
  most one project.
- **A thread with no project is normal**, and the set of them is a work queue —
  it is precisely where new work is arriving, and surfacing it as *"six threads
  that don't belong to anything yet"* is more useful than any dashboard.
- **Promotion is a real operation.** *"This thread is now a project"* should be
  one action that creates the project, links the thread, and back-links its
  history. That is how work actually starts.
- **Link to tasks, not just projects.** This is the handoff surface. A thread
  linked to a task is what lets an AI agent pick up *"chase the retaining wall
  quote"* with the full conversation attached, and what lets it hand back with
  the reply threaded into the same place. Project-level linking alone is too
  coarse for the coworking loop — it tells an agent what the work is about but
  not what the work *is*.
- **Team membership from the CRM is a threading signal, not just a display
  field.** A communication from someone on exactly one project team has a strong
  prior for that project — which is what `attach_communication_project()`
  already does with its `contact` rule and its correct insistence on *exactly
  one*. Generalise that: with two or three memberships, don't discard the signal,
  narrow the candidate set with it and let the later rungs choose.

## What to store

You have date, time and transcript. Here is the rest, split by what it costs.

**Free — the provider already knows it.** Capture all of it, including fields
with no use today; it is far cheaper to store now than to backfill.

| Field | Notes |
|---|---|
| `external_message_id`, `in_reply_to`, `references[]` | Rung 1. Non-negotiable for email. |
| `provider_thread_id` | Nylas gives it. Free, correct threading for email. |
| `calendar_event_id`, `ical_uid` | The anchor. |
| `participants[]` with role | from / to / cc / attendee / caller / callee. **A set, not a contact.** |
| `direction`, `duration`, `answered`, `status` | Call logs. A 6-second unanswered call is not a communication. |
| `subject`, `snippet`, `attachments[]`, `labels[]` | Provider folders and labels are a free human-authored signal. |
| `source_uri` | Back to the original, always. |

**Cheap to derive — SQL, at write time, in the trigger.**

| Field | Why |
|---|---|
| `participant_set` (sorted contact ids, hashed) | Makes overlap a join instead of a scan. |
| `delta_to_prev` per participant | The temporal signal, precomputed. |
| `is_business_hours`, `day_of_week` | Weak domain signal, free to compute. |
| `is_first_contact` | Never threads to anything. Worth knowing early. |
| `length`, `turn_count` | Filters trivia out of candidate generation. |

**Costly — model-derived, asynchronous, always with confidence and evidence.**

| Field | Why |
|---|---|
| `mentions[]` → normalised entity table | Rung 4. Highest value here by a distance. |
| `domain` + confidence | Prior-updated, not from scratch. |
| `commitments[]` — who owes what by when | Not threading, but the same extraction pass, and it is what feeds tasks. |
| `thread_assignment` + confidence + evidence | The output. |

The entity table deserves its own index and its own care:

```
mentions
  communication_id
  kind        project_alias | lot | address | invoice | person | date | amount
  raw         "forty-two"
  normalised  "lot-42"
  confidence
```

`raw` and `normalised` both, because a voice transcript says *"forty-two"* and
an email says *"Lot 42"*, and you need the normalised form to join and the raw
form to explain the join to a human who is asking why.

## Where this lands in this codebase

Concretely, in dependency order:

1. **Extend `communications`.** Add `participants uuid[]`, `external_thread_id`,
   `calendar_event_id`, `in_reply_to`, `domain`, `needs_threading boolean`. The
   table's existing `(source_table, source_id)` key and its trigger-projection
   pattern both survive unchanged — this is additive.
2. **Add `threads`, `thread_assignments`, `mentions`, `contact_domains`,
   `channel_decay`.** New tables, no changes to existing writers.
3. **Generalise `attach_communication_project()`** into
   `attach_communication_thread()`, keeping its existing structure: rules in
   priority order, each recording its reason. It gains rungs 1–3, and gains the
   ability to give up and set `needs_threading` instead of guessing. Its current
   project rules survive as the thread→project link rules, one level up.
4. **A worker for the residue.** Polls `needs_threading`, generates candidates,
   calls the ranker, writes assignments. Off the call path entirely.
5. **Calendar and email providers in `context.js`.** Already planned —
   `PLANNED_CHANNELS` lists them, `resolveSubject` already loads `email`, and
   the provider interface takes them without modification. Nylas fills both.
6. **A thread provider in `context.js`.** This is the payoff, and it is worth
   pulling forward.

That last point deserves expanding, because it is the part that pays for the
rest before the orchestration engine exists.

`getContext` and `searchHistory` currently read **by contact**: everything Bruce
ever said, merged and budgeted. On a live call that is the wrong window, and the
budget-trimming logic in `applyBudget` exists to paper over exactly that
mismatch. What the assistant actually wants when Bruce rings is *the two or
three live threads with Bruce, most recent first* — which is a fraction of the
tokens, far more relevant, and directly answers the question the caller is about
to ask.

So threading is not only infrastructure for the coworking engine. It makes the
voice assistant measurably better on the next call after it ships, using code
paths that already exist. Add a thread-aware provider, let `{{history}}` prefer
active threads over raw recency, and the same work serves both products.

## The human is not an approver

The thesis is that the human is a coworker, not an approval gate. That has a
specific consequence for this system, and it is a design constraint rather than
a philosophy:

**Never present a queue of things to classify.** Nobody will do it twice. A
system that requires labelling to work has already lost, because the labelling
happens at the moment of lowest motivation and highest volume — the end of the
day, with fifty items in a list.

Instead:

- **Act under the assumption and show the assumption in place.** *"Filed under
  Arkendeith — move?"* attached to the work, not in a separate inbox. The
  correction costs one click, at the moment the human is already looking at that
  thread, with the context loaded in their head. That is when a correction is
  nearly free; a queue asks for the same information at the moment it is most
  expensive.
- **Budget the interruptions.** Cap corrections surfaced per day — five, ten —
  and choose them by *expected information gain*, not by raw uncertainty. The
  ambiguity that will misroute the next thirty communications is worth asking
  about. The one-off that will never recur is not. This is the difference
  between a colleague who asks a good question at the right moment and one who
  asks about everything.
- **Every correction updates a prior**, visibly. `contact_domains`, the decay
  table, project alias lists. The human should be able to feel it getting
  better, because that is what makes the next correction worth making.
- **Handoffs travel as threads.** When an agent hands work to a human or takes
  it back, the unit that moves is the thread — full history, every channel,
  the current assumption and its confidence. Not a task with a link to a
  transcript. That is what makes it feel like being handed work by a colleague
  rather than being assigned a ticket.

## How you will know it works

Tuning τ, the thresholds and the candidate cap needs a number, and you cannot
get one from intuition.

**Bootstrap ground truth from what is already threaded.** Email has RFC
threading; calendar events have attendee sets. Take sixty days of history, hide
the hard identifiers, run the pipeline, and measure whether rungs 2–5 recover
the threads that rung 1 already knows. That is a real accuracy number, on your
own data, before shipping anything — and it costs one script.

Then measure three things continuously, and only three:

- **Precision in the silent band** (≥0.90). Target: very high. This is the one
  that matters. Every miss here is trust spent.
- **Unplaced rate.** How much lands in the queue. Should fall over time. If it
  does not fall, the priors are not learning.
- **Correction rate by method.** Which rung produces the most human corrections.
  That tells you where to spend the next week of effort, and it is the only one
  of the three that is directly actionable.

Resist adding a fourth. Overall accuracy in particular is a misleading number
here, because it averages a cheap error and an expensive one.

## What will go wrong

- **Over-threading.** Everything with Bruce becomes one thread. Mitigated by
  dormancy, by a maximum span, and by making split cheap. Watch for threads with
  a single participant and unbounded lifetime — that is the shape of the failure.
- **Thread drift.** A thread genuinely changes subject halfway through. That is
  a split, and the system should notice it: a sharp drop in entity overlap
  between the recent tail of a thread and its earlier body is a detectable
  signal, and probably worth a scheduled pass rather than an inline check.
- **The 65% intuition is a prior, not a rule.** It is right on average and wrong
  in exactly the cases you will remember — the personal call that follows the
  work call. Which is why temporal proximity must be a multiplier on a candidate
  rather than a candidate generator: it should never be able to thread two
  things that share nothing but a clock.
- **Injection through routing.** Now that untrusted text influences where
  things are filed and what agents can see, misfiling becomes an attack rather
  than just an annoyance. Identifiers over text, always; participation over
  assertion, always.
- **Silent degradation.** If the worker stalls, communications keep arriving and
  keep being unthreaded, and nothing looks broken — it just gets gradually less
  useful. `needs_threading` should have an age alarm, in the same spirit as the
  rest of this codebase's insistence that a failure be legible as a failure
  rather than an empty result.

## If you only do three things

1. **Threading as a revisable assignment with a reason**, not a column. Nothing
   else on this list survives without it.
2. **Calendar before email.** It is the anchor, it rescues meetings from having
   no metadata at all, and it links to projects in one hop.
3. **Entity extraction into a joinable table.** More threading accuracy per hour
   of work than any model, and it is the only rung that threads a voice
   transcript to an email with no shared participant and no shared time.
