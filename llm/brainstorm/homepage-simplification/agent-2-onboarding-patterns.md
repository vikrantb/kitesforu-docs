# Agent 2 — Onboarding Simplification Patterns

Pattern library for simplifying the first-time-user experience on creator/consumer hybrid products. Every pattern is grounded in at least one named real product. Where a claim depends on internal data I cannot verify, it is called out explicitly.

---

## Pattern A: Single input, everything else deferred

**Precedents**
- Perplexity (`perplexity.ai`) — a centered search bar is effectively the entire signed-out homepage. Model-picker, focus-scope, and attach-file sit on the input itself; the rest of the surface is near-empty.
- Google (`google.com`) — the canonical instance. Logo, one input, two buttons.
- DuckDuckGo (`duckduckgo.com`) — same shape, with a small value-prop line ("Privacy, simplified").
- v0.dev (Vercel) — the signed-out homepage is "describe your UI" as a single prompt box with a few example chips underneath.
- Granola (pre-expansion) — historical landing used a single "start a meeting" primitive.

**When it works**
- The product's value collapses cleanly into one verb ("search", "ask", "generate"). The user already knows the input shape (natural language, URL, query).
- The output is self-explanatory after one submission — the result page teaches the product.
- Pricing and account creation can be deferred to after first value.
- The competitor set has already trained users on what a search bar means.

**When it fails**
- Users don't know what to type. Empty-input anxiety dominates — Midjourney's original Discord-only era punished first-timers who didn't know prompt grammar.
- The product has multiple output modalities and the input alone can't disambiguate them (e.g. should "Marcus Aurelius" produce a podcast, a course, a writeup, or a Car Mode session?). Without a content-type signal, a single input forces an ambiguous default.
- The product needs a persona signal (creator vs. consumer) before it knows what to show next.
- Brand/trust building is critical pre-submission (health, finance, B2B enterprise).

**Fit for KitesForU**
KitesForU's first verb is close to "generate a lesson / episode / course about X" — that is a legitimate single-input collapse. But the product spans 5+ output modalities (podcast, course, writeup, class, Car Mode), and a bare input cannot convey that range. A single input works only if accompanied by either (a) lightweight modality chips near the input, or (b) an upfront modality-inference step on the backend. Without that, new visitors who type "stoicism" cannot predict what they will get back.

---

## Pattern B: Single hero CTA + one proof section

**Precedents**
- Linear (`linear.app`) — signed-out homepage is a single "Get started" CTA over a crisp screenshot / loop. One proof section ("Powering the world's best product teams" with logos) sits below the fold.
- Cal.com (`cal.com`) — hero is "Get started" + a short email capture. Proof is "Trusted by..." logos.
- Stripe (historical and current) — hero is an API code sample next to a single primary CTA; proof below.
- Vercel (`vercel.com`) — single "Start deploying" CTA with a secondary "Get a demo".

**When it works**
- The product has a clear primary user (developer, ops lead, PM) and the homepage is a sales surface, not a consumption surface.
- The visitor is evaluating whether to *try*, not whether to *consume content now*.
- Proof (logos, testimonials, numbers) is a material trust unlock — B2B and paid-tier consumer products.

**When it fails**
- Media / content products where the visitor expects to *consume immediately* (YouTube, Netflix, Spotify all reject this — they show content).
- When trust isn't the bottleneck; the bottleneck is "what is this thing?" Linear-style homepages assume the visitor already knows the category.

**Fit for KitesForU**
Works as a secondary layer, not the whole homepage. KitesForU's signed-out visitor will include many people who don't yet know the category ("AI-generated learning audio") exists as a thing. A single CTA + proof block risks feeling like a B2B SaaS landing for a product that is actually a consumer content experience.

---

## Pattern C: Persona branching at the top

**Precedents**
- Notion (`notion.so/templates`, `notion.so/product`) — the product surface funnels visitors into "Teams", "Personal", "Students", "Startups" lanes.
- Linear's `/customers` and `/method` — role-based reading paths (founders, PMs, engineers).
- Cursor's homepage (`cursor.com`) surfaces "for individuals" vs "for teams" pricing early, implicitly branching.
- Webflow and Framer both have "For designers / For marketers / For developers" toggle strips.
- Substack (`substack.com`) hero historically toggled between "Start writing" (creators) and "Discover writers" (readers) as dual CTAs.

**When it works**
- The two personas need *materially different* first actions (write vs. read, build vs. buy, host vs. attend). If one lane leads to "Sign up" and the other also leads to "Sign up", the branch is fake.
- The personas self-identify easily ("I'm a writer" vs. "I'm a reader"). No one hesitates.
- The product can afford to *lose* some of the smaller persona — branching taxes attention.

