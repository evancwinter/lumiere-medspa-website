# Winter Growth Systems — Chatbot Build Playbook

Everything learned building Elise for the Lumière demo, written so the next one is faster and at least as good.

**How to use this:** at the start of a new build, tell Claude *"read CHATBOT-BUILD-PLAYBOOK.md and follow it"* and give it the client's website. Most of what follows is copy-paste prompt text with placeholders in `{{braces}}`.

---

## 0. Gather before you start

Do not begin building until you have these. Every one of them ends up inside the bot.

| What | Why it matters |
|---|---|
| The live site (or the prototype HTML) | The bot's facts must come from here, or it will contradict the page |
| Exact prices, packages, durations | Never let the bot invent a number |
| Who the practitioner is, credentials | Trust is most of the sale in these industries |
| The brand's actual voice | Steal phrases straight off the page |
| What they will NOT do | The strongest differentiator, and the best guardrail |
| A phone number for escalation | Every bot needs a human exit |
| What counts as a conversion | Booking? A phone number? Changes the whole design |

**Read the client's page source before writing anything.** On the Lumière build, the page already contained a three-question qualification quiz with recommendation logic in the JavaScript. Reusing that logic verbatim meant the bot and the site could never disagree. That was the single highest-value thing in the whole build, and it came from reading the code rather than the copy.

---

## 1. Build order

Do it in this order. Each step depends on the one before.

1. `voiceflow_workspace.list` → get workspaceID
2. `voiceflow_project.create` → type `webchat`. Save the **projectID** and note both IDs it returns (see §5)
3. `voiceflow_agent_global_prompt.update` → identity, goal, tone, guardrails
4. `voiceflow_project.get_api_key` → needed for knowledge base calls
5. `voiceflow_document.create_text` × 3–5 → the knowledge base
6. `voiceflow_playbook.create_batch` → the conversation flows
7. `voiceflow_routing.update` → make playbooks reachable
8. `voiceflow_agent_instructions.update` → when to pick each playbook
9. `voiceflow_environment.compile` → **required after every change, or tests run against stale config**
10. `voiceflow_test_conversation.interact` → test before publishing
11. `voiceflow_environment.publish` → ship it
12. Embed on the site, then set appearance in the dashboard Widget tab

---

## 2. The three-playbook structure

This shape works for any appointment-based local business. Rename to fit.

**Answer Questions** — factual lookups. Prices, duration, policies, credentials. Answers, then pivots to a qualifying question. Never dead-ends.

**Qualify and Recommend** — the money path. Someone with a problem and no vocabulary for it. Runs the funnel, names a treatment and a price, hands to booking.

**Book a Consultation** — closes. Points at the real booking mechanism, clears last objections, stops selling once they've agreed.

### The qualification funnel

Three questions, **one per message**, in this order:

1. **What's the problem?** (in their words, mapped to your categories)
2. **What's the timeline?** (event soon / next few months / just exploring)
3. **What's the budget?** (three bands, asked lightly — public pricing is a selling point)

Then: *"{{Practitioner}} would likely start with {{treatment}}, which is {{price}}. She'll confirm that, or talk you out of it, at the free consult."* → booking.

**Rules that make it work:**
- If their opening message already answers a step, skip it. Never make someone repeat themselves.
- If they answer two questions at once, take both.
- If they refuse the budget question, ask once and move on.
- Write the mapping table explicitly into the playbook instructions, including the awkward branches (budget too low for the obvious treatment). Say so honestly rather than pretending it fits.

---

## 3. Copy-paste prompt blocks

### 3a. Anti-AI-tells (use verbatim, every build)

This is the highest-value block in this file and it is not industry-specific.

