# Inworld TTS Voice Expression, Personalities, and Styling Research

**Research Date**: April 2, 2026
**Sources**: Inworld official docs, API reference, community forums, WaveSpeed/Runware/LiveKit integrations, internal KitesForU codebase analysis
**Confidence Level**: HIGH for expression system, HIGH for voice catalog (verified against multiple third-party integrations), MEDIUM for some content-type recommendations (based on documentation + inference, not blind A/B tests)

---

## 1. Complete Voice Catalog

Inworld TTS 1.5 ships with **65+ system voices** across **15 languages**. The actual count on third-party platforms (Runware) shows **73 allowed voice values**, suggesting some voices have been added since initial launch. Both TTS-1.5-Max and TTS-1.5-Mini share the same voice library.

### 1.1 English Voices (25)

| Voice ID | Gender | Description | Personality/Tone | Best For |
|----------|--------|-------------|-----------------|----------|
| Alex | Male | Energetic and expressive mid-range voice, mildly nasal | Energetic, expressive | News, sports, tech explainers |
| Ashley | Female | Warm, natural voice | Warm, natural, conversational | Education, onboarding, general podcasts |
| Blake | Male | Rich, intimate voice | Intimate, warm, romantic | Audiobooks, storytelling, romantic content |
| Carter | Male | Energetic, mature radio announcer-style | Authoritative, energetic, broadcast | Ads, announcements, news, radio-style content |
| Craig | Male | Older British male, refined and articulate | Refined, authoritative, British | Education, documentaries, formal narration |
| Deborah | Female | Gentle and elegant | Gentle, elegant, poised | Meditation, luxury brands, formal narration |
| Dennis | Male | Middle-aged, smooth, calm and friendly | Calm, friendly, trustworthy | Education, customer service, general podcasts |
| Edward | Male | Fast-talking, emphatic, streetwise | Streetwise, fast, emphatic | Comedy, urban content, fast-paced segments |
| Elizabeth | Female | Professional middle-aged woman | Professional, authoritative | Narrations, voiceovers, corporate, news |
| Hades | Male | Commanding and gruff | Commanding, gruff, omniscient | Gaming, fantasy narration, dramatic storytelling |
| Hana | Female | Bright storytelling voice | Bright, engaging, youthful | Storytelling, children's content, upbeat segments |
| Julia | Female | Quirky, high-pitched, playful energy | Quirky, playful, energetic | Comedy, entertainment, character roles |
| Luna | Female | Calm, relaxing | Calm, soothing, peaceful | Meditations, sleep stories, ASMR, relaxation |
| Olivia | Female | Friendly British warmth | Friendly, warm, British | Onboarding, education, conversational |
| Pixie | Female | High-pitched, childlike, squeaky | Childlike, playful, squeaky | Gaming (fairy/companion), children's content |
| Dominus | Male | Menacing robotic tone | Menacing, robotic, commanding | Sci-fi, gaming villains, dramatic content |
| Celeste | Female | (Inferred: elegant, storytelling) | Narrative, atmospheric | Storytelling, literary narration |
| Sebastian | Male | (Inferred: dramatic, narrative) | Dramatic, commanding | Epic storytelling, fantasy narration |
| Serena | Female | (Inferred: warm, literary) | Warm, intimate, reflective | Literary narration, emotional storytelling |
| Claire | Female | (Inferred: professional, clear) | Clear, professional | Corporate narration, instructions |
| Timothy | Male | (Inferred: clear, professional) | Professional, articulate | Business narration, explainers |

**Note**: The API documentation only shows 3 English voices in its example response (Alex, Ashley, Dennis). The full list is obtained from the Voices API endpoint at `/tts/v1/voices`. Voices marked "(Inferred)" have descriptions derived from third-party platform usage examples, not official Inworld docs. Additional English voices beyond the 21 listed here likely exist (total is 25 per WaveSpeed).

### 1.2 Chinese Voices (4)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| Yichen | Male | Calm, flat young adult | Narrations, professional content |
| Xiaoyin | Female | Youthful, gentle, sweet | Conversational, friendly content |
| Xinyi | Female | Neutral tone | Narrations, formal content |
| Jing | Female | Energetic, fast-paced | Dynamic content, ads, entertainment |

### 1.3 Japanese Voices (2)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| Asuka | Female | Friendly, young adult | Conversational, education |
| Satoshi | Male | Dramatic, expressive, energetic | Storytelling, entertainment, gaming |