**When it fails**
- When the split is contrived. If 80%+ of visitors are on one lane, forcing the choice hurts the 80% to serve the 20%.
- When the personas overlap heavily ("creator-consumers" — most TikTok / Substack / YouTube users are both). Asking them to pick one is a false dichotomy; the product should let them slide between modes.
- When the first meaningful action is identical for both lanes (e.g. both need an account). The branch is then theater.

**Fit for KitesForU**
The creator/consumer duality is real, but the split is likely asymmetric — most first-time visitors will start as consumers (listen to something) before they create. A hard branch risks making the consumer lane feel like an afterthought. A soft branch (primary CTA = consume, tertiary link = "I want to create") tends to win; see Substack's evolution away from dual-CTA.

---

## Pattern D: Show-don't-tell (live content grid)

**Precedents**
- YouTube (`youtube.com` signed-out) — a live grid of trending videos. Zero explanation text. The product teaches itself.
- TikTok (`tiktok.com`) — signed-out homepage autoplays a For-You-style feed.
- Spotify (`spotify.com`) — the logged-out home shows live playlists / podcasts in a grid with thumbnails.
- Pinterest (`pinterest.com`) — signed-out page is an image wall that auto-teaches the product shape.
- Midjourney (`midjourney.com/showcase`) — the "Explore" / showcase surface is the marketing site for new visitors; it sells by example.
- Sora (OpenAI Sora page) and Pika (`pika.art`) — both rely heavily on a sample-output grid to convey what the tool does. Exact layout evolves; I have not re-checked in the last week, so treat as "observed pattern" not "current state".
- Lex.page — documents gallery / example Lex-written pieces serve the same role.

**When it works**
- The product's output is visually or aurally compelling in a thumbnail / snippet. If the output needs explanation, show-don't-tell fails.
- Discovery is itself the value prop. "What's on?" is the right question.
- The cold-start problem is solved: there is enough high-quality public content to populate the grid without human curation on every visit.

**When it fails**
- Empty grids. A show-don't-tell homepage with sparse content signals a dead product.
- Content the visitor can't *immediately* sample without an account (breaks the promise).
- B2B / productivity tools where the "output" is a private document — you can't show a customer's Notion page.

**Fit for KitesForU**
Strong fit if there is a library of public episodes / courses / writeups that can be sampled in <5 seconds. Audio is harder than video for show-don't-tell (can't scan visually), so thumbnails + 10-second preview clips + topic labels become critical. The pattern also answers "what is this" without copy — a visitor who sees "Stoicism Explained: 12 min episode" instantly understands format.

---

## Pattern E: Conversational entry

**Precedents**
- ChatGPT (`chat.openai.com`) — the signed-out landing is essentially the chat interface itself, with prompt suggestions as chips.
- Claude (`claude.ai`) — same shape: a single textarea that *is* the product.
- Anthropic's `claude.ai` new-conversation screen uses starter chips ("Write", "Learn", "Code", "Life stuff") to reduce empty-input anxiety.
- Character.ai (`character.ai`) — signed-out home shows a character grid and a "Talk to..." input.
- Inflection's Pi (historical) landed directly into a chat with a pre-seeded greeting.

