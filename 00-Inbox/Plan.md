## 1. Suppose a user downloads and then visits the landing screen - we need to track him from being anonymous to registering.

- Create some kind of token when app initializes first time and keep it stored in local cache and when user logs in send this along with login event with current user_id

- So post-hog gives every anon user a distinct_id the moment SDK initializes and then if i use posthog.identify() on that distict_id with user_id as parameter then posthog tracks all the event for the distinct_id before signup.

## 2. Onboarding page 2 - others where user can specify which course he wants — make analysis

- What we can do is instead of doing this all one by one we can create onboarding funnel and we can see where users are dropping off 


## 3. How will we analyse user's choice user selects a course or courses and the Educator he selects

- we can see distribution graph to see which is the most used/clicked button
- 