### 1.4 Korean Voices (4)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| Hyunwoo | Male | Young adult | General purpose, conversational |
| Minji | Female | Energetic, friendly, young | Entertainment, education |
| Seojun | Male | Clear, deep, mature | News, professional, authoritative |
| Yoona | Female | Gentle, soothing | Meditation, calm content, narration |

### 1.5 French Voices (4)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| Alain | Male | Deep, smooth, middle-aged, composed, calm | Narration, storytelling, formal |
| Helene | Female | Middle-aged, smooth, musical, graceful | Education, cultural content |
| Mathieu | Male | Nasal quality | Character roles, casual content |
| Etienne | Male | Calm, young adult | Conversational, education |

### 1.6 German Voices (2)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| Johanna | Female | Calm, older, low, smoky | Narration, atmospheric content |
| Josef | Male | Articulate, announcer-like | News, professional, formal |

### 1.7 Spanish Voices (4)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| Diego | Male | Soothing, gentle | Calm content, storytelling |
| Lupita | Female | Vibrant, energetic, young | Entertainment, ads, upbeat |
| Miguel | Male | Calm, adult, storytelling | Storytelling, narration |
| Rafael | Male | Middle-aged, deep, composed | News, professional, authoritative |

### 1.8 Portuguese Voices (2)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| Heitor | Male | Composed, neutral tone | Narration, professional |
| Maite | Female | Middle-aged | General purpose, conversational |

### 1.9 Italian Voices (2)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| Gianni | Male | Deep, smooth, rapid speech | Entertainment, dynamic content |
| Orietta | Female | Calm, adult, soothing cadence | Narration, education |

### 1.10 Dutch Voices (4)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| Erik | Male | Older, weathered edge | Character roles, dramatic |
| Katrien | Female | Expressive | Entertainment, dynamic content |
| Lennart | Male | Confident, calm, relaxed | Professional, narration |
| Lore | Female | Clear, calm | Narrations, education |

### 1.11 Polish Voices (2)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| Szymon | Male | Warm, friendly | Conversational, education |
| Wojciech | Male | Middle-aged | Professional, narration |

### 1.12 Russian Voices (4)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| Svetlana | Female | Soft, high-pitched, breathy | ASMR, intimate content, storytelling |
| Elena | Female | Clear, mid-range, neutral, informational | News, professional, education |
| Dmitry | Male | Deep, gravelly, commanding, narrative | Dramatic storytelling, authority |
| Nikolai | Male | Deep, resonant, theatrical | Theatrical content, epic narration |

### 1.13 Hindi Voices (estimated 2-3)

| Voice ID | Gender | Description | Best For |
|----------|--------|-------------|----------|
| (Names not publicly documented) | M/F | Professional clarity | Education, news, general |

**Note**: WaveSpeed mentions "South Asian & Middle Eastern -- Hindi, Hebrew, Arabic -- 6 voices with professional clarity" but individual voice names for Hindi, Hebrew, and Arabic are not enumerated in any public documentation found. These must be retrieved via the `/tts/v1/voices?language=hi` API endpoint.

### 1.14 Hebrew Voices (estimated 1-2)

Not individually documented. Available via API.

### 1.15 Arabic Voices (estimated 1-2)

Not individually documented. Available via API.

---

## 2. Expression System Deep Dive

### 2.1 Audio Markup Tags (Complete Reference)

**CRITICAL**: All emotion, delivery, and non-verbal markups are currently **experimental** and **English-only**. Non-English languages do NOT support expression tags. This is the single biggest limitation.

#### 2.1.1 Emotion Tags (6)

Place at the BEGINNING of text. ONE per request. Affects entire segment.

| Tag | Effect | Use When |
|-----|--------|----------|
| `[happy]` | Cheerful, upbeat, positive delivery | Good news, excitement, celebrations, warm greetings |
| `[sad]` | Sorrowful, melancholy, subdued delivery | Bad news, empathy, disappointment, loss |
| `[angry]` | Frustrated, aggressive, forceful delivery | Strong disagreement, corrections, heated debate |
| `[surprised]` | Startled, astonished, wide-eyed delivery | Breaking news, unexpected revelations, amazement |
| `[fearful]` | Anxious, frightened, tense delivery | Warnings, suspense, concern, danger |
| `[disgusted]` | Revulsion, strong disapproval delivery | Moral outrage, rejection, strong negative reactions |

#### 2.1.2 Delivery Style Tags (2)

Place at the BEGINNING of text. ONE per request. Mutually exclusive with emotion tags.

