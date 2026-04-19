# Seven Leverage Options — Beyond Surface UX (2026-04-19)

**Context**: R1 is substantively shipped (dark mode, unified library, sharing analytics, hub metrics). R2 P1 (voice intelligence) + R2 P2 D1 (mastery ribbon) + R2 P2 D5 (interview hub in library) are done. This doc steps back to ask: **what would materially move the product beyond surface polish?**

Each option is scored on four axes:

- **Impact** — How much does this change the product for real users?
- **Reach** — What share of the user base touches it?
- **Effort** — Weeks of work across repos.
- **Risk** — Likelihood of regression / complexity pitfalls.

Scale: ● (low) → ●●●●● (high).

---

## 1. Time-to-first-audio: streaming delivery

**Problem**: First-time users wait 53s for an episode to generate before they can hit play. 53s is cold for the "magic moment" — any curious visitor who doesn't convert in that window is a churn risk. The product's wow is audio; delay between prompt and audio is the core acquisition friction.

**Proposal**: Progressive audio delivery. Generate the first 30s of audio while the rest compiles, stream it to a pre-armed player the moment the first chunk lands. Target: audio playback begins within **8-12s** of prompt submission.

**How**: The streaming generator already splits into segments (the SSE pipeline). The missing piece is: TTS first segment eagerly, send audio URL over SSE, frontend pre-loads into player, auto-play when ready. Requires backend to upload segment audio to Cloud Storage as it finishes (not batched at end), and frontend to concatenate segments client-side or use HLS.

**Score**: Impact ●●●●●, Reach ●●●●●, Effort ●●●●, Risk ●●●

---

## 2. RSS podcast distribution for solo podcasters (R3 D1)

**Problem**: Solo creators generate an audio series on KitesForU but can't send subscribers to "the podcast." Their only share option is a `/courses/shared/[token]` link — fine for one-off views, but not discoverable and doesn't plug into listeners' podcast apps.

**Proposal**: Per-user RSS feed at `kitesforu.com/p/[username]/rss.xml`, validates against Apple Podcasts + Spotify. One episode per Course. Creator opens a "Distribute" panel (already shipped in `ShareContentModal`), gets ingestion instructions for Apple/Spotify.

**How**: Backend generates RSS via a simple Jinja template from existing Course + Episode Firestore data. Needs a new route, audio file stable URL, and show-artwork upload. Requires Cloud Storage public-read ACLs on episode audio.

**Score**: Impact ●●●●, Reach ●●● (solo creators / Quick Creator persona), Effort ●●●, Risk ●●

---

## 3. Real-time voice mock interview (R2 P2 D3)

**Problem**: `/interview-prep/mock/voice-preview` is explicitly a scripted setTimeout tour. Users expecting a real interview with an AI interviewer get a demo. This is the obvious differentiator for the interview-prep vertical — competitors (Interviewing.io, Pramp) lack the personalization layer, but right now KitesForU lacks the live voice layer.

**Proposal**: Wire LiveKit (STT + TTS) to the existing mastery engine. User speaks, AI interviewer listens, asks the next question, gives inline feedback mid-session. Mastery ribbon already exists; the hidden state machine already works from text. Voice is the I/O.

**How**: LiveKit Cloud (managed) for transport; Deepgram/AssemblyAI for STT; ElevenLabs/Inworld for TTS (already wired for podcasts). Turn-taking logic + barge-in handling on the client. Session cost: ~$0.10-0.30 per 15-min mock at current rates.

**Score**: Impact ●●●●●, Reach ●●● (interview-prep segment), Effort ●●●●●, Risk ●●●●

---

## 4. Quality auto-regeneration loop

**Problem**: The post-generation validator (`common/quality_gate.py` on workers) already detects low-emotion scripts, AI-tell words, and speaker-imbalance issues. Today it **logs** them. The user still gets the low-quality output.

**Proposal**: When the validator flags critical issues (emotion variety below threshold, >2 AI-tell words, speaker-balance skewed >70/30), transparently re-generate with a stronger prompt before the episode ships. Budget: 1 retry per episode; if second attempt still fails, ship with a quality note.

**How**: Already-built validator reports → conditional re-trigger of script generator with augmented system prompt ("previous attempt was flagged for X, address specifically"). Cost increment: ~10% extra LLM calls since only ~15% of episodes trigger regeneration.

**Score**: Impact ●●●●, Reach ●●●●●, Effort ●●, Risk ●●

---

## 5. Series memory ("Continue [Series Name]")

**Problem**: A user creates a 3-episode horror series about vampires. They come back next week. Nothing remembers the vampire cast, the world rules, or where the arc ended. They have to recreate the whole mental model or start from scratch.

