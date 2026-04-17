# Guided Voice-First "Paved Path" Architecture

**Status**: Draft - Strategic Pivot
**Date**: 2026-04-13
**Author**: Lead Architect (Pro)
**Directive**: Shift from text-heavy UI to a zero-friction, voice-first guided interview preparation flow.

## 1. The Core Philosophy
Users come to the system to prepare for an interview, not to fill out forms. The experience must be magical, guided, and entirely navigable via voice. Text input is an anti-pattern. 

## 2. The Three-Stage "Paved Path"

### Stage 1: Intent & Context (The Intake)
- **Action**: User speaks their goal: "I'm preparing for an HR Analyst interview at Stripe."
- **Context Injection**: The system silently checks for an existing resume profile. If present, it's injected into the context window. If not, it's omitted. No forms are required.

### Stage 2: The Dynamic Syllabus (The Map)
- **Action**: The `GuidedPathEngine` (Backend) generates a structured curriculum based on the intent.
- **Example Flow**:
  1. Behavioral: Stakeholder Management
  2. Technical: HRIS Systems & Data Integrity
  3. Scenario: Conflict Resolution
- **Steering**: The user can interrupt via voice at any point. "Actually, I want to skip the technical part and focus entirely on conflict resolution." The engine dynamically prunes the graph and updates the path.

### Stage 3: The Magical Simulation (The Arena)
- **Action**: The user enters the simulation.
- **Mechanics**: 
  - Utilizes existing expressive TTS voices and tuning blocks.
  - Supports interruption (AI can talk over the user, user can talk over the AI).
  - Feedback is delivered verbally, in-character ("That was a good start, but remember to use the STAR method. Let's try that answer again, focusing on the Result."). No text-heavy scorecards.

## 3. Technical Implementation Plan
1. **Backend (`kitesforu-workers`)**: Create a new `GuidedPathEngine` state machine that handles the intake, syllabus generation, and state transitions without requiring frontend route changes.
2. **Frontend (`kitesforu-frontend`)**: Strip away the `create-smart` forms. Replace them with a single, dominant voice-capture surface that streams audio directly to the `GuidedPathEngine`.
3. **Audio Infrastructure**: Wire up the interruption handling and expressive voice selection logic to the new simulation endpoints.

## 4. Resource Allocation
- Heavy generative tasks (schemas, rote backend boilerplate) will be delegated to Claude Code in tightly constrained, single-shot prompts to avoid token bloat once its quota resets.
- Architectural design and complex orchestration will be handled by the Lead Architect.