| Tag | Effect | Use When |
|-----|--------|----------|
| `[laughing]` | Speech delivered with audible laughter | During or after jokes, genuine amusement |
| `[whispering]` | Hushed, breathy, quiet delivery | Secrets, quiet emphasis, intimate tone, ASMR |

#### 2.1.3 Non-Verbal Vocalization Tags (6)

Place INLINE where the sound should occur. Multiple allowed per request. Can combine with ONE emotion/delivery tag.

| Tag | Effect | Natural Placement |
|-----|--------|-------------------|
| `[breathe]` | Audible breath/pause | Before important statements, between thoughts |
| `[clear_throat]` | Throat clearing sound | Before corrections, important announcements, transitions |
| `[cough]` | Coughing sound | Sparingly for realism |
| `[laugh]` | Laughter sound (not laughing delivery) | After humor, when expressing amusement |
| `[sigh]` | Sighing sound | Resignation, relief, empathy, frustration |
| `[yawn]` | Yawning sound | Tiredness, boredom (use very sparingly) |

**Pro tip from Inworld docs**: If a non-verbal is consistently being omitted, repeat the markup (e.g., `[laugh] [laugh]`) for emphasis. Works best for naturally repeatable sounds.

#### 2.1.4 SSML Break Tags

Supported across ALL languages (not just English). Insert silence of specified duration.

```
<break time="500ms"/>    -- half-second pause
<break time="1s"/>       -- one-second pause
<break time="2s"/>       -- two-second pause
```

**Syntax**: Standard SSML `<break>` tags inline in text. Supports `ms` (milliseconds) and `s` (seconds) units.

**Example**:
```
One second pause <break time="1s"/> two seconds pause <break time="2s"/> this is the end.
```

**Note**: In KitesForU's capabilities.py, Inworld is configured with `pause=PauseMethod.POST_PROCESS` meaning our codebase currently handles pauses via silence insertion in the audio combiner rather than using Inworld's native SSML break support. This could be upgraded.

#### 2.1.5 Custom Pronunciation (IPA)

Inline IPA notation using forward slashes:

```
Your interests are a perfect match for a honeymoon in /kriit/.
```

**Syntax**: Replace the word with its IPA phoneme sequence wrapped in `/` delimiters.

**Common examples**:
- "Crete" -> `/kriit/`
- "Yosemite" -> `/joUsEmIti/`
- "Nguyen" -> `/NwIen/`
- "Acai" -> `/AisAi'ii/`

**Finding IPA**: Use online IPA dictionaries or phoneme converters. The IPA must be accurate for the target language.

#### 2.1.6 Conversational Naturalness Features

Inworld TTS responds well to natural speech patterns in the input text:

| Feature | Syntax | Effect |
|---------|--------|--------|
| Filler words | `uh`, `um`, `well`, `like`, `you know` | Creates natural thinking pauses |
| Emphasis | `*word*` (asterisks) | Emphasizes the word |
| Ellipsis | `...` | Creates a lingering pause/beat |
| Short sentences | Period-separated short phrases | Creates urgency and emphasis |
| Long sentences | Flowing compound sentences | Creates calm, measured delivery |
| Contractions | `don't`, `we'll`, `you're` | More natural conversational feel |

### 2.2 Voice Quality Parameters

#### 2.2.1 speaking_rate

| Value | Effect | Use Case |
|-------|--------|----------|
| 0.5 | Half speed -- very slow, dramatic | Dramatic pauses, meditation, emphasis |
| 0.7 | Slow -- deliberate, measured | Meditation, calm instruction, storytelling tension |
| 0.85 | Slightly slow -- thoughtful | Educational content, complex explanations |
| 0.9 | Just below normal -- calm | Calm narration, relaxation content |
| 1.0 (default) | Normal native speed | General purpose, conversation |
| 1.1 | Slightly fast -- energetic | Happy/excited content, upbeat segments |
| 1.15 | Fast -- high energy | Excited delivery, sports commentary |
| 1.2 | Rapid -- urgent | Breaking news, urgent updates |
| 1.5 | Maximum -- very rapid | Speed reading, rapid announcements (quality degrades) |

**KitesForU internal mapping** (from `shared.py`):
```
"excited": 1.15,  "calm": 0.90,     "sad": 0.85,
"tense": 1.10,    "urgent": 1.20,   "thoughtful": 0.90,
"melancholic": 0.85,  "neutral": 1.0
```

#### 2.2.2 temperature

Controls expressiveness/randomness of voice output.