**When it works**
- The product *is* a conversation. The homepage-as-product collapse is honest.
- Starter chips carry the cold-start problem (they teach what's possible without a wall of copy).
- First response must be fast and high-quality — a slow or generic first reply kills conversion.

**When it fails**
- When the product is not conversational and the chat is a gimmick wrapper over a form.
- When the "reply" takes >3 seconds or costs money to produce — first-time visitors will bounce before they see the payoff.
- When the conversation can't gracefully route to the product's other modalities (e.g. "I want to make a podcast" in a chat that can only answer questions).

**Fit for KitesForU**
Partial fit. A conversational entry ("What do you want to learn today?") could front-door the whole platform and route to the right modality on the backend. Risk: if the reply is a generated episode that takes 60+ seconds to produce, the first-5-seconds promise breaks. Works only if the chat returns *something* (a plan, an outline, a library match) inside 2-3 seconds, with the heavy generation backgrounded.

---

## Pattern F: Feed-first (you are the consumer, creation is elsewhere)

**Precedents**
- TikTok — signed-out page is the feed. Creation UI is entirely inside the app post-signup.
- Instagram (`instagram.com`) signed-out — explore grid. Creation is post-signup.
- YouTube — same. "Create" is a secondary-nav item behind auth.
- Medium (`medium.com`) — the signed-out home is a feed of curated posts. "Write" is a header link, not a hero.
- Substack Reader — the Substack homepage leans on "Discover writers" as the primary surface; the "Start writing" CTA is present but secondary on most surfaces.

**When it works**
- The creator/consumer ratio is heavily skewed toward consumers (classic 1:9:90 or 1:99).
- Consumption is the retention loop; creation is a separate, smaller funnel.
- The product can survive without surfacing creation on the homepage at all (creators find the "New" button after signup).

**When it fails**
- When the product's growth depends on creator acquisition and creators need to be convinced on the homepage (YouTube arguably lost this lane to TikTok by hiding creation).
- When there is no feed-worthy content yet (cold start).

**Fit for KitesForU**
Strong default candidate. If KitesForU's 1:99 ratio is real, the homepage's job is to convert consumers; creators can be served by a secondary surface (`/create`, `/studio`, or a "Made with KitesForU" link in the nav). This pattern also sidesteps the persona-branching tax in Pattern C.

---

## Pattern G: Empty-state-as-onboarding (in-product, not homepage)

**Precedents**
- Linear's first-run — a pre-seeded example project teaches the product through its own empty state.
- Notion's first-run — a template gallery modal appears on account creation.
- Figma's first-run — an interactive tutorial file ("Figma Basics") is auto-added to a new account.
- Loom — first-run flow records a sample video and immediately shows playback + sharing.
- Descript — new-account flow imports a sample audio file and opens it in the editor.

**When it works**
- The public homepage is separable from the onboarding experience. The homepage sells; the first-run teaches.
- The first-run can be rich and interactive without hurting marketing-page conversion.
- The product has a clear "aha" moment that a template / sample can deliver.

**When it fails**
- When the homepage and first-run are conflated into one surface (common mistake in "single input" products where the signed-in and signed-out states look identical and neither teaches).
- When the template library is huge and unfiltered — choice paralysis re-emerges.

**Fit for KitesForU**
This is a homepage-adjacent pattern. It argues: don't overload the homepage with teaching; move that load into post-signup first-run. The homepage can then commit to a simpler primary job (e.g. Pattern D or F), and the first-run can carry the "here's how to make your first episode" load.

---

## Pattern H: Examples-as-navigation (template / gallery homepage)

**Precedents**
- Midjourney's Explore / Showcase — the gallery *is* the marketing.
- Replit Templates (`replit.com/templates`) — starting points double as feature tour.
- Bolt.new and v0.dev both lean heavily on "Start from an example" chips under the input.
- Notion Templates (`notion.com/templates`) — a huge template library serves as both marketing and onboarding.
- Lovable / Create.xyz / similar AI-app builders — all anchor their homepage on a strip of "built with X" examples.

**When it works**
- The product's range is wide and hard to describe in copy; examples are faster than words.
- Each example is itself a starting point (click → begin your own version). This collapses marketing + onboarding into one click.
- The gallery is curated, fresh, and diverse enough to hit multiple visitor intents.

**When it fails**
- Stale galleries. A gallery is a commitment to ongoing curation.
- Galleries that link to results but don't let the visitor *start from* an example — the marketing-to-product seam leaks.

**Fit for KitesForU**
Natural fit alongside Pattern D. A row of "Start from this episode" or "Continue this course" tiles compresses "here's what we make" + "here's where you start" into a single surface. The consumer lane gets sample content; the creator lane gets sample prompts ("Make something like this").

---

## Cross-pattern insights

**What the winners share**
- **Deferred feature breadth.** Every winner pushes feature lists, pricing tables, and secondary modalities to second-tier pages. The homepage commits to one job.
- **A single above-the-fold action.** Perplexity (submit query), YouTube (play a video), Linear (Get started), Midjourney (view gallery). There is always exactly one thing the visitor is invited to do first.
- **Honest collapse.** The homepage's primary surface is the product's primary verb. Search products show a search bar. Feed products show a feed. Tool products show a tool. Winners don't market their product — they *are* their product, on the homepage.
- **Proof is secondary, never primary.** Logos and testimonials exist but sit below the first screen. The first screen is action, not reassurance.
- **Low-cost first interaction.** The first meaningful action is free, fast, and doesn't require an account (Perplexity's first query, TikTok's first scroll, YouTube's first play).

**What the losers share**
- Three or more competing CTAs above the fold ("Sign up", "Book a demo", "Watch video", "Read docs").
- Hero copy that describes the product instead of letting the product describe itself ("The all-in-one platform for...").
- Persona branches without materially different destinations.
- Feature grids as hero — six tiles of "AI-powered", "Real-time", "Collaborative" with no concrete artifact.
- Autoplay video heroes that explain instead of demonstrate.

