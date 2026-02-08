# Quality Baselines — Example Good vs Bad Outputs
<!-- last_verified: 2026-02-07 -->

## Curriculum Generation Examples

### BAD Curriculum (Score 3/10)
```json
{
  "summary": "A comprehensive interview prep course for software engineers",
  "episodes": [
    {"title": "Introduction to Technical Interviews", "description": "Overview of what to expect"},
    {"title": "Data Structures and Algorithms", "description": "Common DS&A topics"},
    {"title": "System Design Basics", "description": "How to approach system design"},
    {"title": "Behavioral Interview Tips", "description": "STAR method and examples"},
    {"title": "Mock Interview Practice", "description": "Full practice session"}
  ]
}
```
Problems: Generic titles, no candidate reference, no company reference, no gap analysis, cookie-cutter progression

### GOOD Curriculum (Score 9/10)
```json
{
  "summary": "Targeted prep for Sarah's transition from AWS-focused backend at Stripe to Google's L5 Full-Stack role, focusing on frontend gaps and Google-specific system design expectations",
  "gap_analysis": {
    "strengths": ["5 years distributed systems at Stripe", "Strong Python/Go", "Payment system expertise"],
    "gaps": ["Limited React/frontend experience vs full-stack requirement", "No exposure to Google's infrastructure (Spanner, Bigtable)", "System design at Google scale (billions of users vs millions)"],
    "unique_advantages": ["Payment domain expertise rare at Google Cloud", "Incident response leadership experience"]
  },
  "episodes": [
    {
      "title": "Bridging Stripe's Payment APIs to Google's Cloud Infrastructure Patterns",
      "episode_type": "technical_deep_dive",
      "description": "Leverage Sarah's deep payment API knowledge to understand Google Cloud's analogous patterns",
      "gaps_addressed": ["Google infrastructure familiarity"],
      "practice_questions": ["Design a payment processing system that handles 10x Stripe's volume using Google Cloud services"]
    }
  ]
}
```

## Episode Script Examples

### BAD Script Opening
"Welcome to our interview prep podcast! Today we're going to talk about technical interviews. Technical interviews can be challenging, but with the right preparation, you can succeed. Let's dive in!"
Problems: Generic, no candidate reference, filler phrases, no hook

### GOOD Script Opening
"Sarah, you've spent 5 years building Stripe's payment infrastructure — that's a massive advantage walking into Google. But here's what most candidates from fintech miss: Google doesn't just want to know you can build reliable systems. They want to see you think at 100x scale. Today, we're going to take your Stripe Checkout API experience and show you exactly how to frame it for Google's L5 system design bar."
Strengths: Candidate-specific, company-specific, sets up the gap, creates urgency