| Value | Effect | Use Case |
|-------|--------|----------|
| 0.6 | Very consistent, predictable | IVR, phone systems, automated workflows |
| 0.7 | Consistent with slight variation | Professional narration requiring consistency |
| 0.8 | **Minimum recommended by Inworld** | Below risks broken/monotone generation |
| 0.9 | Moderate variation | General narration, education |
| 1.0 (default) | Natural variation | Conversational, general purpose |
| 1.1 | Expressive | Storytelling, character dialogue, entertainment |
| 1.12 | Highly expressive | Dramatic narration, emotional storytelling |
| 1.2 | Very expressive | Dynamic character dialogue, gaming |
| 1.3 | **Maximum safe** | Maximum expressiveness (above may produce artifacts) |

**KitesForU internal formula** (from `shared.py`):
```python
temperature = max(0.8, min(1.3, 0.8 + (intensity * 0.5)))
# intensity 0.0 -> temp 0.8 (min)
# intensity 0.5 -> temp 1.05 (moderate)
# intensity 1.0 -> temp 1.3 (max)
```

#### 2.2.3 Interaction: Tags + Parameters

The three expression channels work together:

```
Expression = Emotion Tag + speaking_rate + temperature
```

- **Emotion tags** set the emotional color (happy, sad, angry)
- **speaking_rate** controls pacing (slow for sad, fast for excited)
- **temperature** controls how much vocal variety/expressiveness
- **Non-verbal tags** add human-like sounds at specific points

**Key interaction rules**:
1. Emotion tags conflict with each other -- only ONE per segment
2. Non-verbal tags should match the emotion (no `[yawn]` with `[angry]`)
3. Higher temperature amplifies emotion tag effects
4. Lower speaking_rate + low temperature = monotone risk
5. Higher speaking_rate + high temperature = most dynamic (but least predictable)

### 2.3 Text Normalization

Inworld supports a `text_normalization` parameter:
- `"ON"` (default): Numbers, dates, abbreviations are expanded ("Dr." -> "Doctor")
- `"OFF"`: Text is read exactly as written

For podcast content, keep `ON` but still pre-format critical items (currency, dates, emails) as spoken words in the LLM prompt to avoid inconsistencies.

---

## 3. Content-Type Expression Profiles

### 3.1 News/Analysis

**Voice recommendations**: Carter (announcer energy), Elizabeth (professional authority), Craig (refined British authority)

**Voice pairing**: Carter (male anchor) + Elizabeth (female anchor)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| speaking_rate | 1.0 -- 1.1 | Professional broadcast pace |
| temperature | 0.9 -- 1.0 | Natural but consistent |
| Primary emotion | None (neutral) | News should be mostly objective |
| Breaking news | `[surprised]` + rate 1.15 | Convey urgency for breaking stories |
| Analysis segments | rate 0.95, temp 0.9 | Slower, more measured for analysis |
| Interviews | `[breathe]` between Q&A | Natural pauses between thoughts |

**Expression pattern**:
```
ANCHOR: This morning, three major tech companies announced... <break time="500ms"/>
[surprised] In a move nobody anticipated, the merger was valued at one hundred billion dollars.
[breathe] Industry analysts are calling this the largest technology deal of the decade.
```

**Tag frequency**: Minimal. 1 emotion tag per 5-8 segments. News should sound professional, not theatrical. Use `[surprised]` only for genuinely unexpected developments.

### 3.2 Educational/Explainer

**Voice recommendations**: Dennis (calm, friendly teacher), Ashley (warm, natural), Olivia (friendly British)

**Voice pairing**: Dennis (primary explainer) + Ashley (supporting examples/stories)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| speaking_rate | 0.9 -- 1.0 | Slightly slower for comprehension |
| temperature | 0.95 -- 1.05 | Natural with some enthusiasm |
| Complex topics | rate 0.85, temp 0.9 | Slow down for difficult concepts |
| Aha moments | `[happy]` + rate 1.05 | Convey discovery and wonder |
| Key definitions | rate 0.85, `*emphasis*` | Slow and stress important terms |
| Transitions | `[breathe]` or `<break time="800ms"/>` | Give listener processing time |

**Expression pattern**:
```
TEACHER: So, the key concept here is... <break time="500ms"/> photosynthesis.
[breathe] Think of it like this -- plants are basically tiny solar panels.
[happy] And here's the really cool part...
Once you understand this, everything else clicks into place.
```