**Timing heuristic — how long until a new visitor should have taken their first meaningful action**
- 5 seconds: the visitor should have *understood* the product's primary verb. Not signed up — just understood.
- 15 seconds: the visitor should have *seen* a concrete output (a sample, a search result, a feed card, an example).
- 30 seconds: the visitor should be able to trigger their *own* first action (type a query, play a sample, click an example) without creating an account.
- 60 seconds: the visitor should have received a *payoff* from that first action. If payoff requires signup, the signup gate must sit exactly at the moment of value-expectation, not before.

(These numbers are heuristics informed by general UX literature and observed product patterns, not a cited study.)

---

## First-time-visitor reading-order principles

**The 5-second test — what a visitor should be able to answer in 5 seconds**
- "What is this?" (category)
- "Who is it for?" (me or not me)
- "What do I do next?" (one obvious action)

If any of these three answers requires scrolling, reading a paragraph, or watching a video, the homepage has failed the 5-second test. Winners answer all three with a single glance at the above-the-fold surface.

**The single-action rule**
Every simplification winner has exactly one above-the-fold action. Not one *primary* action with two secondaries — one action, period. Secondary actions live in the header nav or below the fold. The mental model: the homepage is a funnel, not a menu. Multiple CTAs create a *decision*, and decisions cost attention. Perplexity doesn't have a "Watch demo" button next to the search bar. Google doesn't either.

**The "show one thing beautifully" heuristic**
When in doubt, pick the single most beautiful / compelling artifact the product can produce and put *only that* above the fold.
- Midjourney: one stunning image wall.
- Linear: one crisp product screenshot with a loop.
- Apple: one product photograph.
- Stripe: one code sample.
- YouTube: one grid of thumbnails.

The homepage is not a catalogue. It is a poster. A poster shows one thing, not ten. If the product's best artifact is a generated episode, put one episode (with a cover, a title, a length, and a play button) above the fold and let everything else wait.

**Reading-order for hybrid creator/consumer products specifically**
1. Show what consumers get (artifact, feed, gallery) — this is 80%+ of first visitors.
2. Let them sample without an account — fastest possible payoff.
3. Surface the creator lane as a *second* entry point, not a co-equal branch. "Make your own" as a small, persistent affordance beats a 50/50 hero split.
4. Defer all feature lists, pricing, and modality catalogues to dedicated pages.

---

## Anti-patterns to avoid

- **3+ CTAs above the fold.** Every CTA past the first dilutes the first. "Get started / Watch demo / Book a call / Read docs" is the classic confused homepage.
- **Persona selection without a reason to pick one.** If both lanes lead to the same signup flow, the persona split is theater and should be deleted.
- **Feature lists as hero.** Six tiles of "AI-powered / Real-time / Collaborative / Multi-platform / Secure / Fast" communicate nothing concrete. Replace with one artifact.
- **Autoplay video heroes that explain the product.** A video that *demonstrates* the product is fine (Linear's loops). A video that *describes* the product is a confession that the product can't describe itself.
- **Copy that describes, product that hides.** "The all-in-one platform for X" above a screenshot-free hero is a structural failure. Show the product.
- **Empty-state homepages on content products.** A show-don't-tell homepage with a sparse grid is worse than no grid.
- **Signup gates before first payoff.** If the visitor must create an account to understand the product, conversion dies. The first payoff must be free and anonymous whenever physically possible.
- **Modality catalogues as hero.** A hybrid-content product's homepage should not enumerate all its content types as equal-weighted tiles. Pick one, feature it, let the others exist on secondary surfaces.
- **Mismatched promise-to-payoff time.** Promising "instant" and delivering a 60-second generation on the first action. If payoff is slow, the homepage copy must set that expectation *before* the visitor commits (e.g. "We'll generate your first episode — takes ~90 seconds").
- **Two heroes stacked vertically.** Some sites have a hero, then another hero, then another hero. The visitor never reaches a coherent below-the-fold narrative because the "fold" keeps moving. Commit to one hero.
- **Live chat widgets as primary engagement.** The homepage is not a support surface for first-time visitors. A chat bubble competing with the primary CTA cannibalizes attention.

---

## Notes on confidence and freshness

- Patterns A–H are grounded in publicly observable homepage layouts of the named products as of my general knowledge. Individual homepages iterate frequently — specific CTA copy or hero layout for any single product may have shifted within the last weeks.
- Pattern D examples for Sora and Pika in particular: the *pattern* (sample-grid marketing) is stable; exact current layout was not re-verified in this session.
- The timing heuristics (5s / 15s / 30s / 60s) are directional industry norms derived from UX literature, not a cited study from a single source.
- I deliberately did not prescribe a KitesForU choice. Pattern F (feed-first) and Pattern D (show-don't-tell) both look like strong candidates on the evidence above, but the synthesis stage owns that call.
