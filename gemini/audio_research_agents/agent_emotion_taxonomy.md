# KitesForU: Emotion Taxonomy & Vocalization Framework (2026)

## 1. Executive Summary
This framework establishes a standardized language for "Performance-Aware TTS." It bridges the gap between LLM script generation and TTS provider execution. Instead of vague descriptors, it uses a **Layered Emotional Logic** designed for KitesForU’s multilingual and multi-genre requirements.

---

## 2. The Three-Layer Taxonomy

### Layer 1: The Emotional State (Global Intent)
*Purpose: Sets the baseline prosody for the TTS model.*

| State | Acoustic Profile (General) | KitesForU Use Case |
| :--- | :--- | :--- |
| **Neutral-Inquisitive** | Stable pitch, rising endings | Educational walkthroughs, AI Tutors. |
| **Empathetic-Warm** | Lowered register, soft onset | Mentorship, mental health content, comforting stories. |
| **Urgent-Authoritative** | High intensity, fast tempo | Security alerts, climax of a narrative, "Car Mode" prompts. |
| **Vulnerable-Fraying** | Unstable pitch, frequent micro-pauses | Personal testimonials, dramatic storytelling. |
| **Buoyant-Playful** | Wide pitch range, rhythmic variability | Gamified learning, children's content, banter. |

### Layer 2: The Texture Layer (Acoustic Parameters)
*Purpose: Provides specific directions for prompt-steerable models (Gemini, OpenAI, ElevenLabs v3).*

*   **Breathiness (0-1):** High for intimacy/vulnerability; Low for authority.
*   **Tension (0-1):** Clenched jaw (Anger/Stress) vs. Relaxed (Joy/Relief).
*   **Brightness:** Forward-resonance (Happiness/Clarity) vs. Dark-resonance (Sadness/Mystery).
*   **Pace Fluidity:** Robotic (Monotone) vs. Staccato (Anxiety) vs. Legato (Comfort).

### Layer 3: The Artifact Layer (Naturalness Tokens)
*Purpose: Discrete markers to be injected into scripts.*

#### A. Vocalizations (Non-Verbal)
*   `[soft_inhale]`: Preparation for a difficult truth.
*   `[weighted_exhale]`: Relief or exhaustion.
*   `[micro_laugh]`: A breathy chuckle that doesn't interrupt flow.
*   `[voice_crack]`: Single-point emotional peaking.

#### B. Linguistic Fillers (Multilingual/Localized)
*   **English:** `um`, `uh`, `well...`, `I mean`.
*   **Hindi:** `voh...` (that...), `matlab...` (meaning...), `toh...` (so...).
*   **Spanish:** `este...`, `pues...`.

---

## 3. Critical Analysis: Missing vs. Overkill

### What was Missing (Corrected in this Framework)
1.  **Emotional Inertia:** Cues should not just be *per line*. We implement "Decay Rates." If a line is `[SAD]`, the first 20% of the next line should inherit the acoustic profile of sadness.
2.  **Cultural Resonance:** Laughter and pauses are culturally coded. In Hindi content, "respectful silence" (`[shant_silence]`) carries more weight than in Western banter.
3.  **Mic-Distance Awareness:** Naturalness is often about "perceived proximity."
    *   *Close-Mic:* Intimate, whispers, breathy (The "Friend" persona).
    *   *Room-Mic:* Echo, higher volume, less breath (The "Teacher" persona).

### What is Overkill (Optimized for Production)
*   **Biological Artifacts:** "Swallowing" and "Licking lips" are often rejected by listeners as "creepy" in an educational context. **Recommendation:** Disable these for KitesForU Learning, enable only for "KitesForU Stories."
*   **Hyper-Granular Laughter:** Current TTS models struggle to distinguish between 15 types of laughter. **Standardize to 3:** `[joyous]`, `[nervous]`, `[ironic]`.

---

## 4. Provider Implementation Mapping

| Provider | Mechanism | Best Use Case for KitesForU |
| :--- | :--- | :--- |
| **ElevenLabs v3** | Parenthetical prompts `(whispers)` + Style Exaggeration. | High-stakes narrative, "KitesForU Stories." |
| **Gemini 2.5 Flash** | Natural Language "Director's Notes." | **Primary for Multilingual (Hindi/Spanish)**; cost-effective. |
| **OpenAI Mini** | Instructions parameter (State-based). | Conversational agents, "Car Mode" interactivity. |
| **Cartesia Sonic** | SSML-like tags for laughter/breathing. | Real-time interactive sessions where latency matters. |

---

## 5. KitesForU Implementation Guidelines

### The "KitesForU Script Protocol" (KSP)
When the LLM generates a script, it must output a JSON structure or tagged text:

```json
{
  "line": "I didn't think you'd actually show up.",
  "performance": {
    "state": "vulnerable",
    "texture": "breathy",
    "artifacts": ["soft_inhale", "hesitation"],
    "mic_distance": "close"
  }
}
```

### Future-Proofing Strategy
1.  **Decouple Emotion from Text:** Store the `performance` metadata separately from the `line`. If we switch providers (e.g., from ElevenLabs to a better open-source model), we only need to update the *mapping* layer, not the entire library of content.
2.  **A/B Test the "Uncanny Valley":** Periodically test if users prefer "Pure Natural" (with mouth sounds) vs. "Enhanced Natural" (breath/pace only).
3.  **Hindi-First Validation:** Prioritize localizing the Artifact Layer for Hindi, as this is the platform's key growth area.

---
*Created by: Emotion Taxonomy Expert Agent*
*Status: Reusable Framework for KitesForU Audio Strategy 2026*