**Curiosity/wonder**: Use `[happy]` sparingly at "aha moment" transitions. Combine with slightly elevated rate (1.05) and temperature (1.05). The excitement should feel genuine, not forced.

**Pacing strategy**:
- New concept: rate 0.85, pause after
- Example/analogy: rate 1.0 (normal conversational)
- Summary/recap: rate 0.95
- "Here's the big insight": `[happy]` + rate 1.05

### 3.3 Comedy/Entertainment

**Voice recommendations**: Edward (streetwise, fast), Julia (quirky, playful), Alex (energetic)

**Voice pairing**: Edward (setup/deadpan) + Julia (reactions/punchlines) OR Alex (straight man) + Julia (comic)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| speaking_rate | 0.9 -- 1.15 (varies) | Pacing is timing; vary for effect |
| temperature | 1.1 -- 1.2 | High expressiveness for comedy |
| Setup | rate 1.0, neutral | Straight delivery builds tension |
| Punchline | rate 0.9 + `<break time="300ms"/>` before | Slow slightly, pause for beat |
| Reaction | `[laughing]` or `[laugh]` | Only AFTER genuinely funny moments |
| Sarcasm | `[surprised]` + rate 0.9 | Slight disbelief tone |

**Expression pattern**:
```
HOST1: So I asked the waiter, what's the soup of the day?
HOST2: [laughing] Let me guess...
HOST1: He says... <break time="500ms"/> it's a surprise.
HOST2: [laugh] A surprise? In a restaurant?
HOST1: [happy] I know, right? And then -- get this -- [breathe] he brings out cereal.
HOST2: [surprised] Cereal?!
```

**Critical: `[laughing]` placement**:
- Too much = sounds fake and grating
- Maximum frequency: 1 `[laughing]` per 8-10 exchange turns
- Use `[laugh]` (non-verbal inline) more often than `[laughing]` (delivery style)
- `[laugh]` is a quick chuckle; `[laughing]` makes the ENTIRE segment sound like giggling speech
- Place `[laugh]` AFTER the funny thing, not before
- The listening host should `[laugh]` more often than the joke-teller

**Timing for punchlines**:
1. Slow down slightly (rate 0.9) approaching the punchline
2. Insert `<break time="300ms"/>` right before the key word
3. Deliver key word at normal rate
4. Follow with `[laugh]` from the other host
5. `<break time="500ms"/>` before moving on (let it land)

### 3.4 Storytelling/Narrative

**Voice recommendations**: Blake (rich, intimate -- audiobooks), Hades (commanding -- fantasy), Celeste/Serena (atmospheric)

**Voice pairing**: Blake (narrator) + Hades (villain/authority character) OR Serena (narrator) + any character voice

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| speaking_rate | 0.85 -- 1.1 (dynamic) | Varies with narrative pace |
| temperature | 1.08 -- 1.15 | High expressiveness for drama |
| Tension building | rate 0.85 -> 0.9, `[fearful]` | Slow, anxious delivery |
| Action scenes | rate 1.1 -- 1.15 | Fast, urgent pace |
| Reveals | `<break time="1s"/>` + `[surprised]` | Dramatic pause before reveal |
| Calm scenes | rate 0.9, temp 1.0, neutral | Measured, peaceful |
| Sad scenes | `[sad]` + rate 0.85 | Slow, melancholy delivery |

**Emotion arc example** (tension -> release):
```
NARRATOR: The hallway stretched endlessly before her. [breathe]
[fearful] Each step echoed against the cold stone walls.
She could hear it now... closer... <break time="1s"/>
[surprised] The door flew open, revealing -- [breathe]
[happy] her daughter, holding a birthday cake, grinning from ear to ear.
[laugh] Everyone was there. It was just a surprise party.
```

**Character voice differentiation** (same voice, different settings):
- Use `[whispering]` for mysterious/quiet characters
- Use `[angry]` sparingly for antagonist lines
- Use speaking_rate variation: slow for elderly/wise, fast for young/anxious
- Use temperature variation: low for stoic characters, high for emotional ones
- Keep non-verbal tags character-appropriate: villains `[clear_throat]`, nervous characters `[breathe]`

### 3.5 Meditation/Calm

**Voice recommendations**: Luna (calm, relaxing -- BUILT for this), Deborah (gentle, elegant), Yoona (Korean, gentle, soothing)

