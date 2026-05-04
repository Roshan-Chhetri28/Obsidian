### A/B Testing: Our Complete Playbook

---

## 1. Identify the Problem

We start with a clear problem — we never test randomly.

### Step 1: Find Where the Highest Drop-offs Are

We map our user journey as a funnel and find where the biggest leaks are:

```
Landing Page → Sign Up → Onboarding → First Action → Retention
  10,000        4,000      2,500          800           300
             (60% drop) (37% drop)   (68% drop) ← we test here
```

The biggest drop = highest leverage point. Fixing here affects the most users.

### Step 2: Set Up Analytics to See Drop Rates

To find these drop-offs we need instrumentation in place:

1. **PostHog** — funnel analysis and session replay (Abhishek is handling this)
2. **Rage tap counter** — detect when users tap the same element rapidly out of frustration
3. **Time spent tracking** — high time on a screen before exit = confusion
4. **Error tracking** — a lot of drop-offs are silent technical failures, not UX problems
5. **Exit survey** — show a popup when a user leaves checkout without paying, ask them directly what stopped them

---

## 2. Form a Hypothesis

Once we know WHERE and WHY, we commit to a hypothesis upfront:

> _"Changing {X} will improve {metric} because of {reason}"_

**Example:**

> _"Changing the CTA from 'Submit' to 'Get Started Free' will improve sign-up rate because it's less intimidating and communicates value."_

The "because" must come from what we found in our investigation — not from a gut feeling.

---

## 3. Define Our Success Metrics

We pick a primary metric that decides the winner — but we never use it alone. We pair it with guardrail metrics to protect against Goodhart's Law (optimising a number until it stops reflecting reality).

**Primary metric:**

```
Purchase conversion rate
→ did they complete payment?
→ this decides the winner
```

**Guardrail metrics:**

```
D7 retention
→ did they come back after buying?
→ if this drops, variant is rejected even if primary won

Course completion rate
→ did they actually use what they bought?
→ ensures we're not attracting the wrong users
```

**North Star:**

```
Weekly active study sessions
→ number of users completing at least one study session per week
→ the "so what?" check
→ if primary improves but North Star doesn't move, something is wrong
```

The logic chain we always follow:

```
More purchases  →  so what?  →  are they studying?
More studying   →  so what?  →  are they learning and coming back?
                                        ↓
                                 North Star moves
                                 = real value delivered
```

---

## 4. Calculate How Long to Run the Test

Before touching any code, we figure out our required sample size.

### Our Funnel (Real Numbers)

```
Users who downloaded the app (via ads):   4,000
Users who visited the course:             3,800  (5% drop)
Users who fired up payment:                 400  (89% drop)  ← biggest leak
Users who paid:                              40  (90% drop)
```

### Calculate Baseline Rate

```
Baseline = 400 / 3,800 = 10.5% ≈ 10%
```

This is our starting point — the number we're trying to beat.

### Decide MDE (Minimum Detectable Effect)

We first think about real business impact:

```
Current:         3,800 users/day × 10% = 400 purchases/day
After 2% lift:   3,800 users/day × 12% = 456 purchases/day

Extra purchases/day:    56
Extra purchases/year:   20,440
At ₹499/course:         ₹1,01,99,560 extra revenue/year
```

That's worth detecting → we set **MDE = 2%**

### Plug Into Calculator

