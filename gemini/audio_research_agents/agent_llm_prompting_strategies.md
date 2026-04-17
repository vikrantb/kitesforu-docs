# LLM Script Generation Prompting Strategies for Natural AI Speech

## 1. Executive Summary
This document outlines a robust, future-proof framework for prompting LLMs to generate audio-first scripts. While traditional scriptwriting focuses on semantic content, **Natural Speech Prompting (NSP)** focuses on the "pragmatic layer"—the breaths, hesitations, and emotional leaks that signal human presence. For KitesForU, this framework prioritizes a balance between **engagement** and **clarity**, ensuring that AI voices sound like relatable mentors rather than robotic announcers.

---

## 2. The Multi-Layer Prompting Architecture
To generate high-quality audio, the LLM must process the script in three distinct layers:

### Layer A: The Semantic Layer (The "What")
*   **Goal:** Convey information or narrative.
*   **Prompt Constraint:** Use "Spoken-word English/Hindi/Spanish" rather than "Written-word." Avoid complex subordinate clauses that are hard to parse by ear.

### Layer B: The Pragmatic Layer (The "How")
*   **Goal:** Add human behavior (hesitations, fillers, repairs).
*   **Prompt Constraint:** Inject language-specific fillers. 
    *   *English:* "I mean," "you know," "sort of."
    *   *Hindi:* "Matlab," "dekhiye," "voh kya hai na."

### Layer C: The Acoustic Metadata Layer (The "Direction")
*   **Goal:** Provide instructions to the TTS engine.
*   **Prompt Constraint:** Use "Director's Notes" or "Tags" based on the target provider.

---

## 3. Core Framework: Naturalness Tokens

### 3.1 Conversational Debris (The "Glue")
Instruct the LLM to use these to bridge thoughts:
*   **False Starts:** "We—actually, we should look at the data first."
*   **Self-Corrections:** "There were five—no, six people there."
*   **Thinking Pauses:** "[beat] ... let me think."

### 3.2 Biometric Leakage (The "Body")
Use these sparingly to signal emotion:
*   **The Inhale:** Use `[sharp inhale]` before a surprising fact.
*   **The Exhale:** Use `[relieved sigh]` after resolving a complex topic.
*   **The Smile:** Use `[smiling voice]` to increase warmth and trust.

### 3.3 Backchanneling (The "Presence")
Essential for multi-speaker content (Interviews/Conversations):
*   **Active Listening:** `(soft mm-hmm)`, `(right)`, `(exactly)`.

---

## 4. Provider-Specific Mapping Strategy
Different TTS engines require different prompting styles. The LLM should be instructed to output in a format that a "Translation Layer" can handle:

| Provider | Strategy | Example Output |
| :--- | :--- | :--- |
| **ElevenLabs v3** | Parenthetical Emotion | `(whispering) This is a secret.` |
| **OpenAI / Gemini** | Natural Language Instructions | `[Sound like a warm, encouraging mentor]` |
| **Cartesia / Fish** | Granular Emotion Tags | `<emotion: excitement>Check this out!</emotion>` |
| **Generic/SSML** | Pacing & Pitch Tags | `<break time="500ms"/>` |

---

## 5. KitesForU Specific Application: The "Relatable Mentor"

KitesForU content (Education/Corporate Training/Life-Long Learning) requires a specific persona: **The Relatable Mentor**.

### The "Relatable Mentor" Profile:
*   **Voice Style:** Warm, authoritative but accessible, slightly informal.
*   **Naturalness Level:** Moderate (15-20% tag density).
*   **Key Behavior:** 
    *   Uses **"Check-ins"**: "Makes sense, right?" 
    *   Uses **"Encouragement Breaths"**: `[soft chuckle]` before simplifying a hard concept.
    *   **Hindi Nuance:** Use "Hinglish" (Hindi-English mix) for modern urban personas, or "Shuddh Hindi" with gentle pauses for more traditional contexts.

### Avoid the "Uncanny Valley" in Learning:
*   **Overkill:** Do not use "voice cracks" or "sobs" in a tutorial about Excel. 
*   **Overkill:** Avoid "wet mouth sounds" or "heavy breathing" which can feel invasive in an e-learning environment.

---

## 6. Prompt Engineering Templates

### High-Level System Instruction (for LLM)
> "Act as an expert Audio Script Director. When generating scripts, your output is for a Text-to-Speech engine, not a reader. Prioritize the rhythm of speech over the grammar of writing. Use contractions (it's, won't). Insert natural pauses using `[beat]` or `[long pause]`. Add emotional direction in brackets at the start of paragraphs. For KitesForU, maintain a 'Relatable Mentor' persona: warm, clear, and engaging, with occasional self-corrections to sound more human."

### Scene-Specific Prompt (for LLM)
> "Scenario: Explaining a complex financial concept to a beginner.
> Persona: Friendly, patient.
> Language: Hindi.
> Direction: Use gentle thinking pauses. If you hit a technical term, pause `[beat]` and explain it simply as if you just thought of a better analogy."

---

## 7. Future-Proofing: Semantic-Acoustic Coupling
As TTS models move toward "Speech-to-Speech" or "Fully Steerable LLM-TTS" (like Gemini 2.5), the need for manual tags will decrease. The prompting strategy should transition from **Tags** to **Intent**:
*   *Current:* `[voice cracking with emotion]`
*   *Future:* `[Perform this line with the weight of someone who has lost everything but is trying to stay brave for their child.]`

The LLM should be trained to provide these **High-Context Intent Blocks** which modern models can "read" to generate the appropriate prosody.

---

## 8. Implementation Roadmap for KitesForU
1.  **Lexicon Development:** Create a standardized library of "Naturalness Tags" (e.g., `[L1]` for short pause, `[E1]` for warmth).
2.  **LLM Tuning:** Prompt the script-generator to strictly adhere to the "Relatable Mentor" persona.
3.  **Language Adaptation:** Develop specific filler-word libraries for Hindi, Spanish, and French to be injected by the LLM.
4.  **A/B Testing:** Compare "Clean Scripts" vs. "NSP Scripts" with user groups to find the optimal naturalness-to-clarity ratio.