```
<sound_human>
This is the highest priority rule in this prompt. People spot AI writing
instantly, and a {{business type}} cannot afford to sound automated.

PUNCTUATION, absolute bans:
- NEVER use an em dash. Not once, ever. Use a comma, a period, or just start
  a new sentence. This is the single biggest giveaway that a machine wrote something.
- No semicolons. Nobody at a front desk uses a semicolon.
- No trailing "..."

BANNED OPENERS. Do not begin a message with any of these:
"Honestly," • "Great question" • "I hear you" • "You're not alone" •
"Absolutely" • "Of course" • "I completely understand" • "That's such a"
Just start with the actual content instead.

BANNED PHRASES anywhere in a message:
"one of the most common things I hear" • "you're in good hands" • "let me know
if you have any questions" • "feel free to" • "I'd be happy to" • "that said" •
"rest assured" • "it's not just X, it's Y" • "whether you're X or Y" • "when it
comes to" • "dive into" • "journey" • "elevate" • "curated" • "seamless" •
"tailored" • "personalized experience" • "look no further"

STRUCTURE:
- Never list three parallel things in a row. Two reads human, three reads generated.
- Never restate the question before answering it. Just answer.
- Vary sentence length hard. A four-word sentence next to a longer one sounds
  like a person. Three evenly balanced sentences sound like a machine.
- Never reuse an opener or phrase already used in this conversation. Repetition
  across turns is what makes someone realize they're talking to software.
- Vary how you ask the closing question.
</sound_human>
```

**Critical:** the rule alone does not work. **Your WRONG/RIGHT examples must also be free of em dashes**, or the model copies them straight out of the examples. That is exactly what happened on the Lumière build.

### 3b. Length rule

```
LENGTH IS A HARD RULE. One to three sentences. ONE paragraph. Never use a line
break to start a second paragraph. One question per message, at the end.
```

Long-windedness is the default failure mode of every chatbot. Set this on day one rather than fixing it later. Also spread multi-step flows **across turns** — if the playbook says "acknowledge, ask, explain, price, offer," the model will do all five in one message unless you tell it the flow spans several turns.

### 3c. Tone with worked examples

Abstract tone descriptions do not constrain output. Every tone section needs WRONG/RIGHT pairs covering **both** failure modes:

- too cold and clinical (reads like a form letter)
- too bubbly and empty (says nothing, sells hard)
- dead-ended (no next step)
- canned disclaimer (for regulated industries)

Write the client's actual bad output into the WRONG slot as you find it. That works far better than describing the problem.

### 3d. Emoji policy

```
<emoji>
Roughly one in every two or three messages. Always use one when delivering good
news or reassurance. At most ONE per message, at the end of a warm clause,
never mid-sentence, never stacked.

Use only: ✨ 😊 🤍 🙌
Never: 💕 💅 💋 🔥 😍

NEVER use an emoji when: quoting a price someone may find steep, deflecting a
question to the practitioner, responding to a complaint or complication, giving
out the phone number, or replying to anyone who sounds upset. Warmth is right in
those moments. Cheerfulness is not, and an emoji reads as though you weren't listening.
</emoji>
```

"Sparingly" alone gets read as "never." Give it a rate.

### 3e. Guardrails (adapt per industry)

```
<data_handling>
- NEVER state a price, package, timeframe, or service that isn't in the knowledge
  base. If you don't have it, say so and give {{phone}}.
- NEVER invent availability or confirm a booking.
- Do NOT collect health or financial information. If someone volunteers it,
  acknowledge in one clause and move on. Ask no follow-ups.
- Never discuss other customers.
- Never promise an outcome.
</data_handling>

<escalation>
- Anything requiring professional judgement goes to {{practitioner}} in person.
  Warmly and specifically, never as a canned disclaimer.
- Anyone describing a problem with work already done: direct them to call {{phone}}
  rather than handling it in chat.
- Anyone who sounds distressed: drop the sales flow entirely.
</escalation>
```

### 3f. The router rule

The Main Agent will answer messages itself instead of routing, which silently bypasses your funnel. The reply looks fine; only the trace reveals it. Put this in the agent instructions:

```
YOU ARE A ROUTER, NOT A RESPONDER. You do not answer substantive messages
yourself. You hand them to a playbook, which writes the reply. The ONLY two
messages you ever write are the opening greeting and a single clarifying
question. If you find yourself composing an answer, stop and hand off.

Whenever you hand off, produce NO user-facing text of your own on that turn.
Emitting a message alongside a handoff shows the visitor two messages in a row.
```

---

## 4. The opening message

Do **not** use "How can I help you?" It is harder to answer than it looks — people freeze because they don't know what's fair to ask.

Pattern that works:

```
Hi, I'm {{name}} 😊 What brought you here today, anything specific or just having a look?
```

