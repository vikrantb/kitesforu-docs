# Framework: Emotional Strain, Hurt, and Grief in AI Speech Synthesis
**Expert Agent:** Emotional Strain Specialist
**Version:** 1.0 (2026)
**Platform Focus:** KitesForU (Multilingual Expressiveness)

## 1. The Taxonomy of Strain
Instead of treating "crying" as a single tag, this framework divides emotional strain into four psychological "Pressure Zones" that map to TTS model capabilities.

### Zone A: Suppression (The "Tight" Voice)
*   **Internal State:** Holding it together, masking, professional distance.
*   **Vocal Profile:** High tension, clipped endings, minimal pitch variance, audible swallows.
*   **KitesForU Use Case:** A narrator or character relaying difficult news while trying to remain "composed."
*   **Key Tags:** `[strained composure]`, `[brittle tone]`, `[swallowed hurt]`, `[thin delivery]`.

### Zone B: The Leak (The "Cracking" Voice)
*   **Internal State:** The moment the mask slips. High "vocal fry" and pitch instability.
*   **Vocal Profile:** Voice cracks on high-energy vowels, shaky inhales, trailing off.
*   **KitesForU Use Case:** The climax of a personal story or a "moment of realization" in a module.
*   **Key Tags:** `[voice cracks]`, `[shaky inhale]`, `[unsteady pitch]`, `[breath hitch]`.

### Zone C: The Release (The "Wet" Voice)
*   **Internal State:** Active crying/breakdown.
*   **Vocal Profile:** Nasal quality, "wet" texture (sniffles), irregular pacing (staccato bursts of speech).
*   **KitesForU Use Case:** Dramatic reenactments or high-empathy scenarios.
*   **Key Tags:** `[tearful]`, `[sniffles]`, `[speaking through tears]`, `[broken delivery]`.

### Zone D: The Hollow (The "Dead" Voice)
*   **Internal State:** Deep grief, numbness, exhaustion.
*   **Vocal Profile:** Lowered volume, "flat" affect, long silences, airless quality.
*   **KitesForU Use Case:** Reflecting on loss or the "aftermath" of a crisis.
*   **Key Tags:** `[hollow]`, `[numb]`, `[airless whisper]`, `[defeated tone]`.

---

## 2. Platform-Specific Strategy for KitesForU

### ElevenLabs v3 (Primary for Hindi/English)
*   **Strategy:** Use **Parenthetical Director's Notes**.
*   **Technique:** Instead of `[sniffle]`, use `(with a heavy heart and a voice close to breaking)`. ElevenLabs "understands" the semantic intent better than the mechanical instruction.
*   **Validation:** Ensure the "Style Exaggeration" is set to 0.45 for Hindi to avoid the "US Accent bleed" while maintaining emotion.

### Google Gemini 2.5 (Budget/Scale)
*   **Strategy:** **Natural Language Prompting**.
*   **Technique:** Feed the entire paragraph into the "Director's Notes" field: *"The speaker is a mother in Delhi who just lost her job. She is trying to sound brave for her child but her voice is trembling with fear."*
*   **Benefit:** Gemini’s LLM-core translates this "story" into vocal prosody more effectively than SSML tags.

---

## 3. Critical Nuances (Missing in General Proposals)

### The "Persistent Echo" (Recovery)
**Rule:** When a character "breaks down," the emotional markers must persist for at least **150-300 characters** afterward.
*   *Bad:* `[crying] I can't do this. [normal] Anyway, let's move on.`
*   *Good:* `[crying] I can't do this. [residual shakiness] I'm sorry. I just... [heavy exhale] I need a second. [returning to composure] Okay. Let's move on.`

### The "Hindi Viraha" (Cultural Specificity)
In KitesForU's Hindi content, "Grief" should be directed toward **Yearning**.
*   **Directing Note:** "Focus on the 'mushiness' of the consonants and the elongation of the final vowels in the sentence." This mimics the traditional Indian vocal expression of deep sorrow.

---

## 4. Implementation Guidelines (Anti-Overkill)

1.  **Tag Density:** No more than ONE emotion-state tag per 2 sentences.
2.  **Silence is a Sound:** Use `[long beat]` instead of `[sobbing]` to imply depth. Let the listener's mind fill in the tears.
3.  **Avoid "Mechanical" Tags:** Never use `[click tongue]` or `[swallow]` in the middle of a high-strain sentence; it often sounds like a digital error. Use them during the *silence* between sentences.

---

## 5. Reusable Prompt Fragment for KitesForU LLMs
> "When generating scripts for high-emotion scenes, prioritize the 'Shadow of Emotion'—the numbness and the recovery—rather than just the peak of the breakdown. Use suppression markers (clipped, brittle) for tension and hollow markers (airless, flat) for deep grief. For Hindi content, infuse a sense of melodic yearning. Avoid mechanical micro-tags; use psychological state descriptions in brackets."