**Proposal**: When a Course exists with >=2 episodes in a fiction genre, surface a "Continue [Series]" CTA on the home dashboard. Backend persists character names + world rules + arc-so-far in a lightweight memory doc. Next-episode generation pulls that doc into the system prompt so characters + world stay coherent.

**How**: Memory doc is a Firestore doc per Course keyed by `series_memory`. Writer: post-episode hook that extracts characters + events via a single LLM call. Reader: system-prompt injection in script generator. Frontend: new "Continue" card on `HomeDashboard`.

**Score**: Impact ●●●●, Reach ●●● (returning creative users), Effort ●●●, Risk ●●

---

## 6. Creator-side share analytics loop

**Problem**: `shared_content_view` + `shared_content_play` events fire today (R3 D2). Nothing surfaces to the creator. The feedback loop is broken — creators can't see their links working.

**Proposal**: On the `/library` card for any content with a share link, show a badge: "Viewed 12 times · Played 4 times this week". On the course detail page, a "Share performance" mini-card with a 7-day sparkline.

**How**: Backend aggregates the already-captured events into a per-content summary (daily roll-ups, served via a cheap GET). Frontend reads the summary, renders the badge + sparkline component (same pattern as the new `ScoreTrendSparkline`).

**Score**: Impact ●●●, Reach ●●●● (everyone who shares), Effort ●●, Risk ●

---

## 7. Creation quality tier surfacing

**Problem**: Standard ($0.08/ep, Inworld default) vs Premium ($4.50/ep, ElevenLabs) — the tier toggle is buried in "Advanced options" on `/create-smart`. Users default to Standard and don't hear the difference. The monetization path is hidden.

**Proposal**: First-class tier selector in the plan-review screen with A/B audio samples. "Hear the difference" — play the same 10-sec line in Standard vs Premium. Credits/tier consumption explicit.

**How**: Backend already has the tier plumbing. Needs: 2 pre-generated sample audio clips (one Standard, one Premium, same line); UI toggle in `PlanSection` that updates `plan.settings.tier`; clear price or credit badge.

**Score**: Impact ●●● (on monetization), Reach ●●●● (all creators), Effort ●●, Risk ●●

---

## Scoring summary

| # | Option | Impact | Reach | Effort | Risk | Notes |
|---|---|---|---|---|---|---|
| 1 | Streaming audio delivery | ●●●●● | ●●●●● | ●●●● | ●●● | Magic-moment win, acquisition |
| 2 | RSS podcast distribution | ●●●● | ●●● | ●●● | ●● | Solo creator monetization |
| 3 | Real-time voice mock | ●●●●● | ●●● | ●●●●● | ●●●● | Interview-prep moat |
| 4 | Quality auto-regeneration | ●●●● | ●●●●● | ●● | ●● | Already-built validator, low effort |
| 5 | Series memory | ●●●● | ●●● | ●●● | ●● | Retention for fiction creators |
| 6 | Share analytics loop | ●●● | ●●●● | ●● | ● | Closes an existing open loop |
| 7 | Tier surfacing | ●●● | ●●●● | ●● | ●● | Revenue, not scope |

## Recommended concrete next move

**Pick: #4 Quality auto-regeneration.**

Reasoning:
- **Highest Reach × lowest Effort × lowest Risk** in the table.
- Infrastructure already exists: validator already runs, logs results, and classifies issues. All that's missing is the conditional re-trigger.
- Touches **every episode** produced on the platform — impact compounds over time.
- Unblocked: does not need LiveKit, RSS infra, or frontend design debate. Workers-only change.
- Demonstrable quality lift without the user needing to change any behavior. Pure backend win.

**Runner-up**: #6 Share analytics loop (low-effort frontend + thin backend). Ship after #4.

**Deeper research warranted**: #1 (streaming audio) and #3 (real-time voice mock) both warrant dedicated proposal docs before committing — they're 4-week+ investments with architectural implications.

## What changes for users on each

1. **Streaming audio**: "I just asked for a podcast about X and it started playing in 10 seconds" — possibly the single most shareable moment in the product.
2. **RSS**: "I can put my KitesForU series on Apple Podcasts."
3. **Real-time voice**: "I just had a real conversation with an AI interviewer about my resume."
4. **Auto-regeneration**: "The audio I get from KitesForU sounds noticeably better than last month" — invisible to the user but visible in retention curves.
5. **Series memory**: "Episode 4 remembers the characters from episode 1."
6. **Share analytics**: "I can see 12 people played my episode this week."
7. **Tier surfacing**: "Oh, I can get Premium voice quality" — creators discover the upgrade path.

---

**Next action**: propose #4 in its own spec doc (`proposed/R2-quality-auto-regeneration.md`) with acceptance criteria and a 1-week implementation plan; simultaneously scope research doc for #1 and #3.
