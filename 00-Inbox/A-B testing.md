1. **Identify the Problem:** 
		Start with clear Problem
	1. Find where is highest drop-offs are
		example:
		 Landing Page → Sign Up → Onboarding → First Action → Retention
					   10,000         4,000           2,500               800           300
		               (60% drop)  (37% drop)  (68% drop) ← test here
	2. To do the above we need to create an analytics to see drop rate:
		1. Add postHog (Abhisek is doing it)
		2. Add tap-counter (rage clicks)
		3. Track time spent on a particular screen
		4. Error Tracking
		5. Exit Survey: Show popup when user exists from checkout without paying.
2. Form a Hypothesis:
   "*Changing {X} will improve {metric} because of {reason}*"
	1. Example: "Changing the CTA from *'Submit'* to *'Get Started Free'* will improve sign-up rate because it's less intimidating and communicates value. "
3. Define Success Metric:
	Pick a primary metric, the one that decides the winner and pair it along with a Guardrail metric.
	1. **Guardrail metric**: Things whose performance shouldn't degrade. Even if the primary metric improves but the guardrail fails the variant is rejected.
		example: ```Primary: Purchase conversion rate → did they buy? Guardrail: D7 retention → did they come back after buying? Downstream: Course completion rate → did they actually use what they bought?```
		
	2. **Primary metric**: 
		1. Purchase conversion rate 
			→ did they complete payment?
			→ this decides the winner.
		2. Guardrail metrics: D7 retention → did they come back after buying? 
			→ if this drops, variant is rejected even if primary won Course completion rate 
			→ did they actually use what they bought? → ensures we're not attracting wrong users.
		3. North Star: Weekly active study sessions 
			→ number of users completing at least one study session per week 
			→ the "so what?" check — if primary improves but North Star doesn't move, something is wrong.