It names her, asks what brought them, and hands them a two-word way out. Then **make sure vague answers route into the funnel** — "just looking," "idk," "not sure," one-word replies. Those are not dead ends, they're your highest-volume entry. The playbook should take the pressure off in one clause and ask one easy question: *"No problem. What made you click, out of curiosity?"*

---

## 5. Voiceflow gotchas that cost real time

**Two different environment IDs.** `project.create` returns an environment object with an `id` AND a `draftVersionID`. They are not interchangeable:
- Agent tools (global prompt, instructions, playbooks, routing, compile) want the **draftVersionID**
- Knowledge base and document tools want the **environment id**
- `environment.publish` wants the **environment id**

Getting it wrong returns `Version does not exist` or `Environment not found`.

**Compile after every change.** Tests run against the last compiled version. Skipping this makes you think a fix didn't work.

**Appearance is dashboard-only — with one exception.** Name, avatar, colors, description, placeholder and launcher label are set in the **Widget** tab; passing them in code does nothing. The exception is `stylesheet`, which *does* work in code but **only nested under an `assistant` block** (see below). Earlier versions of this playbook said the snippet accepts no `assistant` block at all. That was wrong, and it cost a silently dead stylesheet on the Lumière build.

**Never hand-write the embed snippet.** Get it from the Widget tab. A hand-written `versionID: 'production'` will make every request fail with `Cannot resolve version alias`, and the widget silently never appears.

**`stylesheet` IS supported in code — but it MUST be nested under `assistant`.** It injects a CSS file into the widget's shadow DOM, so you can brand the launcher, header and buttons.

```js
window.voiceflow.chat.load({
  verify: { projectID: '...' },
  url: 'https://general-runtime.voiceflow.com',
  assistant: {                                        // <-- required wrapper
    stylesheet: window.location.origin + '/widget.css'
  }
});
```

**At the top level it fails silently — and this is nasty.** No console error, no `<link>` in the DOM, no missing-file warning. The widget just keeps the dashboard's colors, which are usually close enough to the brand that the page looks correct. On the Lumière build the stylesheet was dead from day one and nobody caught it for weeks; the giveaway was that the red unread badge never appeared, and even that read as a widget quirk rather than a broken file.

**Verify it loaded, don't eyeball it.** In the console:

```js
const sr = document.getElementById('voiceflow-chat').shadowRoot;
[...sr.querySelectorAll('link[rel=stylesheet]')].some(l => l.href.includes('widget.css'));  // must be true
getComputedStyle(sr.querySelector('.vfrc-launcher')).backgroundColor;                        // must be YOUR hex
```

Use `window.location.origin` rather than a hardcoded domain, so preview deployments load their own copy instead of production's.

Class names: `.vfrc-launcher`, `.vfrc-launcher__container`, `.vfrc-launcher__label`, `.vfrc-header`, `.vfrc-chat`, `.vfrc-button`, `.vfrc-proactive`, `.vfrc-proactive-message`.

**The unread badge gets clipped unless you release the launcher's overflow.**
Voiceflow sets `overflow: hidden` on `.vfrc-launcher` alongside
`border-radius: 9999px`. A badge drawn as an `::after` at negative offsets is
then clipped twice — by the box edge and again by the pill's rounded corner —
leaving roughly half the circle showing. It looks like a badly positioned badge
rather than a clipping problem, so it invites fiddling with `top`/`right` that
can't ever fix it.

```css
.vfrc-launcher { overflow: visible !important; }   /* let the badge sit proud */
```

The launcher is auto-width and its contents fit, so nothing else escapes. The
alternative — moving the badge inside the pill — works but reads as decoration
rather than a notification. Note `.vfrc-launcher__container` one level up is
already `overflow: visible`, so that's the other place a badge can live.

**Mobile: the launcher collides with a sticky CTA bar.** Any design with a bottom-fixed mobile CTA shares that corner with the launcher. Measure the bar, then shift the launcher's *container* (not `.vfrc-launcher`, which is `position:relative` for the badge) and the proactive bubble by the same amount so their spacing survives:

```css
@media (max-width: 860px) {
  .vfrc-launcher__container { bottom: calc(93px + env(safe-area-inset-bottom)) !important; }
  .vfrc-proactive           { bottom: calc(113px + env(safe-area-inset-bottom)) !important; }
}
```

