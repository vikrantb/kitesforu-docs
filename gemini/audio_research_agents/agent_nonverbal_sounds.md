# Framework: Nonverbal Mouth and Body Micro-sounds for Natural AI Speech
**Author:** Expert Agent (Nonverbal Mouth and Body Micro-sounds)
**Status:** Reusable Architectural Reference
**Date:** February 2026

## 1. Executive Summary
Natural human speech is "embodied." It is not just the production of phonemes but the result of biological processes (breathing, swallowing, mouth hydration) and emotional "leakage." This framework identifies how to move beyond basic emotion tags to a system of **Biomechanical Micro-signals** that ground AI speech in physical reality.

---

## 2. Internalized Principles (from ChatGPT Proposal)
The proposal correctly identifies that naturalness comes from *imperfection*. Key takeaways I am adopting:
- **Vocal Performance Layer:** Separating "what" is said from "how" it is embodied.
- **Categorization of Naturalness:** Breaking down into pauses, airflow, laughter, strain, and backchanneling.
- **Annotation Language:** Using a bracketed schema `[emotion | delivery | pacing | vocalization]` for LLM-to-TTS communication.

---

## 3. Critical Perspective: The "Expert's Gap"
While the proposal is comprehensive, it misses several critical "expert-level" nuances required for true high-fidelity synthesis:

### What is Missing?
1. **Biomechanical Sequencing (The "Gulp" Rule):** Humans cannot swallow and speak simultaneously. Current LLM generators often place a `[swallow]` tag *during* a word. A robust framework must enforce sequence: `[swallow] -> [inhale] -> [speech]`.
2. **Mouth Hydration Dynamics:** Dehydration (dry mouth) leads to different click sounds (xerostomia-induced clicks) vs. wet mouth sounds (smacking). This changes based on the character's state (e.g., "nervous" usually equals "dry mouth").
3. **Environmental Acoustic Mapping:** A "scoff" sounds different in a bathroom vs. a forest. The framework should include **Acoustic Occlusion** markers for micro-sounds which are often the first to be lost in distance.
4. **Cultural Micro-Phonetics:** Nonverbal sounds are not universal. 
    - *Example:* The Hindi "Acha" (अछा) often carries a specific glottal stop or nasalized hum that isn't just "okay."
    - *Example:* The "tutting" (tsk-tsk) frequency and duration vary wildly between cultures.

### What is Overkill?
1. **Granular Pacing Overlap:** Distinguishing between `staccato`, `clipped`, and `measured` in a single prompt is often "hallucination bait" for current TTS engines. It is better to use **Primary Rhythmic Drivers** (e.g., `Urgent` vs `Languid`) and let the engine derive the rest.
2. **Post-Processing Sounds:** Cues like "speaking while walking" are better handled by adding rhythmic Foley (footsteps) and slight gain-oscillation in post-production rather than forcing the TTS to simulate the "jostle" in the vocal cords, which often just sounds like jitter.

---

## 4. KitesForU Platform Integration
For KitesForU, we must balance **high-engagement storytelling** (Series/Drama) with **instructional clarity** (Courses).

### Strategy for KitesForU:
- **The "Hindi-First" Micro-sound Library:**
    - Focus on specific fillers: `[nasal-hum]`, `[glottal-affirmation]`.
    - Handle aspirated breath patterns common in North Indian dialects which add "warmth" (Maternal/Paternal tones).
- **Cost-Efficiency Calibration:**
    - Use **ElevenLabs v3** only for "Hero" content (Series/Main Characters).
    - Use **Gemini 2.5 Flash** with my "Naturalness Tokens" for mass-scale content. Gemini's "Director's Notes" feature is perfectly suited for these micro-sounds.
- **Educational Guardrails:**
    - In "Course" mode, the system should automatically "dial down" mouth sounds (clicks/smacks) by 70% to prevent ASMR-triggering or distraction, focusing instead on **Intelligent Pacing** and **Clarifying Breaths**.

---

## 5. Reusable Framework: The Biomechanical Markup (BMM)
Future LLM prompts should use this structured BMM for the best results:

### Category: Hydration & Oral Prep
- `[click_tongue_dry]`: Skepticism, annoyance.
- `[moisten_lips]`: Preparation to speak something difficult/long.
- `[audible_swallow]`: Nervousness, fear, "swallowing pride."

### Category: Airflow & Glottal
- `[sharp_inhale_nose]`: Sudden realization, offense.
- `[slow_hiss_teeth]`: Restrained anger, physical pain.
- `[glottal_stop_breath]`: Sudden interruption, shock.

### Category: Phrasal Repair (The "Human" Layer)
- `[false_start]`: Repeating the first syllable (e.g., "I- I think").
- `[filler_nasal]`: The "um" that vibrates in the nose rather than the throat (common in deep thought).

### Category: Relationship Dynamics (Social Micro-sounds)
- `[sympathetic_hum]`: Low frequency, 0.5s duration.
- `[derisive_scoff]`: High frequency, sharp exit.

---

## 6. Implementation Recommendation
When generating scripts for KitesForU, the "Script Naturalizer" agent should perform a **Secondary Pass** specifically for these nonverbal sounds, ensuring they do not violate biomechanical logic (e.g., no breathing while laughing).

*This analysis serves as a foundation for the KitesForU Audio Engineering standards for 2026.*
