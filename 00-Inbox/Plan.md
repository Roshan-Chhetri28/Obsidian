## 1. Suppose a user downloads and then visits the landing screen - we need to track him from being annonomous to registering.

- Create some kind of token when app initializes first time and keep it stored in local cache and when user logs in send this along with login event with current user_id

- So posthog gives every anon user a distinct_id the moment SDk initializes and then if i use posthog.identify() on that distict_id with user_id as parameter then posthig tracks all the event for the distinct_id before signup.

## 2. Onboarding page 2 - others where user can specify which course he wants make analysis

- 


## 3. How will we analyse user's choice user selects a course or courses and the Educator he selects
