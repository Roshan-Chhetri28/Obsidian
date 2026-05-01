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
		5. Exit Survey: Show popup when user exists from checkout without paying
2. Form a Hypothesis:
   "Changing {X} will improve {metric} because of {reason}"
	1. Example: "Changing the CTA from *'Submit'* to *'Get Started Free'* will improve sign-up rate because it's less intimidating and communicates value. "
3. Define Success Metric:
	Pick a primary metric, the one that decides the winner and pair it along with a Guardrail metric.
	1. Guardrail metric: Things whose performance shouldn't degrade. Even if the primary metric improves but the guardrail fails the variant is rejected.
		example: ```Primary: Purchase conversion rate → did they buy? Guardrail: D7 retention → did they come back after buying? Downstream: Course completion rate → did they actually use what they bought?```
	If conversion goes up but retention crashes, you haven't won — you've just made it easier for the wrong users to buy.
4. Define/Calculate time to run the test: 
	   Without touching any code, figure out required sample size
	   