**Voice pairing**: Typically single-voice only. If paired: Luna (guide) + Deborah (ambient narration)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| speaking_rate | 0.8 -- 0.9 | Slow, deliberate, calming |
| temperature | 0.8 -- 0.9 | Very consistent, predictable |
| Breathing cues | `[breathe]` | Guide the listener's breathing |
| Pauses | `<break time="2s"/>` liberally | Give space for reflection |
| Emotion | None (neutral) or very occasional `[happy]` | Mostly neutral, serene |

**Expression pattern**:
```
GUIDE: Allow yourself to settle into this moment. <break time="2s"/>
[breathe] Take a deep breath in... <break time="3s"/> and slowly release.
<break time="2s"/>
Notice the weight of your body. <break time="1s"/> Feel the support beneath you.
<break time="2s"/>
There is nowhere else you need to be. <break time="1s"/> Nothing else you need to do.
```

**Rules**:
- NEVER use `[angry]`, `[surprised]`, `[fearful]`, `[disgusted]`
- NEVER use `[laughing]`, `[clear_throat]`, `[cough]`, `[yawn]`
- Only allowed tags: `[breathe]` (frequently), `<break>` (liberally)
- `[happy]` only at the very end for gentle positive closure
- Keep sentences short. Lots of periods. Lots of breaks.
- Never exceed rate 0.9. Ideal sweet spot: rate 0.85, temp 0.85

### 3.6 Interview/Debate

**Voice recommendations**: Carter (interviewer -- authoritative), Dennis (interviewer -- friendly), Elizabeth (moderator)

**Voice pairing**: Dennis (interviewer) + Ashley (interviewee -- warm) OR Carter (moderator) + Craig (guest expert)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| speaking_rate | 1.0 (varies per speaker) | Natural conversation pace |
| temperature | 1.0 -- 1.1 | Moderately expressive |
| Questions | rate 1.0, slight uptone via `?` | Natural questioning intonation |
| Active listening | `[breathe]`, `uh huh`, `right` | Show engagement |
| Strong disagreement | `[surprised]` + rate 1.05 | Not `[angry]` -- too aggressive |
| Emphasis | `*word*` for key terms | Highlight important points |

**Expression pattern**:
```
INTERVIEWER: [breathe] So, let me make sure I understand this correctly...
You're saying the entire market shifted in just three weeks?
GUEST: [surprised] Well, that's the thing -- nobody saw it coming.
Not the analysts, not the traders... <break time="500ms"/> nobody.
INTERVIEWER: [breathe] And what do you think was the catalyst?
GUEST: [happy] Ah, now *that* is the interesting question.
```

**Differentiating interviewer vs interviewee**:
- Interviewer: more `[breathe]` (active listening), more questions, rate ~1.0
- Interviewee: more emotion tags, more varied rate, higher temperature
- Moderator: most neutral, lowest temperature, controlled rate
- For debates: use `[surprised]` for counterpoints, never `[angry]` (sounds hostile)

**Natural interruption simulation**:
- End the interrupted speaker's segment mid-sentence with `--`
- Start the interrupter's segment with no pause
- Use `[breathe]` at the start of "cutting in"

---

## 4. Voice Pairing Recommendations for 2-Host Podcasts

### 4.1 Principles

1. **Contrast**: Pair voices with different pitches, paces, and timbres
2. **Complement**: Both should be listenable for extended periods
3. **Role clarity**: Listener should instantly know who is speaking
4. **Gender mix**: Male + Female is the most natural pairing for contrast (though M+M and F+F work with sufficient voice contrast)

### 4.2 Recommended Pairings by Content Type

| Content Type | Host 1 (Anchor/Lead) | Host 2 (Color/Support) | Chemistry Notes |
|-------------|----------------------|------------------------|-----------------|
| News/Analysis | Carter (M, announcer) | Elizabeth (F, professional) | Classic anchor desk pairing |
| Education | Dennis (M, calm teacher) | Ashley (F, warm natural) | Approachable, non-intimidating |
| Comedy | Alex (M, energetic) | Julia (F, quirky playful) | Energy contrast, good banter dynamics |
| Storytelling | Blake (M, rich intimate) | Hana (F, bright) | Dramatic contrast, narrator + character |
| Tech/Explainer | Craig (M, refined British) | Olivia (F, friendly British) | Cerebral, authoritative, British pairing |
| Interview | Dennis (M, calm) | varies by guest | Dennis is the ideal interviewer voice |
| Debate | Carter (M, authoritative) | Craig (M, refined) | Two authoritative voices for serious discourse |
| Casual/Chat | Ashley (F, warm) | Alex (M, energetic) | Natural, relatable, everyday conversation |
| True Crime | Elizabeth (F, professional) | Blake (M, intimate) | Serious tone with narrative depth |
| Fantasy | Hades (M, commanding) | Deborah (F, elegant) | High fantasy, epic scope |
| Wellness | Luna (F, calm) | Dennis (M, calm friendly) | Soothing but not monotone |