4. Define/Calculate time to run the test: 
	-    Without touching any code, figure out required sample size.
		- How do we figure sample size?
			1. Following is the example of our funnel: 
				- Users who downloaded the app (through ads): (say) 4000
				- Users who visited the course: (say) 3800 (5% drop)
				- User who fired up payment: 400 (89% drop)
				- User who paid: 40 (90% drop)
				- Calculate base line:
				 $Baseline = 400/3800 = 10.5$
				- Decide MDE (Minimum Detectable Effort):
				- current:  $3800 \text{ users/day} ×10\% = 400 \text{ purchases/day}$
				- After 2%: $3800×12\%=456 \text{ purchase/day}$
				- $Extra purchase/day : 56$
				- $Extra purchase/year: 20,440$
					- At $₹499/course: ₹1,01,99,560 extra revenue/year$
					- Baseline: 10%
					- MDE: 2%
					- Statistical power:80% (default)
					- Significance level: 5% (default)
					- use [Evan Miller's](https://www.evanmiller.org/ab-testing/sample-size.html) calculator 
					- which gives us 3,623 users per variation
					- RunTime 7346/3,800=1.9 days (so round it to 7 days)
5. Build the variant: 
	1. make one change only
		-  The "Credit" Problem
			- ==If you change two things at once, you can't give "credit" to the winner.
			- Imagine you're coaching a football team. You bring in a new Striker and a new  Goalie at the same time. The team starts winning.
				- Was it the Striker scoring more goals?
				- Was it the Goalie blocking more shots?
				- If you don't know, you won't know how to improve the team next time.
			
		- The "Canceling Out" Effect
			This is the most dangerous part of changing multiple things. Sometimes, one change is great and the other is terrible, and they cancel each other out.
			Change A (Color): Users hate it (-5\% sales).
			Change B (Copy):Users love it (+5\% sales).
			Total Result: 0\% change.
			You look at the data and think, "Well, that was a waste of time, nothing happened." In reality, you found a great new headline, but you killed it by pairing it with an ugly color. You just threw away a +5\% win because you didn't test them separately.
			
		- The "Scientific Method" Approach
			- For your app specifically, you want to build a "Playbook" of what works for your users.
			- Test 1: Change the Color. Result: Green is better than Red. (Now you keep Green).
		    - Test 2: Change the Text. Result: "Start Learning" is better than "Buy Now." (Now you use that).
		    - By doing it one by one, you are building a "stack" of confirmed wins.
		    - Rule of Thumb: 
			    - *If you want to test a whole new "look and feel," call it a "Layout Test" and change the whole screen. But if you are trying to optimize, change one thing at a time so you can be 100\% sure why the needle moved.*
		- A: the control group
		- B: variant
6. Set Up the Split
	1. You add a feature flag in your backend:
		```js
		// Assign variant based on user ID - consistent across sessions
const variant = userId % 2 === 0 ? 'control' : 'variant'

// Fire event so PostHog knows which variant this user saw
posthog.capture('experiment_assigned', {
  experiment: 'buy_button_position',
  variant: variant
})
		```
7. Run the Test:
	 - You set a reminder for **24 days later**
	- You resist the urge to check results on day 5 even though B looks like it's winning
	- You don't make any other changes to the course detail screen during this period
8. Sanity Check Before Reading:
	1. Expected: 2,400 per variant 
	2. Actual: 
		1. A = 2,380, 
		2. B = 2,420 
		3. ✓ close enough If you saw: A = 2,400, B = 1,800 ✗ something broke randomization is invalid don't trust any results.
	3. This is called **Sample Ratio Mismatch**. If it's off, investigate before proceeding — your feature flag logic likely has a bug.
9. Read the Results:
	1. PostHog shows:
		1. Control (A):  4.0% conversion  (96 purchases / 2,400 users)
		2. Variant (B):  5.8% conversion  (139 purchases / 2,400 users)
		3. Absolute lift:   +1.8%
		4. Relative lift:   +45%
		5. p-value:         0.01   ✓ statistically significant (below 0.05)
	2. What p-value means here: There's only a 1% chance this difference is random noise. You didn't just get lucky — the difference is real.
	3. Now check guardrails:
		1. D7 retention: A = 38%,  B = 37%   ✓ no meaningful regression
		2. Course completion: A = 22%,  B = 21%   ✓ acceptable
		3. North Star
			1. (weekly study sessions): moved up slightly  ✓
		4. Primary metric improved. Nothing regressed. North Star moved.
10. Make the Decision:
	1. Primary metric improved?  +1.8% conversion 
		- Statistically significant?  p = 0.01
		-  Practically significant?  ₹18L+ annual revenue
		-  impact Guardrails held?  retention and completion fine
		-  North Star moved?  weekly sessions up slightly
11. Document Everything:
	1. Problem: 480 users/day saw course detail but didn't buy
	2. Evidence found: Exit surveys → 42% unsure about quality Session replays → users never scrolled to free preview Data → users who saw free preview converted at 3x rate.
	3. Hypothesis: Moving buy button below free preview builds trust first.
	4. Result: +1.8% conversion lift No guardrail regression North Star moved up.
	5. Why it worked: Reordering the screen forced users to see free content before the buy decision — directly addressed the trust gap.
	6. Next to test: Can we improve the free preview content itself to further increase trust and push conversion higher?


- Note: Below is how we should formulate the mental model
	- WHERE is the problem? 
		- Funnel: 80% drop on course detail screen
	- WHY is it happening? 
		- Replays + surveys + data: users don't see free preview, unsure about quality before price is shown
	- WHAT will we change? 
		- Move buy button below free preview (one change only)
	- HOW will we measure success? 
		- Primary: conversion rate 
		- Guardrails: retention + completion (Goodhart protection)
		- North Star: weekly study sessions (real value check)
	- HOW LONG do we run it? 
		- Baseline 4%, MDE 1%, 80% power 
		- Calculator: 2,400 users per variant 
		- Traffic: 200/day → 24 days.
	- DID IT WORK? 
		- +1.8% lift, p = 0.01, guardrails held, North Star up 
		- Ship it
	- WHAT DID WE LEARN? 
		-  Document it 
		- informs next hypothesis