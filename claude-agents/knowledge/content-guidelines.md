# KitesForU Content Guidelines
<!-- last_verified: 2026-02-07 -->

## Brand Voice
- **Inspiring**: "Give Your Ideas Wings to Fly", "Your kite awaits"
- **Encouraging**: "What Can You Imagine?"
- **Action-oriented**: "Start Creating", "Build a Series", "Prepare Now"
- **Benefit-focused**: Emphasize user outcomes over features
- **Time-conscious**: "2-3 minutes", "minutes not hours", "Ready in Minutes"

## Terminology Standards
| Concept | Use | Don't Use |
|---------|-----|-----------|
| Audio content | "audio series", "podcast" | "recording", "file" |
| Course | "series", "course" | "playlist", "collection" |
| Episode | "episode" | "chapter", "segment", "part" |
| Generation | "create", "generate" | "produce", "make", "build" |
| AI process | "AI-powered", "intelligent" | "machine learning", "neural network" |
| User | "creator" | "customer", "subscriber" |
| Quality | "studio quality", "professional" | "high quality", "premium" |

## Microcopy Standards

### Form Labels
- Short, descriptive: "Topic", "Duration", "Style"
- Helper text in lighter color below field
- Optional fields marked with "(Optional)" not asterisks

### Error Messages
- Empathetic tone: "We're sorry" not "Error occurred"
- Actionable: Always include a next step ("Try again", "Contact support")
- Technical details available but hidden (expandable)
- Include job/error IDs for support reference

### Progress Messages
- Stage-based: "Gathering information about your topic" not "Processing"
- Time estimates: "This usually takes 2-3 minutes"
- Honest: Don't say "almost done" if it's not

### Success Messages
- Celebratory but not over the top
- Include next action: "Create Another" or "Back to Home"
- Highlight the value: What was just created and why it's special

### Empty States
- Encouraging: "Your courses will appear here" not "No courses found"
- Include CTA: "Create your first course ->"

## Content Quality Standards (for AI-generated content)

### From quality_guidelines.yaml:
- Information quality: Specific facts, statistics, examples
- Conversational flow: Each turn builds on previous
- Research quality: Prioritize official sources, cross-reference
- Anti-repetition: Each result must provide unique information

### From density_guidelines.yaml:
- 5-Second Rule: Every 5 seconds must deliver fact, insight, example, or meaningful question
- Replace vague with specific:
  - "That's interesting" -> follow-up question or related fact
  - "Many people think" -> specific number or source
  - "It's growing" -> "Growing 40% year over year"
  - "Recently" -> actual date ("January 2025")

## SEO Metadata
- Homepage title: "Kitesforu - Give Your Ideas Wings to Fly"
- Meta description: "Transform any topic into captivating audio content with AI"

## Accessibility Standards (Target)
- All interactive elements: aria-labels
- Images: alt text
- Forms: associated labels
- Color contrast: WCAG AA minimum
- Keyboard navigation: full support
- Screen reader: semantic HTML

## Areas Needing Content
- Email templates (verification, subscription, notifications)
- In-app tooltips and help text
- Knowledge base / FAQ
- Terms of Service, Privacy Policy
- Payment error messages
- i18n/localization (noted "44+ Languages" in marketing but no i18n config)
- Invite/sharing templates