Defaults are `bottom:20px` for the launcher container and `40px` for the bubble; both sit inside `.vfrc-widget` (fixed, full height). Include `env(safe-area-inset-bottom)` or it will look right in a desktop emulator and wrong on a real iPhone.

**Cheap models degrade the conversation that matters.** `voiceflow-flash-4.1` cut cost 65% and was 5× faster, but truncated the nervous-first-timer flow and drifted into promising results. Keep the quality model on any playbook doing persuasion.

**Handoffs cause duplicate messages. This is the single most recurring bug.** It appeared three separate times on the Lumière build, in three different disguises:

1. The router wrote its own reply *and* handed off, so the visitor saw two messages.
2. The router greeted *and* handed off on the same turn.
3. A playbook confirmed a booking, handed control back, and the router then restated the same confirmation in different words.

Two rules fix all three, and both belong in every build:

```
- Whenever you hand off to a playbook, produce NO user-facing text of your own
  on that turn. Call the handoff and stay silent. The playbook writes the reply.
- If a playbook has just handed control back to you, say NOTHING unless the
  visitor sends a new message. Never restate what the playbook already said.
```

Plus, in any playbook that ends a task (booking, order, ticket):

```
Do NOT hand back on the same turn you deliver the confirmation. Send it and end
the turn inside this playbook. Hand back only on a LATER turn.
```

**When you add a capability, hunt down every rule written for when it didn't exist.** Stale instructions are worse than missing ones, because the agent obeys them.

On Lumière, the bot originally could not book, so the qualify playbook said: *"never imply you can book, say 'want me to show you where to book' instead."* Later it gained real booking tools. That old line stayed. Result: it offered to point the visitor at a form, the visitor said "ok," and it booked them instead. The words and the behaviour had silently drifted apart.

Before shipping any new capability, grep your own prompts for the negation you wrote earlier: "never," "cannot," "you don't have," "instead." Every one of those is a candidate.

**Read the trace, not just the reply.** Every real bug on this build produced replies that read perfectly fine: the router answering instead of routing, the greeting doubling up, the duplicated confirmation, the stale booking offer. None were visible by reading the conversation. You only catch them by checking *which component* produced each message.

---

## 6. Website integration

### Notification instead of auto-open

Auto-opening gets closed reflexively. A badge gets clicked. Four seconds after load, once per browser session: chime + red "1" + proactive bubble + launcher reading **"Reply to {{name}}"**.

Working implementation lives in `Landing page prototype review/deploy/index.html` (`lumiereNudge()`), with styles in `widget.css`. Copy both.

Key API: `window.voiceflow.chat.proactive.push({ type:'text', payload:{ message:'...' }})` shows a bubble **without** opening the window. `proactive.clear()` removes it.

**Browsers block audio until the visitor interacts with the page.** The chime cannot play on a cold load. The code falls back to firing on their first click or scroll. The badge is always reliable; the sound is not. Say this to clients before they ask.

### Hosting

Cloudflare Pages, on a subdomain of `wintergrowthsystems.com`:

```
npx wrangler pages project create {{client}} --production-branch=main
npx wrangler pages deploy "{{folder}}" --project-name={{client}} --branch=main
```

Then a CNAME at Namecheap (`{{client}}` → `{{project}}.pages.dev`) and add the custom domain in the Cloudflare dashboard. Free, commercial use allowed, no bandwidth ceiling.

**Cloudflare's edge serves stale content for up to a minute after deploy.** Verify against the direct `https://{{hash}}.{{project}}.pages.dev` URL before concluding a change didn't ship. Do not use `?cachebuster=` query strings on static assets — Pages may fall back to serving index.html.

---

## 7. Test before every publish

Run these and read the traces. Any failure is a fix, not a note.

**Funnel:** a vague concern in the customer's own words → should qualify in ≤3 questions and land on a named service with a price.
**Facts:** a price question → must match the website exactly. Any other number is a hallucination.
**Refusal:** something requiring professional judgement → must decline warmly and route to a human, never advise.
**Complaint:** a problem with previous work → must give the phone number, not troubleshoot.
**Scope:** a service they don't offer → must say so, not invent it.
**Booking:** "book me Thursday at 2" → must offer only real slots, never invent one, and never confirm before the tool returns a code.
**Booking, all at once:** give name, phone, email and a time in a single message → must take all four and skip straight to confirming.
**Booking, junk data:** give "12345" as a phone → must reject it and re-ask for only that field, not restart.
**Booking, closed day:** ask for a Sunday → must say they're closed and offer the nearest open slot.
**Booking, after confirming:** check the confirmation appears exactly ONCE.
**Distress:** someone upset about themselves → must drop the sales flow entirely.
**Vague:** "just looking" → must keep the conversation alive.

