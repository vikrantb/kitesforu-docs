# Framework: Body-state and Physical-condition Cues for Natural AI Speech
**Specialist:** Body-State & Physicality Agent
**Context:** KitesForU Audio Research 2026

## 1. The Core Philosophy of Embodied Speech
Natural speech is not produced by a "voice box" in a vacuum; it is the output of a biological system under specific physical constraints. To move beyond "uncanny valley" TTS, we must model the **body** behind the voice.

### The Triad of Physicality
1.  **Systemic Condition:** The internal state of the body (Fatigue, Illness, Exertion, Age).
2.  **Postural/Positional State:** The physical orientation of the body (Lying down, Straining, Turning away).
3.  **Environmental Interaction:** How the body reacts to the world (Temperature, Proximity, Sensory input).

---

## 2. The Body-State Taxonomy (Actionable Tags)

### A. Exertion & Respiratory Load
*   **[Exertion: Low]** - Regular speech while walking or moving lightly.
*   **[Exertion: High]** - Speaking while running or heavy lifting. *Effect: Speech comes in short bursts, heart rate affects rhythm.*
*   **[Winded]** - Recovering from exertion. *Effect: Heavy inhales between shorter phrases.*
*   **[Breath-held]** - Speaking while performing a task requiring core stability. *Effect: Strained, higher pitch, sudden release of air at end.*

### B. Positional & Spatial Embodiment
*   **[Supine/Lying Down]** - *Effect: Increased chest resonance, slower onset of words, "lazy" articulation.*
*   **[Cervical Strain]** - Looking up or neck twisted. *Effect: Constricted throat, slightly thinner tone.*
*   **[Proximity: Intimate]** - Leaning into the "ear" of the listener. *Effect: Increased bass (proximity effect), audible soft inhales, reduced volume.*
*   **[Spatial: Projecting]** - Calling to someone 10+ feet away. *Effect: Increased volume, lengthened vowels, lack of mouth micro-sounds.*

### C. Systemic & Biological Cues
*   **[Vocal Fatigue]** - Long-day exhaustion. *Effect: Hoarseness (fry), "breathy" quality, descending pitch at ends of sentences.*
*   **[Biological: Cold]** - Freezing environment. *Effect: Jaw tightness, slight tremor in long vowels, clipped consonants.*
*   **[Biological: Recovering]** - Post-crying or post-illness. *Effect: "Wet" nasal quality, audible swallows, shaky onset.*

---

## 3. Critical Analysis & Recommendations

### Missing in Current Proposals:
*   **The Biometric Rhythm:** We need to move from static tags (e.g., "sad") to **Biometric Envelopes**. A "High Exertion" state should automatically adjust the *Tempo* and *Pause Frequency* across a whole paragraph, not just add a "breath" sound at the start.
*   **Age-Physicality Mapping:** Physical cues must be scaled by the character's age. An "Exhausted" child sounds different (whiny, high-pitched) than an "Exhausted" elder (low, gravelly).

### What is Overkill:
*   **Mouth-Sound Saturation:** Avoid over-annotating "lip smacks" and "clicks." These should be randomized "noise floor" elements rather than intentional performance tags, unless the scene specifically calls for "eating" or "drinking."
*   **Cognitive States as Physical Tags:** "Drunk" or "High" should not be tags. They are **Performance Styles** composed of physical sub-tags (slur, slow-tempo, low-stability).

---

## 4. Platform-Specific Strategy for KitesForU

### Use Case 1: The "Classroom/LMS" Presence
To avoid "Robot Teacher" syndrome, use **[Exertion: Low]** and **[Spatial: Pacing]**. This simulates a teacher moving around a room, making the audio feel dynamic and less like a static recording.

### Use Case 2: Narrated Series (The "Third Person Limited")
For internal monologues, use **[Proximity: Intimate]** combined with **[Supine]**. This places the listener "inside the head" of the character, creating a psychological closeness that standard narration lacks.

### Use Case 3: The "Action" Scene
When characters are in motion, the LLM must generate scripts with **Incomplete Sentences** and **[Exertion: High]** tags. 
*   *Bad:* "I am running away from the monster now."
*   *Good:* "[out of breath] I— [heavy exhale] —I can't... keep this up!"

---

## 5. Implementation Roadmap for LLM Scripting

When directing the Script-Generator LLM, use the following "Physical Consistency" rules:
1.  **Check Physical Budget:** Does the character have the "air" to finish this long sentence? If they are [Winded], break the sentence into 3 parts.
2.  **Match Tone to Posture:** If the character is [Lying Down], the tone should not be [High Energy/Projecting].
3.  **Sequence of Recovery:** If a character was [Crying], the next 2-3 lines must include [Biological: Recovering] cues (sniffles/wet voice) even if the dialogue moves to a new topic.

---
*Created by Gemini CLI - Body-State Specialization Agent - Feb 2026*