We use [Evan Miller's calculator](https://www.evanmiller.org/ab-testing/sample-size.html):

```
Baseline rate:      10%
MDE:                2%
Statistical power:  80%   (default — means 80% chance of catching a real improvement)
Significance level: 5%    (default — means 5% chance of a false positive)

Output: 3,623 users per variant
```

### Convert to Runtime

```
Users needed per variant:    3,623
Total users needed (A + B):  7,246
Daily traffic on screen:     3,800

Runtime = 7,246 / 3,800 = 1.9 days → floor rule → 7 days minimum
```

We always run for at least 7 days regardless of what the math says — to capture the full weekday vs weekend behaviour cycle.

---

## 5. Build the Variant

We make **one change only.** Here's why this matters:

### The Credit Problem

If we change two things at once, we can't give "credit" to the winner.

Imagine we're coaching a football team. We bring in a new Striker and a new Goalie at the same time. The team starts winning.

- Was it the Striker scoring more goals?
- Was it the Goalie blocking more shots?

We don't know — and we won't know how to improve the team next time.

### The Cancelling Out Effect

This is the most dangerous part. Sometimes one change is great and another is terrible and they cancel each other out:

```
Change A (Color): Users hate it → -5% sales
Change B (Copy):  Users love it → +5% sales
Total result:     0% change
```

We look at the data and think nothing happened. In reality we found a great headline but killed it by pairing it with a bad colour. We just threw away a +5% win.

### The Scientific Method Approach

We want to build a **playbook** of what works for our users:

```
Test 1: Change the colour  → Green beats Red     ✓ keep Green
Test 2: Change the text    → "Start Learning" beats "Buy Now"  ✓ keep that
```

By doing it one by one we build a stack of confirmed wins.

**Rule of thumb:**

> _If we want to test a whole new look and feel, we call it a "Layout Test" and change the whole screen. But if we're optimising, we change one thing at a time so we can be 100% sure why the needle moved._

```
A = Control (current version, untouched)
B = Variant (our one change)
```

---

## 6. Set Up the Split

We add a feature flag in our backend:

```javascript
// Assign variant based on user ID — consistent across sessions
const variant = userId % 2 === 0 ? 'control' : 'variant'

// Fire event so PostHog knows which variant this user saw
posthog.capture('experiment_assigned', {
  experiment: 'buy_button_position',
  variant: variant
})
```

We always assign by **user ID, never by session** — so the same user always sees the same variant across multiple sessions. Session-based assignment creates inconsistent experiences and corrupts results.

For risky changes we start 90/10 instead of 50/50 — exposing fewer users to the variant until we're confident it's not harmful.

---

## 7. Run the Test — We Don't Touch It

- We set a reminder for day 7 (or whatever our calculated runtime is)
- We resist the urge to check results early even if B looks like it's winning — early data is just noise
- We don't make any other changes to the same screen during the test — it contaminates results

```
Day 3:  B looks like it's winning at 7% vs 4%  ← noise, don't stop
Day 7:  Real signal emerges from enough data    ← now we read it
```

---

## 8. Sanity Check Before Reading Results

Before we look at any conversion numbers, we verify our split is actually 50/50:

```
Expected per variant:  3,623

Actual:
  A = 3,580  ✓
  B = 3,666  ✓  close enough — randomization is fine

If we saw:
  A = 3,623  ✓
  B = 1,800  ✗  something broke — don't trust results
```

This is called **Sample Ratio Mismatch (SRM)**. If the split is off, our feature flag logic has a bug — the groups are no longer comparable and any result we see is invalid. We fix the bug and restart.

Rule of thumb — if the ratio is outside **0.9 to 1.1**, we investigate before trusting anything.

---

## 9. Read the Results

PostHog shows us:

```
Control (A):  4.0% conversion  (96 purchases / 2,400 users)
Variant (B):  5.8% conversion  (139 purchases / 2,400 users)

Absolute lift:   +1.8%
Relative lift:   +45%
p-value:         0.01  ✓ statistically significant (below 0.05)
```

**What p-value means:** There's only a 1% chance this difference is random noise. We didn't just get lucky — the difference is real.

**Now we check our guardrails:**

```
D7 retention:            A = 38%,  B = 37%  ✓ no meaningful regression
Course completion:       A = 22%,  B = 21%  ✓ acceptable
North Star
(weekly study sessions): moved up slightly  ✓
```

Primary metric improved. Nothing regressed. North Star moved. This is a genuine win — not a Goodhart's Law trap.

---

## 10. Make the Decision

```
Primary metric improved?     ✅  +1.8% conversion
Statistically significant?   ✅  p = 0.01
Practically significant?     ✅  ₹18L+ annual revenue impact
Guardrails held?             ✅  retention and completion fine
North Star moved?            ✅  weekly sessions up slightly
```

**Decision: We ship the variant.**

---

## 11. Document Everything

This is the step we must never skip. It becomes our team's institutional memory.

```
Problem:
  480 users/day saw course detail but didn't buy

Evidence found:
  Exit surveys  → 42% unsure about quality
  Session replays → users never scrolled to free preview
  Data → users who saw free preview converted at 3x rate

Hypothesis:
  Moving buy button below free preview builds trust first

Result:
  +1.8% conversion lift
  No guardrail regression
  North Star moved up

Why it worked:
  Reordering the screen forced users to see free content
  before the buy decision — directly addressed the trust gap

Next to test:
  Can we improve the free preview content itself to
  further increase trust and push conversion higher?
```

---

## Our Mental Model

Every time we run a test, we work through this framework:

```
WHERE is the problem?
  → Funnel: 80% drop on course detail screen

WHY is it happening?
  → Replays + surveys + data: users don't see free preview,
    unsure about quality before price is shown

WHAT will we change?
  → Move buy button below free preview (one change only)

HOW will we measure success?
  → Primary: conversion rate
  → Guardrails: retention + completion (Goodhart protection)
  → North Star: weekly study sessions (real value check)

HOW LONG do we run it?
  → Baseline 10%, MDE 2%, 80% power
  → Calculator: 3,623 users per variant
  → Traffic: 3,800/day → 7 days

DID IT WORK?
  → +1.8% lift, p = 0.01, guardrails held, North Star up
  → Ship it

WHAT DID WE LEARN?
  → Document it → informs our next hypothesis
```