### 4.3 Pairings to Avoid

- Hades + Dominus: Both too dark/commanding, fatiguing to listen to
- Pixie + Julia: Both high-pitched, hard to distinguish
- Luna + Deborah: Both too calm/similar, loses energy
- Edward + Alex: Both fast/energetic, overwhelming

---

## 5. Fallback Behavior Analysis

### 5.1 Current KitesForU Fallback Chain

From `tts_health.py` and the orchestrator:
```
Inworld -> ElevenLabs -> OpenAI -> Google (circuit breaker aware)
```

### 5.2 Expression Features Lost Per Fallback

| Feature | Inworld | ElevenLabs | OpenAI | Google |
|---------|---------|------------|--------|--------|
| `[happy]` etc. emotion tags | YES | Own format* | NO | NO (SSML prosody) |
| `[laughing]`, `[whispering]` | YES | `(whispers)` prompt* | Via instructions | NO |
| `[breathe]`, `[sigh]` etc. | YES | NO | NO | NO |
| `<break time="Xs"/>` | YES | NO | NO | YES (SSML) |
| `/IPA/` pronunciation | YES | NO | NO | YES (SSML phoneme) |
| speaking_rate control | YES (0.5-1.5) | YES (stability) | YES (speed) | YES (SSML prosody) |
| temperature control | YES (0.6-1.3) | YES (style_exaggeration) | NO | NO |

*ElevenLabs v3 uses parenthetical directions: `(cheerfully) Hello!` and `(whispers) This is a secret`

### 5.3 Graceful Degradation Strategy

When Inworld fails and falls back to another provider, the `adapt_script_for_provider()` function in `capabilities.py` handles transformation:

