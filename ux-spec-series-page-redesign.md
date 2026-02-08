# UX Design Specification: Series Page Redesign
# Progressive Episode Unlock & Generation

**Product**: KitesForU
**Document Version**: 1.0
**Date**: 2026-02-07
**Scope**: Redesign of `/courses/[courseId]` series detail page
**Status**: Draft

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [User Flow Diagram](#2-user-flow-diagram)
3. [Page Layout Specification](#3-page-layout-specification)
4. [Episode Card States](#4-episode-card-states)
5. [Interaction Patterns](#5-interaction-patterns)
6. [Credit Communication](#6-credit-communication)
7. [Confirmation Dialogs](#7-confirmation-dialogs)
8. [Empty, Loading, and Error States](#8-empty-loading-and-error-states)
9. [Mobile Considerations](#9-mobile-considerations)
10. [Accessibility Requirements](#10-accessibility-requirements)
11. [Micro-copy](#11-micro-copy)
12. [Technical Data Model Changes](#12-technical-data-model-changes)
13. [Component Inventory](#13-component-inventory)

---

## 1. Executive Summary

### Problem

The current series detail page operates on an all-or-nothing model: after the curriculum is generated, the user is presented with a single "Generate All Episodes" button. Clicking it deducts all credits at once and generates every episode simultaneously. Users have no control over which episodes to generate, no ability to review or edit content before committing credits, and no way to pace their spending.

### Solution

Replace the bulk generation flow with a progressive, per-episode unlock-and-generate model:

- Display the full curriculum preview immediately after curriculum generation.
- Episode 1 is unlocked by default (no credit cost to unlock).
- Remaining episodes appear in a "locked" state with a visible per-episode credit cost.
- Users unlock episodes one at a time, with credits deducted per unlock.
- Once unlocked, users can edit the episode title and description.
- Users explicitly click "Generate" on individual episodes to start audio generation.
- The experience is progressive, user-controlled, and credit-transparent.

### Design Principles Applied

| Principle | Application |
|-----------|-------------|
| Progressive disclosure | Locked episodes reveal details only after unlock |
| User control | Per-episode unlock, edit, and generate decisions |
| Feedback loops | Clear status for every action; real-time generation progress |
| Credit transparency | Cost shown before every deduction; running balance visible |

---

## 2. User Flow Diagram

### Primary Flow: Form Submission to Episode Playback

```
[Create Series Form]
        |
        v
[POST /v1/courses] --> redirect to /courses/{courseId}
        |
        v
[Series Detail Page - Status: draft]
  - Header shows topic, style, language, episode count
  - CTA card: "Generate Your Curriculum" button
  - Episodes section: empty state ("No curriculum yet")
        |
        v  (user clicks "Generate Curriculum")
[Status: syllabus_generation]
  - Polling every 3s for curriculum completion
  - Animated skeleton loading in episodes section
  - Header status badge: "Generating Curriculum" (purple, animated)
        |
        v  (30-60 seconds later)
[Status: curriculum_ready]
  - Full curriculum visible
  - Credit Balance Bar appears at top of episodes section
  - Episode 1: UNLOCKED state (free, auto-unlocked)
  - Episodes 2-N: LOCKED state (shows credit cost per episode)
  - CTA card removed; replaced by inline episode controls
  - "Unlock All Remaining" secondary action in section header
        |
        +---> [User unlocks Episode 2]
        |       - Confirmation dialog with credit cost
        |       - Credits deducted on confirm
        |       - Episode 2 transitions to UNLOCKED state
        |       - Title and description become editable
        |
        +---> [User edits Episode 2 title/description]
        |       - Inline edit mode (existing pattern, extended)
        |       - Save/Cancel buttons
        |       - Optimistic update with rollback on failure
        |
        +---> [User clicks "Generate" on Episode 1]
        |       - Confirmation dialog (no additional credits; generation starts)
        |       - Episode 1 transitions to GENERATING state
        |       - Progress indicator on the episode card
        |       - Polling for status updates
        |
        +---> [Episode 1 generation completes]
        |       - Episode 1 transitions to COMPLETED state
        |       - Play button appears
        |       - Download button appears
        |       - Audio player available
        |
        +---> [Episode generation fails]
                - Episode transitions to FAILED state
                - Error message displayed
                - "Retry" button available (no additional credits)
```

### Secondary Flows

**Bulk Unlock Flow:**
```
[User clicks "Unlock All Remaining"]
        |
        v
[Confirmation Dialog]
  - Shows total credit cost for remaining locked episodes
  - Shows current balance and balance after unlock
  - "Unlock N Episodes for X Credits" / "Cancel"
        |
        v  (confirm)
[All remaining episodes transition to UNLOCKED]
  - Each episode now editable and ready to generate
```

**Bulk Generate Flow (unlocked episodes only):**
```
[User clicks "Generate All Unlocked"]
        |
        v
[Confirmation Dialog]
  - Shows count of unlocked-but-ungenerated episodes
  - No additional credit cost (already paid at unlock)
  - "Generate N Episodes" / "Cancel"
        |
        v  (confirm)
[All unlocked episodes begin generation]
  - Sequential or parallel generation per backend capacity
  - Individual progress tracked per episode
```

**Re-entry Flow (returning user):**
```
[User navigates to /courses/{courseId}]
        |
        v
[Page renders current state]
  - Completed episodes: playable with audio controls
  - Generating episodes: progress indicators
  - Unlocked episodes: editable, generate button available
  - Locked episodes: unlock button with credit cost
  - Failed episodes: retry button
```

---

## 3. Page Layout Specification

### Overall Page Structure

```
+----------------------------------------------------------+
|  Breadcrumb + Actions Bar                                 |
|  [My Series > Series Title]            [Play All] [Back]  |
+----------------------------------------------------------+
|                                                           |
|  Series Header Card                                       |
|  +------------------------------------------------------+ |
|  | [Status Badge]  5 episodes  5 min each  Explainer    | |
|  |                                                      | |
|  | Introduction to Machine Learning            [Progress| |
|  | Clear, educational explanations...           Circle] | |
|  |                                                      | |
|  | Created Jan 15, 2026  Total: 25 min  en-US          | |
|  +------------------------------------------------------+ |
|                                                           |
|  Credit Balance Bar (sticky on scroll, curriculum_ready+) |
|  +------------------------------------------------------+ |
|  | Credits: 847 remaining   |   Est. cost: 50 credits   | |
|  +------------------------------------------------------+ |
|                                                           |
|  Episodes Section                                         |
|  +------------------------------------------------------+ |
|  | Episodes (2 of 5 unlocked)                           | |
|  | [Unlock All Remaining (30 cr)] [Generate All Unlocked]| |
|  +------------------------------------------------------+ |
|  |                                                      | |
|  | Episode 1 - UNLOCKED / COMPLETED / etc.              | |
|  | Episode 2 - LOCKED                                   | |
|  | Episode 3 - LOCKED                                   | |
|  | Episode 4 - LOCKED                                   | |
|  | Episode 5 - LOCKED                                   | |
|  |                                                      | |
|  +------------------------------------------------------+ |
|                                                           |
|  Learning Outcomes (curriculum_ready only)                 |
|                                                           |
+----------------------------------------------------------+
|  Sticky Audio Player (when episode is playing)            |
+----------------------------------------------------------+
```

### Credit Balance Bar

A new persistent element that appears once the curriculum is ready. It provides continuous credit awareness without interrupting the workflow.

**Layout:**
```
+----------------------------------------------------------+
| [coin icon] 847 credits remaining                        |
|                                                           |
| [progress bar showing credits used vs remaining]          |
|                                                           |
| This series: 10 used / 50 total est.    [View Pricing]   |
+----------------------------------------------------------+
```

**Behavior:**
- Appears when `status` is `curriculum_ready` or later.
- Sticks to the top of the episodes section on scroll (below the navbar).
- Updates in real-time when credits are deducted.
- The progress bar is segmented: green (used), orange (pending unlocks), gray (remaining balance).

### Episodes Section Header

**Layout:**
```
+----------------------------------------------------------+
| [book icon] Episodes                                      |
| 2 of 5 unlocked  |  1 ready to play                      |
|                                                           |
| [Unlock All Remaining - 30 credits]  [Generate Unlocked]  |
+----------------------------------------------------------+
```

**Behavior:**
- "Unlock All Remaining" button: visible when at least one episode is locked. Shows total credit cost in the button label.
- "Generate All Unlocked" button: visible when at least one episode is unlocked but not yet generated. Appears as a secondary/outline button.
- Both buttons hidden when all episodes are completed.

### Episode Card Layout

Each episode is rendered as a card within a vertical list. The card layout varies by state (detailed in Section 4).

**Base Card Structure (all states):**
```
+----------------------------------------------------------+
| [Number/Play]  Episode Title                    [Actions] |
|                Episode description (1-2 lines)            |
|                5 min  |  [Status Badge]                   |
+----------------------------------------------------------+
```

**Unlocked Card (expanded with edit + generate):**
```
+----------------------------------------------------------+
| [1]  Introduction to Neural Networks    [Edit] [Generate] |
|      Learn the fundamental building                       |
|      blocks of neural networks...                         |
|      5 min  |  Unlocked - Ready to generate               |
|                                                           |
|      Key Topics: [Neural Networks] [Perceptrons] [...]    |
|      Objectives: Learn X, Understand Y, Apply Z           |
+----------------------------------------------------------+
```

**Locked Card:**
```
+----------------------------------------------------------+
| [lock icon]  Episode Title (dimmed)      [Unlock - 10 cr] |
|              Episode description (dimmed, 1 line)         |
|              5 min  |  Locked                             |
+----------------------------------------------------------+
```

---

## 4. Episode Card States

### State Machine

```
                    +--------+
                    | LOCKED |
                    +--------+
                        |
                   [unlock action]
                   (credits deducted)
                        |
                        v
                   +----------+
         +-------->| UNLOCKED |<--------+
         |         +----------+         |
         |              |               |
         |        [generate action]     |
         |              |               |
         |              v               |
         |        +------------+        |
         |        | GENERATING |        |
         |        +------------+        |
         |          /        \          |
         |     [success]   [failure]    |
         |        /            \        |
         |       v              v       |
         | +-----------+   +--------+  |
         | | COMPLETED |   | FAILED |--+
         | +-----------+   +--------+
         |                  (retry returns
         |                   to GENERATING)
         |
    [edit action while UNLOCKED]
              |
              v
         +----------+
         | EDITING  |
         +----------+
         (save/cancel returns to UNLOCKED)
```

### State Definitions

#### 4.1 LOCKED

**Visual Treatment:**
- Background: `bg-gray-50` (light wash)
- Border: `border-gray-200` (standard)
- Content: Text at `text-gray-400` (dimmed)
- Episode number: Gray circle with lock icon overlay
- Description: Single line, truncated, dimmed
- Key topics / objectives: hidden

**Actions Available:**
- "Unlock" button (primary, shows credit cost)

**Accessibility:**
- `aria-disabled="false"` on the unlock button
- Card has `role="article"` with `aria-label="Episode N - Locked. Unlock for X credits"`

#### 4.2 UNLOCKED

**Visual Treatment:**
- Background: `bg-white`
- Border: `border-blue-200` (indicates actionable)
- Content: Full opacity text
- Episode number: Blue circle
- Description: Full text visible (2-3 lines)
- Key topics: Shown as tags
- Objectives: Listed

**Actions Available:**
- "Edit" button (icon button, pencil icon)
- "Generate" button (primary, orange gradient)

**Accessibility:**
- Card has `role="article"` with `aria-label="Episode N - Unlocked. Ready to edit or generate."`
- Both action buttons are focusable with descriptive `aria-label`

#### 4.3 EDITING

**Visual Treatment:**
- Same as UNLOCKED but with inline form fields
- Title: Rendered as an `<input>` with the current title pre-filled
- Description: Rendered as a `<textarea>` with the current description pre-filled
- Subtle blue border glow (`ring-2 ring-blue-200`)

**Actions Available:**
- "Save" button (primary)
- "Cancel" button (ghost)
- Keyboard: Enter to save (title field), Escape to cancel

**Accessibility:**
- Focus trapped within the editing area
- `aria-live="polite"` region for save/error feedback
- Form fields have associated `<label>` elements

#### 4.4 GENERATING

**Visual Treatment:**
- Background: `bg-yellow-50`
- Border: `border-yellow-300` with subtle animation
- Episode number: Yellow circle with animated pulse
- Progress indicator: Horizontal progress bar below the episode card content
- Status text: "Generating audio..." with spinner

**Actions Available:**
- None (generation in progress)
- Episode card is non-interactive during generation

**Progress Sub-states (mapped from job stages):**

| Job Stage | Label | Progress Bar % |
|-----------|-------|----------------|
| queued | "Queued..." | 5% (indeterminate pulse) |
| research | "Researching..." | 15% |
| script | "Writing script..." | 35% |
| voice | "Generating voice..." | 60% |
| mix | "Mixing audio..." | 80% |
| publish | "Finalizing..." | 95% |

**Accessibility:**
- `role="progressbar"` with `aria-valuenow`, `aria-valuemin="0"`, `aria-valuemax="100"`
- `aria-label="Generating Episode N: stage name, X percent complete"`
- Status text uses `aria-live="polite"` for screen reader updates

#### 4.5 COMPLETED

**Visual Treatment:**
- Background: `bg-white` (default) / `bg-gradient-to-r from-brand-50 to-orange-50` (currently playing)
- Border: `border-green-200` (default) / `border-brand-400 ring-2 ring-brand-200` (currently playing)
- Episode number: Green circle with play icon overlay
- Play button: `bg-gradient-to-br from-green-500 to-green-400` (matches existing pattern)
- Full content visible including objectives and topics (expandable)

**Actions Available:**
- Play/Pause toggle (primary interaction -- click anywhere on card)
- Download button (icon)
- Expand/Collapse details (chevron icon)

**Accessibility:**
- Card is clickable with `role="button"` and `aria-label="Play Episode N: Title"`
- Download button: `aria-label="Download Episode N"`
- Expand button: `aria-expanded="true|false"` with `aria-controls` pointing to details panel

#### 4.6 FAILED

**Visual Treatment:**
- Background: `bg-red-50`
- Border: `border-red-200`
- Episode number: Red circle with exclamation icon
- Error message: Displayed below description in `text-red-600`
- Retry count badge (if retried before): "Retry 1 of 3"

**Actions Available:**
- "Retry" button (outline, red accent) -- does NOT deduct additional credits
- "Edit" button (if episode is still in editable window)

**Accessibility:**
- Error message uses `role="alert"` for immediate screen reader announcement
- `aria-label="Episode N - Generation failed. Error: message. Retry available."`

---

## 5. Interaction Patterns

### 5.1 Unlock Episode

**Trigger:** User clicks "Unlock" button on a locked episode card.

**Sequence:**
1. Unlock button shows credit cost: "Unlock - 10 credits"
2. Click opens Confirmation Dialog (see Section 7.1)
3. On confirm:
   a. Optimistic UI: Episode card immediately transitions to UNLOCKED visual state
   b. API call: `POST /v1/courses/{courseId}/episodes/{episodeNumber}/unlock`
   c. Credit balance updates in the Credit Balance Bar
   d. On API failure: Revert to LOCKED state, show error toast
4. On cancel: Dialog closes, no change

**Animation:**
- Card transition: 300ms ease-out scale + opacity from locked to unlocked
- Lock icon morphs to number icon (CSS transition)
- Credit balance bar animates the number down (count animation)

### 5.2 Unlock All Remaining

**Trigger:** User clicks "Unlock All Remaining" button in the episodes section header.

**Sequence:**
1. Button shows total credit cost: "Unlock All Remaining - 30 credits"
2. Click opens Bulk Unlock Confirmation Dialog (see Section 7.2)
3. On confirm:
   a. All locked episodes transition to UNLOCKED simultaneously
   b. API call: `POST /v1/courses/{courseId}/unlock-all`
   c. Credit balance updates
   d. On partial failure: Show which episodes failed to unlock
4. On cancel: Dialog closes, no change

### 5.3 Edit Episode

**Trigger:** User clicks "Edit" icon button on an UNLOCKED episode card.

**Sequence:**
1. Episode card transitions to EDITING state
2. Title becomes an input field, description becomes a textarea
3. User modifies content
4. Save: `PUT /v1/courses/{courseId}/curriculum` with updated episode data
5. Cancel: Revert to original values, return to UNLOCKED state

**Validation:**
- Title: Required, 5-200 characters
- Description: Required, 10-1000 characters
- Show character count below each field
- Disable Save button until changes are made and valid

### 5.4 Generate Episode

**Trigger:** User clicks "Generate" button on an UNLOCKED episode card.

**Sequence:**
1. Click opens Generate Confirmation Dialog (see Section 7.3)
2. On confirm:
   a. Episode card transitions to GENERATING state
   b. API call: `POST /v1/courses/{courseId}/episodes/{episodeNumber}/generate`
   c. Polling begins for status updates (every 5 seconds)
   d. Progress bar updates based on job stage
3. On completion: Episode transitions to COMPLETED state
4. On failure: Episode transitions to FAILED state

### 5.5 Generate All Unlocked

**Trigger:** User clicks "Generate All Unlocked" button.

**Sequence:**
1. Click opens Bulk Generate Confirmation Dialog (see Section 7.4)
2. On confirm:
   a. All unlocked (non-generated) episodes transition to GENERATING
   b. API call: `POST /v1/courses/{courseId}/generate` (existing endpoint, modified to only generate unlocked episodes)
   c. Individual polling per episode
3. Generation proceeds sequentially or in batches per backend capacity.

### 5.6 Play Episode

**Trigger:** User clicks the play button or anywhere on a COMPLETED episode card.

**Behavior:** Same as current implementation. The `<audio>` element loads the episode's `audio_url`. Sticky audio player appears at the bottom with transport controls (play/pause, skip, seek, volume, playback speed).

### 5.7 Retry Failed Episode

**Trigger:** User clicks "Retry" on a FAILED episode card.

**Sequence:**
1. No confirmation dialog (retry is free -- credits already paid at unlock)
2. Episode transitions to GENERATING state
3. API call: `POST /v1/courses/{courseId}/episodes/{episodeNumber}/retry` (existing endpoint)
4. Same polling and progress behavior as initial generation

### 5.8 Download Episode

**Trigger:** User clicks the download icon on a COMPLETED episode card.

**Behavior:** Same as current implementation. Creates a temporary `<a>` element with `download` attribute and triggers click.

---

## 6. Credit Communication

### 6.1 Credit Balance Bar

**Location:** Top of the Episodes section, sticky on scroll when curriculum is ready.

**Content:**

```
[coin icon] 847 credits remaining

[========================-----------]
 10 used for this series  |  40 est. remaining

Series total estimate: ~50 credits   [View Pricing ->]
```

**Update Behavior:**
- Animates on credit deduction (number counts down, progress bar shifts)
- Color coding:
  - Green (>50% balance remaining)
  - Orange (20-50% balance remaining)
  - Red (<20% balance remaining)

**Insufficient Credits Warning:**
When the user's balance is insufficient to unlock the next episode:
```
[warning icon] You need 10 more credits to unlock the next episode.
[Upgrade Plan] or [Buy Credits]
```

### 6.2 Per-Episode Credit Display

Each locked episode card shows the credit cost to unlock:

```
[lock icon] Episode 3: Advanced Neural Networks    [Unlock - 10 cr]
```

The credit cost is calculated as:
```
credits = duration_min * quality_multiplier * priority_multiplier
```

For standard quality and standard priority:
```
credits = duration_min * 1.0 * 1.0 = duration_min
```

Example: A 5-minute episode at standard quality = 5 credits.

### 6.3 Credit Confirmation Pattern

Every credit-deducting action shows a clear before/after:

```
+-------------------------------------------+
|  Current balance:           847 credits    |
|  This action:              - 10 credits    |
|  ---------------------------------------- |
|  Balance after:             837 credits    |
+-------------------------------------------+
```

### 6.4 Credit Exhaustion Handling

When a user's credit balance reaches zero:

- Locked episodes remain locked but the "Unlock" button is replaced with a disabled state and tooltip: "Insufficient credits"
- A banner appears above the episode list: "You've used all your credits. Upgrade your plan or purchase a credit pack to continue."
- CTA buttons: "View Plans" and "Buy Credits"

---

## 7. Confirmation Dialogs

All dialogs use the existing `AlertDialog` component pattern from the codebase.

### 7.1 Unlock Single Episode

```
+---------------------------------------------------+
|  [unlock icon]                                     |
|                                                    |
|  Unlock Episode 3?                                 |
|                                                    |
|  "Advanced Neural Network Architectures"           |
|  5 minutes  |  Standard quality                    |
|                                                    |
|  +-----------------------------------------------+|
|  |  Current balance:        847 credits           ||
|  |  Unlock cost:           - 10 credits           ||
|  |  -----------------------------------------    ||
|  |  Balance after:          837 credits           ||
|  +-----------------------------------------------+|
|                                                    |
|  After unlocking, you can edit the title and       |
|  description before generating.                    |
|                                                    |
|           [Cancel]    [Unlock for 10 Credits]      |
+---------------------------------------------------+
```

**Primary action button:** Orange gradient (`btn-primary-elegant`)
**Cancel button:** Outline variant

### 7.2 Bulk Unlock All Remaining

```
+---------------------------------------------------+
|  [unlock icon]                                     |
|                                                    |
|  Unlock All Remaining Episodes?                    |
|                                                    |
|  This will unlock 4 episodes:                      |
|    Episode 2: Supervised Learning (5 min)           |
|    Episode 3: Neural Networks (5 min)               |
|    Episode 4: Deep Learning (5 min)                 |
|    Episode 5: Practical Applications (5 min)        |
|                                                    |
|  +-----------------------------------------------+|
|  |  Current balance:        847 credits           ||
|  |  Total unlock cost:     - 40 credits           ||
|  |  -----------------------------------------    ||
|  |  Balance after:          807 credits           ||
|  +-----------------------------------------------+|
|                                                    |
|           [Cancel]    [Unlock 4 Episodes - 40 cr]  |
+---------------------------------------------------+
```

### 7.3 Generate Single Episode

```
+---------------------------------------------------+
|  [generate icon]                                   |
|                                                    |
|  Generate Episode 1?                               |
|                                                    |
|  "Introduction to Machine Learning"                |
|  5 minutes  |  Standard quality                    |
|                                                    |
|  No additional credits will be charged.            |
|  Generation typically takes 2-5 minutes.           |
|                                                    |
|  You won't be able to edit the title or            |
|  description once generation starts.               |
|                                                    |
|           [Cancel]    [Generate Episode]            |
+---------------------------------------------------+
```

**Note:** No credit cost display needed here because credits were already deducted at unlock time.

### 7.4 Bulk Generate All Unlocked

```
+---------------------------------------------------+
|  [generate icon]                                   |
|                                                    |
|  Generate All Unlocked Episodes?                   |
|                                                    |
|  3 episodes will begin generating:                 |
|    Episode 1: Introduction (5 min)                  |
|    Episode 2: Supervised Learning (5 min)           |
|    Episode 3: Neural Networks (5 min)               |
|                                                    |
|  No additional credits will be charged.             |
|  Total generation time: approximately 10-15 min.   |
|                                                    |
|  Titles and descriptions will be locked during      |
|  generation.                                        |
|                                                    |
|           [Cancel]    [Generate 3 Episodes]         |
+---------------------------------------------------+
```

---

## 8. Empty, Loading, and Error States

### 8.1 Page Loading

**Trigger:** Initial page load while fetching course data.

**Visual:**
```
[Center of page]
  [Spinner - brand-500 colored]
  "Loading series..."
```

Uses the existing `animate-spin rounded-full h-12 w-12 border-b-2 border-brand-500` pattern from the codebase.

### 8.2 Curriculum Generating (Skeleton State)

**Trigger:** Status is `syllabus_generation`.

**Visual:**
```
+----------------------------------------------------------+
| Series Header Card (populated with known info)            |
| Status Badge: "Generating Curriculum" (purple, animated)  |
+----------------------------------------------------------+

+----------------------------------------------------------+
| [brain icon] Your curriculum is being created...          |
|                                                           |
| [animated illustration: three skeleton episode cards]     |
|                                                           |
| Episode Card Skeleton 1:                                  |
| [gray bar --------] [gray bar ----]                      |
| [gray bar ----------------]                               |
| [shimmer animation]                                       |
|                                                           |
| Episode Card Skeleton 2: ...                              |
| Episode Card Skeleton 3: ...                              |
|                                                           |
| This usually takes 30-60 seconds. You can wait here       |
| or come back later -- we'll keep generating.              |
+----------------------------------------------------------+
```

**Behavior:**
- Skeleton cards use `animate-pulse bg-gray-200 rounded` blocks for title, description, and metadata.
- Number of skeleton cards matches `total_episodes` from the course data.
- Auto-refresh every 3 seconds (existing polling pattern).

### 8.3 Draft with No Curriculum

**Trigger:** Status is `draft` and no curriculum exists.

**Visual:** Same as current implementation:
```
[Center of episodes section]
  [clipboard icon - large]
  "No curriculum yet"
  "Click the Generate Curriculum button above to create
   a detailed episode outline based on your topic."
```

### 8.4 Course Not Found (404)

**Visual:**
```
[Center of page]
  Card:
    Title: "Series Not Found"
    Description: "This series doesn't exist or you don't have access to it."
    [Back to My Series] button
```

### 8.5 Network Error / API Failure

**Visual:** Uses existing `ErrorDialog` component with `classifyError` utility.

**Inline Error (for non-blocking errors like single episode retry failure):**
```
[error banner - red-50 bg, red-200 border]
  [alert icon] Failed to unlock episode. Please try again.
  [Dismiss] [Retry]
```

**Toast Pattern (for optimistic update rollbacks):**
```
[top-center, 4s auto-dismiss]
  [alert icon] Could not unlock episode. Credits were not charged.
```

### 8.6 Insufficient Credits

**Trigger:** User attempts to unlock but has insufficient credit balance.

**Visual:**
```
+----------------------------------------------------------+
| [warning icon] Insufficient Credits                       |
|                                                           |
| You need 10 credits to unlock this episode,               |
| but you only have 3 credits remaining.                    |
|                                                           |
|        [View Plans]    [Buy Credits]    [Cancel]           |
+----------------------------------------------------------+
```

### 8.7 All Episodes Completed

**Trigger:** Every episode in the series has `status: completed`.

**Visual:** The CTA area in the episodes section header transforms:
```
+----------------------------------------------------------+
| [party icon] All episodes are ready!                      |
| Your entire series is available to play and download.     |
|                                                           |
|        [Play All]    [Share Series]                        |
+----------------------------------------------------------+
```

---

## 9. Mobile Considerations

### 9.1 Responsive Breakpoints

The design follows the existing Tailwind breakpoint system:

| Breakpoint | Width | Layout Changes |
|------------|-------|----------------|
| Default (mobile) | < 640px | Single column, stacked layouts |
| `sm` | >= 640px | Side-by-side buttons, expanded metadata |
| `md` | >= 768px | Two-column header (info + progress circle) |
| `lg` | >= 1024px | Full layout with sidebar potential |

### 9.2 Mobile Episode Card

On mobile (< 640px), episode cards simplify:

```
+------------------------------------------+
| [Number] Episode Title          [Action]  |
|          Description (1 line)             |
|          5 min | Status                   |
+------------------------------------------+
```

- Action button collapses to icon-only (lock icon for locked, play icon for completed).
- "Unlock" cost shown as a tooltip or small badge on the icon.
- Edit mode uses full-width input fields.
- Key topics and objectives are hidden behind the expand chevron.

### 9.3 Mobile Credit Balance Bar

On mobile, the Credit Balance Bar simplifies to a single line:

```
+------------------------------------------+
| [coin] 847 credits        [10 cr/episode] |
+------------------------------------------+
```

Tapping it expands to show the full breakdown.

### 9.4 Mobile Confirmation Dialogs

- Dialogs render as bottom sheets (full-width, slides up from bottom).
- Primary action button is full-width.
- Credit breakdown table uses 100% width.

### 9.5 Mobile Audio Player

The existing sticky audio player already handles mobile well. No changes needed. Episode info truncates, secondary controls (volume, playback speed) are hidden behind a "more" menu on screens < 640px (existing `hidden sm:flex` pattern).

### 9.6 Touch Targets

- All interactive elements: minimum 44x44px touch target (WCAG 2.5.5).
- Unlock buttons: Full height of the right side of the card on mobile.
- Play buttons: 48x48px circular targets (matching existing pattern).

---

## 10. Accessibility Requirements

### 10.1 Keyboard Navigation

**Tab Order (within episodes section):**
1. Credit Balance Bar (if interactive elements present)
2. "Unlock All Remaining" button
3. "Generate All Unlocked" button
4. Episode 1 card -> Episode 1 actions (Edit, Generate, Play, Download)
5. Episode 2 card -> Episode 2 actions
6. ... (continues for each episode)

**Episode Card Keyboard Interactions:**

| Key | Context | Action |
|-----|---------|--------|
| Tab | Any | Move to next focusable element |
| Shift+Tab | Any | Move to previous focusable element |
| Enter | Focused on locked card's Unlock button | Open unlock confirmation |
| Enter | Focused on completed card | Play/Pause episode |
| Enter | Focused on Generate button | Open generate confirmation |
| Space | Any button | Activate button |
| Escape | Editing mode | Cancel edit, return to unlocked state |
| Escape | Confirmation dialog | Close dialog |
| Enter | Title input in edit mode | Save changes |

**Focus Management:**
- After unlocking: Focus moves to the newly unlocked episode's first action (Edit button).
- After dialog closes: Focus returns to the triggering button.
- After generation completes: No focus change (async event).
- After save in edit mode: Focus returns to the Edit button.

### 10.2 Screen Reader Support

**Episode Card Announcements:**

```html
<!-- Locked Episode -->
<article aria-label="Episode 3: Advanced Neural Networks. Locked. 5 minutes. Unlock for 10 credits.">

<!-- Unlocked Episode -->
<article aria-label="Episode 1: Introduction to ML. Unlocked. 5 minutes. Ready to edit or generate.">

<!-- Generating Episode -->
<article aria-label="Episode 1: Introduction to ML. Generating audio. 60 percent complete. Writing script.">

<!-- Completed Episode -->
<article aria-label="Episode 1: Introduction to ML. Completed. 5 minutes. Press Enter to play.">

<!-- Failed Episode -->
<article aria-label="Episode 1: Introduction to ML. Generation failed. Error: Network timeout. Retry available.">
```

**Live Regions:**

```html
<!-- Credit balance updates -->
<div aria-live="polite" aria-atomic="true">
  Credit balance: 837 credits remaining
</div>

<!-- Generation progress updates -->
<div aria-live="polite">
  Episode 1: Generating voice audio. 60 percent complete.
</div>

<!-- Error announcements -->
<div role="alert">
  Failed to unlock episode. Credits were not charged.
</div>
```

### 10.3 Color and Contrast

All state colors meet WCAG 2.1 AA contrast requirements (4.5:1 for normal text, 3:1 for large text):

| State | Background | Text | Contrast Ratio |
|-------|-----------|------|----------------|
| Locked | `bg-gray-50` (#F9FAFB) | `text-gray-400` (#9CA3AF) | 3.0:1 (large text) + icon reinforcement |
| Unlocked | `bg-white` (#FFFFFF) | `text-gray-900` (#111827) | 17.1:1 |
| Generating | `bg-yellow-50` (#FEFCE8) | `text-yellow-700` (#A16207) | 5.2:1 |
| Completed | `bg-white` (#FFFFFF) | `text-green-600` (#16A34A) | 4.6:1 |
| Failed | `bg-red-50` (#FEF2F2) | `text-red-600` (#DC2626) | 5.1:1 |

**Important:** Locked state text at `text-gray-400` only meets the 3:1 ratio for large text. The lock icon and "Unlock" button provide redundant visual cues beyond color alone, satisfying WCAG 1.4.1 (Use of Color) and WCAG 1.4.11 (Non-text Contrast).

### 10.4 Reduced Motion

```css
@media (prefers-reduced-motion: reduce) {
  /* Disable pulse animations on generating episodes */
  .animate-pulse { animation: none; }

  /* Replace slide transitions with opacity-only */
  .card-transition { transition: opacity 0.15s ease; }

  /* Disable progress bar animations */
  .progress-bar { transition: none; }

  /* Disable credit count animations */
  .credit-count { transition: none; }
}
```

### 10.5 Focus Indicators

All focusable elements use the existing ring pattern:
```css
focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2
```

Custom focus for episode cards:
```css
focus-visible:ring-2 focus-visible:ring-brand-400 focus-visible:ring-offset-2
```

---

## 11. Micro-copy

### 11.1 Button Labels

| Context | Label | Notes |
|---------|-------|-------|
| Locked episode | "Unlock - {N} credits" | Shows cost inline |
| Unlocked episode | "Generate" | No credit cost (already paid) |
| Completed episode | "Play" / "Pause" | Toggle based on state |
| Failed episode | "Retry" | Free retry |
| Bulk unlock | "Unlock All Remaining - {N} credits" | Total cost |
| Bulk generate | "Generate All Unlocked" | No cost |
| Edit mode save | "Save" | |
| Edit mode cancel | "Cancel" | |
| Episode download | (icon only, no label) | Tooltip: "Download episode" |
| Episode expand | (icon only, no label) | Tooltip: "Show details" / "Hide details" |

### 11.2 Status Badge Labels

| State | Badge Text | Color Scheme |
|-------|-----------|-------------|
| Locked | "Locked" | `bg-gray-100 text-gray-600` |
| Unlocked | "Ready to generate" | `bg-blue-100 text-blue-700` |
| Editing | "Editing" | `bg-blue-100 text-blue-700` |
| Queued | "Queued" | `bg-blue-100 text-blue-600` |
| Generating | "Generating..." | `bg-yellow-100 text-yellow-700` |
| Completed | "Ready to play" | `bg-green-100 text-green-700` |
| Failed | "Failed" | `bg-red-100 text-red-700` |

### 11.3 Confirmation Dialog Copy

**Unlock Single:**
- Title: "Unlock Episode {N}?"
- Body: '"{episode_title}"  {duration} min | {quality} quality'
- Credit table: Current balance / Unlock cost / Balance after
- Helper text: "After unlocking, you can edit the title and description before generating."
- Primary CTA: "Unlock for {N} Credits"
- Cancel: "Cancel"

**Unlock All:**
- Title: "Unlock All Remaining Episodes?"
- Body: "This will unlock {N} episodes:" followed by list
- Credit table: same pattern
- Primary CTA: "Unlock {N} Episodes - {total} credits"
- Cancel: "Cancel"

**Generate Single:**
- Title: "Generate Episode {N}?"
- Body: '"{episode_title}"  {duration} min | {quality} quality'
- Helper: "No additional credits will be charged. Generation typically takes 2-5 minutes."
- Warning: "You won't be able to edit the title or description once generation starts."
- Primary CTA: "Generate Episode"
- Cancel: "Cancel"

**Generate All:**
- Title: "Generate All Unlocked Episodes?"
- Body: "{N} episodes will begin generating:" followed by list
- Helper: "No additional credits will be charged. Total generation time: approximately {time}."
- Warning: "Titles and descriptions will be locked during generation."
- Primary CTA: "Generate {N} Episodes"
- Cancel: "Cancel"

### 11.4 Empty and Status Messages

| Context | Message |
|---------|---------|
| No curriculum yet | "No curriculum yet. Click Generate Curriculum above to create a detailed episode outline based on your topic." |
| Curriculum generating | "Your curriculum is being created... This usually takes 30-60 seconds. You can wait here or come back later." |
| All episodes completed | "All episodes are ready! Your entire series is available to play and download." |
| Insufficient credits | "You need {N} more credits to unlock this episode." |
| Unlock success (toast) | "Episode {N} unlocked successfully." |
| Generation started (toast) | "Episode {N} generation started. This typically takes 2-5 minutes." |
| Generation complete (toast) | "Episode {N} is ready to play!" |
| Generation failed (inline) | "Generation failed: {error_message}. You can retry without using additional credits." |
| Edit saved (toast) | "Episode {N} updated." |
| Credit balance low (banner) | "Your credit balance is running low. You have {N} credits remaining." |

### 11.5 Tooltip Text

| Element | Tooltip |
|---------|---------|
| Lock icon on episode | "This episode is locked. Unlock to edit and generate." |
| Edit button | "Edit episode title and description" |
| Generate button | "Start generating audio for this episode" |
| Download button | "Download episode audio" |
| Expand/collapse chevron | "Show episode details" / "Hide episode details" |
| Credit coin icon | "Your remaining credit balance" |
| Progress circle | "{N}% of episodes completed" |

---

## 12. Technical Data Model Changes

### 12.1 New Episode States

The existing `EpisodeStatus` type needs a new value:

```typescript
export type EpisodeStatus =
  | 'locked'      // NEW: Not yet unlocked by user
  | 'pending'     // Unlocked but not yet generated (renamed meaning)
  | 'queued'      // In generation queue
  | 'running'     // Currently generating
  | 'completed'   // Successfully generated
  | 'failed'      // Generation failed
  | 'skipped';    // Manually skipped
```

### 12.2 Episode Unlock Tracking

Each `CourseEpisode` needs additional fields:

```typescript
export interface CourseEpisode {
  // ... existing fields ...

  // NEW: Unlock tracking
  is_locked: boolean;           // Whether the episode is locked
  unlock_cost_credits: number;  // Credit cost to unlock this episode
  unlocked_at?: string;         // ISO timestamp of when unlocked
  credits_charged?: number;     // Actual credits charged for this episode
}
```

### 12.3 New API Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/v1/courses/{courseId}/episodes/{episodeNumber}/unlock` | Unlock a single episode |
| POST | `/v1/courses/{courseId}/unlock-all` | Unlock all remaining locked episodes |
| POST | `/v1/courses/{courseId}/episodes/{episodeNumber}/generate` | Generate a single episode |
| GET | `/v1/courses/{courseId}/credit-estimate` | Get per-episode and total credit estimate |

### 12.4 Course Status Mapping

The `CourseStatus` type remains the same, but the state machine transitions change:

```
draft --> syllabus_generation --> curriculum_ready
                                       |
                          (episodes unlock individually)
                          (episodes generate individually)
                                       |
                                       v
                              generating (any episode generating)
                                       |
                              +--------+--------+
                              v                 v
                          completed          partial
                         (all done)     (some failed)
```

### 12.5 Credit Estimate Response

```typescript
export interface CreditEstimateResponse {
  per_episode_cost: number;          // Credits per episode
  total_locked_episodes: number;     // Number of locked episodes
  total_unlock_cost: number;         // Total credits to unlock all
  user_credits_balance: number;      // Current user balance
  can_unlock_all: boolean;           // Whether user can afford all
  max_unlockable: number;            // How many user can afford
  breakdown: {
    duration_min: number;
    quality_multiplier: number;
    priority_multiplier: number;
  };
}
```

---

## 13. Component Inventory

### 13.1 New Components Required

| Component | File Path | Description |
|-----------|-----------|-------------|
| `EpisodeCard` | `components/EpisodeCard.tsx` | Unified episode card rendering all states |
| `CreditBalanceBar` | `components/CreditBalanceBar.tsx` | Sticky credit display with progress |
| `UnlockConfirmDialog` | `components/UnlockConfirmDialog.tsx` | Credit-aware unlock confirmation |
| `GenerateConfirmDialog` | `components/GenerateConfirmDialog.tsx` | Generation confirmation |
| `CreditBreakdown` | `components/CreditBreakdown.tsx` | Reusable credit before/after table |
| `EpisodeProgressBar` | `components/EpisodeProgressBar.tsx` | Per-episode generation progress |
| `EpisodeSkeleton` | `components/EpisodeSkeleton.tsx` | Skeleton loading for curriculum generation |
| `InsufficientCreditsDialog` | `components/InsufficientCreditsDialog.tsx` | Upgrade/buy credits prompt |

### 13.2 Modified Components

| Component | Changes |
|-----------|---------|
| `app/courses/[courseId]/page.tsx` | Major refactor: remove "Generate All Episodes" CTA, add per-episode unlock/generate flow, add credit balance bar, update episode rendering to use EpisodeCard |
| `shared/schemas.ts` | Add `locked` to `EpisodeStatus`, add new fields to `CourseEpisode`, add new API interfaces |

### 13.3 Existing Components Reused (No Changes)

| Component | Usage |
|-----------|-------|
| `AlertDialog` + family | Confirmation dialogs |
| `Button` | All action buttons |
| `Card`, `CardContent`, `CardHeader`, `CardTitle`, `CardDescription` | Layout containers |
| `Badge` | Status badges |
| `Input` | Edit mode fields |
| `ErrorDialog` | API error handling |
| `ShareCourseModal` | Share functionality |

---

## Appendix A: State Transition Summary

| From | Action | To | Credits Deducted | API Call |
|------|--------|----|-----------------|----------|
| LOCKED | Unlock | UNLOCKED | Yes (per-episode cost) | POST .../unlock |
| UNLOCKED | Edit | EDITING | No | None (client-side) |
| EDITING | Save | UNLOCKED | No | PUT .../curriculum |
| EDITING | Cancel | UNLOCKED | No | None |
| UNLOCKED | Generate | GENERATING | No (already paid) | POST .../generate |
| GENERATING | Complete | COMPLETED | No | None (poll) |
| GENERATING | Fail | FAILED | No | None (poll) |
| FAILED | Retry | GENERATING | No (free retry) | POST .../retry |

## Appendix B: Polling Strategy

| Context | Interval | Max Duration | Endpoint |
|---------|----------|-------------|----------|
| Curriculum generation | 3 seconds | 5 minutes | GET /v1/courses/{courseId} |
| Episode generation | 5 seconds | 15 minutes per episode | GET /v1/courses/{courseId} |
| Multiple episodes generating | 5 seconds | 15 min * episode count | GET /v1/courses/{courseId} |

Polling stops when:
- The target resource reaches a terminal state (completed, failed)
- The max duration is exceeded (show "taking longer than expected" message)
- The user navigates away from the page (cleanup via `useEffect` return)

## Appendix C: Animation Specifications

| Element | Property | Duration | Easing | Trigger |
|---------|----------|----------|--------|---------|
| Card state transition | opacity, transform, border-color | 300ms | ease-out | State change |
| Lock icon to number | opacity, scale | 200ms | ease-in-out | Unlock |
| Credit balance number | number value | 500ms | ease-out | Deduction |
| Progress bar | width | 300ms | ease-out | Progress update |
| Skeleton shimmer | background-position | 1.5s | linear (infinite) | While loading |
| Toast notification | translateY, opacity | 300ms in, 200ms out | ease-out | Trigger/dismiss |
| Generating pulse | opacity | 2s | ease-in-out (infinite) | While generating |
