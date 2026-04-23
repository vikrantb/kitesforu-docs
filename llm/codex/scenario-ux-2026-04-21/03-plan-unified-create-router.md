# Plan B — Unified Create Router

## Goal

Keep specialized creation flows, but stop making users figure out which one to use.

## Current problem

Creation is split across multiple mental models:

- `/create-smart`
- `/classes/create`
- `/interview-prep`

The user has to translate their goal into the correct internal route.

That is backwards.

## Product principle

The user chooses the **job**.
The system chooses the **flow**.

## Proposed structure

### Step 1 — one universal create entry

Use `/create` as the canonical creation entry.

Top-level options should be job-based:

- Prep for an interview
- Create a class
- Train my team
- Study a topic
- Turn notes into audio
- Write a brief

### Step 2 — smart route into the right specialized flow

Examples:

- `Prep for an interview` -> interview-prep flow
- `Create a class` -> class-creation flow
- `Train my team` -> corporate training flow
- `Study a topic` -> smart-create course flow

### Step 3 — shared create shell, specialized internals

The outer structure should feel consistent:

- same header
- same promise
- same step framing
- same progress language

But each scenario keeps its own internals where needed.

## What to remove

- hidden route decisions from the user
- content-type jargon at the point of creation
- education-specific framing on corporate create
- interview-prep-first quick actions for teacher personas

## Specific fixes

### Corporate

Corporate users should never hit a page titled `Create a Classroom`.

### Teacher

Teacher users should never have to infer that "Classrooms" is the correct branch of a broader creation system.

### Interview

Interview prep should stay specialized, but be reachable from the same top-level creation logic.

## Why this matters

You do not want one generic create flow.
You want one **clear entry point** into the right create flow.