**Inworld -> ElevenLabs fallback**:
- Strip `[happy]`, `[sad]`, etc. bracket tags
- Strip `[breathe]`, `[sigh]`, etc. non-verbal tags
- Convert emotion metadata to ElevenLabs' `style_exaggeration` parameter
- `<break>` tags -> post-process silence insertion
- IPA `/phoneme/` -> strip (ElevenLabs doesn't support inline IPA)

**Inworld -> OpenAI fallback**:
- Strip ALL bracket tags (emotion + non-verbal)
- Convert emotion to `speed` parameter only (limited mapping)
- `<break>` tags -> post-process silence insertion
- IPA -> strip

**Inworld -> Google fallback**:
- Strip bracket tags
- Convert emotion to SSML `<prosody>` tags
- `<break>` tags -> keep (Google supports SSML breaks)
- IPA -> convert to SSML `<phoneme>` tags (if pronunciation_control supported)

### 5.4 Detecting Jarring Fallback Transitions

When a podcast has some segments from Inworld and others from a fallback provider within the same episode:

**What sounds jarring**:
1. **Timbre mismatch**: Different voice ID between providers (e.g., Ashley on Inworld vs. "Rachel" on ElevenLabs)
2. **Expression loss**: Segments with rich emotion tags sound lively, fallback segments sound flat
3. **Pacing changes**: Inworld's speaking_rate calibration differs from other providers' speed params
4. **Audio quality**: Different sample rates, codecs, or audio characteristics

**Mitigation strategies**:
1. **Voice matching**: Maintain a cross-provider voice mapping table (same persona -> closest match on each provider)
2. **Expression normalization**: When fallback occurs, slightly increase the fallback provider's expressiveness settings to compensate for lost emotion tags
3. **Audio post-processing**: Normalize volume, apply consistent EQ across segments from different providers
4. **Full-episode fallback**: If >20% of segments fail on Inworld, regenerate the ENTIRE episode on the fallback provider for consistency (rather than mixing)
5. **Transition smoothing**: Insert slightly longer silence gaps between segments from different providers to mask timbre shifts

### 5.5 KitesForU-Specific Upgrade Opportunities

Based on this research, our current implementation in `shared.py` and `capabilities.py` could be improved:

1. **Enable SSML breaks natively**: Change Inworld's `pause` from `POST_PROCESS` to a new method that uses `<break>` tags directly, reducing audio combiner work
2. **Richer emotion mapping**: Current `EMOTION_TO_MARKUP` maps 26 emotions. Add content-type-aware temperature/rate presets
3. **Content-type expression profiles**: The `get_prompt_guidance()` function currently returns generic guidance. Make it content-type-specific per the profiles in Section 3
4. **Voice catalog integration**: Build a voice database with all 65+ voices, their characteristics, and per-content-type suitability scores

---

## 6. Key Constraints and Gotchas

### 6.1 English-Only Expression Tags

The most critical limitation: **All audio markup tags (emotion, delivery, non-verbal) are experimental and English-only.** For Hindi, Spanish, French, Japanese, etc., the ONLY expression controls are `speaking_rate` and `temperature`. No emotion tags, no non-verbal sounds.

This means for multilingual podcasts:
- English segments: Full expression (tags + rate + temperature)
- Non-English segments: Only rate + temperature for expression
- Cross-language parity requires relying on natural text phrasing to convey emotion

### 6.2 One Emotion Per Segment

You cannot change emotion mid-segment. Each API call (max 2,000 chars) gets exactly ONE emotion/delivery tag at the start. To switch emotions, split into separate requests.

### 6.3 Conflicting Tags

The model struggles when tags contradict the text content:
- BAD: `[angry] I appreciate your help and I'm really grateful`
- BAD: `[angry] ... [yawn]` (yawning doesn't occur alongside anger)
- GOOD: `[angry] I can't believe you did that`
- GOOD: `[sad] ... [sigh]` (sighing pairs naturally with sadness)

### 6.4 Temperature Floor

Never go below 0.8 temperature. Inworld docs explicitly warn this "risks broken/monotone generation." Our code enforces this: `max(0.8, min(1.3, ...))`.

### 6.5 Max 2,000 Characters Per Request

Hard API limit. Our code uses 1,900 as `MAX_CHUNK_SIZE` with a safety margin. Long segments must be chunked at sentence boundaries.

### 6.6 Hindi/Hebrew/Arabic Voice Names Unknown

These 6 voices exist but are not documented in any public source. Must be retrieved via API call to `/tts/v1/voices` filtered by language. This is a gap we should fill by making that API call and caching the results.

---

## 7. Recommended Next Steps

1. **Call the Voices API**: Hit `/tts/v1/voices` to get the complete catalog including Hindi/Hebrew/Arabic voice names and descriptions
2. **Build voice personality database**: Store all voice metadata (name, gender, language, description, tags, suitability scores per content type)
3. **Create content-type prompt templates**: Implement the expression profiles from Section 3 as provider-specific prompt guidance
4. **Voice pairing engine**: Build logic to recommend complementary host voice pairs based on content type
5. **Enable SSML breaks**: Upgrade Inworld integration to use native `<break>` tags instead of post-process silence
6. **Expression tag budget**: Implement a per-episode tag frequency limiter to prevent over-use (especially `[laughing]`)
7. **Cross-provider voice mapping**: Build a mapping table linking Inworld voices to their closest matches on ElevenLabs, OpenAI, and Google for seamless fallback

---

## Sources

- Inworld Audio Markups: https://docs.inworld.ai/tts/capabilities/audio-markups
- Inworld Generating Speech Best Practices: https://docs.inworld.ai/tts/best-practices/generating-speech
- Inworld Prompting for TTS: https://docs.inworld.ai/tts/best-practices/prompting-for-tts
- Inworld Custom Pronunciation: https://docs.inworld.ai/tts/capabilities/custom-pronunciation
- Inworld TTS Models: https://docs.inworld.ai/tts/tts-models
- Inworld List Voices API: https://docs.inworld.ai/api-reference/ttsAPI/texttospeech/list-voices
- Inworld Voice Profiles (STT): https://docs.inworld.ai/stt/voice-profiles
- WaveSpeed Voice Catalog: https://wavespeed.ai/models/inworld/inworld-1.5-max/text-to-speech
- LiveKit Inworld Plugin: https://docs.livekit.io/agents/models/tts/inworld/
- Runware Inworld TTS: https://runware.ai/docs/models/inworld-tts-1-5-max
- GetStream Inworld Integration: https://getstream.io/blog/inworld-tts-plugin/
- Voximplant Inworld Integration: https://voximplant.com/blog/inworld-text-to-speech-now-available-in-voximplant
- Inworld Community Forums: https://community.inworld.ai/t/need-guidance-on-emotional-markups/60
- KitesForU Internal: `kitesforu-workers/src/workers/stages/audio/providers/inworld/shared.py`
- KitesForU Internal: `kitesforu-workers/src/workers/stages/audio/providers/capabilities.py`