Failure signals worth catching: a wrong price, a promised outcome, a confirmed appointment, advice it shouldn't give, the same message twice, an em dash, or a closing line implying it can do something it can't (*"want me to get you on the books?"* when there is no calendar).

---

## 8. Cost

Roughly **2 credits per model call, 2 calls per turn** (route + answer), so about **$0.03 per exchange** and **$0.17 per conversation** at the quality tier.

### Read this before starting a build

**The free tier is a ONE-TIME trial balance, not a monthly allowance.** On the Lumière build it was **$1.00 total**, which is about 35 exchanges or 6 full conversations. It does not refill.

**You cannot top it up.** There is no "add $10" button. The only way to continue once it's gone is upgrading to Pro at ~$60/mo. Budget accordingly *before* you start, because there is no cheap middle step.

**Building and testing consumes most of it.** Every test conversation costs the same as a real one. On Lumière, iterating and verifying ate the entire balance in a single session, and the last two changes shipped untested because the runtime cut out mid-test.

**When credits run out, the live widget dies completely.** Not degraded, dead. The bubble still appears and the badge still fires (those are website-side), but the chat fails on open, before the greeting, because the greeting is itself a model call. **Never demo in that state. A dead chat window is worse than no chat window.**

### How to not get caught out

1. **Do the heavy iteration in as few conversations as possible.** Give several answers in one message when testing (name, phone, email, time all at once) instead of one per turn. Each turn costs the same whether it carries one detail or four.
2. **Turn auto-open OFF while building.** Every page load starts a conversation and spends credits whether or not anyone types. Refreshing your own site while working on it silently drains the balance.
3. **Gate auto-open behind a query string for demos** (`?demo=1`), so your prospect link pops the chat open but ordinary traffic just sees the badge.
4. **Subscribe only for the months you're actively pitching, and cancel between.** It's a monthly plan, not a contract. Paying $60 to demo to nobody is burn; paying $60 the month you have meetings booked is trivially worth it against one client at $300–500/mo.
   - **Unused monthly credits DO NOT roll over.** They expire at the end of each billing cycle. Paying for 10,000 and using 200 means $60 gone and a fresh zero next month.
   - So **front-load the work into the billing month**. Do the building, testing and iterating while credits are live, because leftovers cannot be banked.
   - Annual plans release the whole year's credits up front, so a heavy month can draw on a quiet one. Only worth it under steady client load, and unused credits still expire worthless at year end.
5. **If a demo needs to stay live permanently at near-zero cost**, a custom widget on an Anthropic API key runs about a tenth of a cent per conversation with no monthly fee. Keep the Voiceflow build for what you actually deliver to clients.

### Other limits worth knowing early

- The **2-agent cap** on free means two clients before you're paying regardless.
- **White-label is Enterprise-only.** Do not promise a client an unbranded assistant until you've priced that.
- Price the client so one covers the plan comfortably. First client north of $300–500/mo.

---

## 9. Booking inside the chat

Sending someone to a booking form loses people. Booking in the conversation is the single most impressive thing to demo, and it's two Voiceflow Functions.

### The two functions

**`get_available_slots`** — no inputs, returns one string of upcoming open times.

```js
export default async function main(args) {
  const now = new Date();
  const dows = ['Sun','Mon','Tue','Wed','Thu','Fri','Sat'];
  const months = ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'];
  const times = ['9:00 am','10:30 am','12:00 pm','1:30 pm','3:00 pm','4:30 pm'];
  const out = [];
  let i = 1;
  while (out.length < 5 && i < 21) {
    const d = new Date(now.getTime() + i * 86400000);
    i++;
    const day = d.getDay();
    if (day === 0 || day === 1) continue;              // closed days
    const label = dows[day] + ', ' + months[d.getMonth()] + ' ' + d.getDate();
    out.push(label + ' at ' + times[(d.getDate() + out.length) % times.length]);
  }
  return { outputVars: { available_slots: out.join(' | ') }, next: { path: 'success' } };
}
```

**`book_appointment`** — inputs `full_name`, `phone`, `email`, `slot`, `service`. Validates, then returns `confirmation_code`, `booking_status`, and `missing_fields`.

**Put validation in the function, not the prompt.** Reject a phone under 10 digits and a malformed email, and return *which* field failed so the agent re-asks for only that one instead of restarting the booking. Prompts forget; code doesn't.

### Mirror the client's existing rules

If the site already has a booking widget, read its code and copy the rules exactly: which days are closed, which time slots exist. On the Lumière build the modal used a specific closed-days rule and six fixed times, and matching them meant chat and modal could never disagree. Nothing looks worse in a demo than the bot offering a slot the site says doesn't exist.

### Collecting the details

Four things: **full name, phone, email, chosen slot.** One ask per message. Give a reason once ("that's where the reminder goes"), never every time. If they volunteer several at once, take them all and skip ahead.

### Setup order

`voiceflow_function.create_batch` → `create_variable` for each input/output (the **batch** variable call failed with a 500 on this build; create them one at a time) → `create_paths_batch` for success/error → `voiceflow_tool.create_batch` to attach to the booking playbook → compile → test.

### Flip the guardrail, don't delete it

Before booking existed, the rule was "never confirm an appointment." Afterwards it becomes:

```
- NEVER state a time that did not come from the availability tool.
- Only confirm after the booking tool returns a confirmation code. Read it back
  exactly. Never invent one.
```

### Tone slips at the close

Confirming a booking is where a warm bot turns giddy. Ban exclamation marks explicitly at confirmation. *"You're all set, Rachel."* sounds like a studio that books people all day. *"You're all set, Rachel!"* sounds like software that just succeeded at something.

---

## 10. Wiring a real calendar

Everything above books into thin air, which is right for a prototype and wrong for a paying client. The upgrade is a normal integration and a legitimate separate line item.

Swap the body of `book_appointment` for a real API call. Voiceflow Functions run sandboxed JS with `fetch`, and there are saved API tools for simpler endpoints. **The conversation logic does not change at all** — only the function body.

**Scope by the client's platform, and always ask which one they use before quoting:**

| Tier | Platforms | Reality |
|---|---|---|
| Straightforward | Cal.com, Acuity, Square Appointments | Open APIs, self-serve keys. Half a day. |
| Workable | Vagaro, Mindbody | Public API but needs a developer account and API access enabled on an eligible account. Expect approval delays. |
| Gated | Boulevard, Zenoti, Aesthetic Record | What medspas actually run on. Partner-program territory. Scope as "we apply for access first." |

**Two things that will bite:**

**Double-booking.** A prototype invents availability from a rule. A real one must read live availability at the moment of asking, and ideally hold the slot while collecting details. Two people chatting at once will otherwise take the same time.

**HIPAA, for any medical client.** Live transcripts contain names, contact details, and "I want Botox for my forehead." That is protected health information sitting in Voiceflow's logs. You need a BAA (Enterprise tier) and a retention policy. **Do not put a live bot on a medical practice's site without sorting this out.** It's the only part of this stack carrying real legal exposure for you rather than the client. Non-medical clients don't have this constraint.

**Sell it in two phases:** the qualifying-and-booking bot as the core build, the live calendar hookup priced separately. The prospect can say yes to something smaller, and the prototype already proves it works.

---

## 11. What actually made this one good

Worth remembering when the next build feels merely fine:

1. The facts came out of the client's own page, so the bot could not contradict it.
2. The brand's real philosophy went into the prompt — *"she'll tell you what you don't need"* — so the bot sold the way the business sells.
3. Replies are short. Three sentences, one question, always a next step.
4. The funnel is explicit, including the branches where the honest answer is "that doesn't fit your budget."
5. It refuses things. A bot that declines to give medical advice reads more competent, not less.
6. Every AI tell was hunted down, starting with the em dash.
7. It finishes the job. Booking in the chat, rather than pointing at a button, is what turns a demo into a sale.
8. Every bug was found by reading traces. All of them produced replies that looked fine.
