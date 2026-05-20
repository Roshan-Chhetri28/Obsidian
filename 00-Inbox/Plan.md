## 1. Suppose a user downloads and then visits the landing screen - we need to track him from being anonymous to registering.

- Create some kind of token when app initializes first time and keep it stored in local cache and when user logs in send this along with login event with current user_id

- So post-hog gives every anon user a distinct_id the moment SDK initializes and then if i use `posthog.identify()` on that `distict_id` with `user_id` as parameter then posthog tracks all the event for the distinct_id before signup.

## 2. Onboarding page 2 - others where user can specify which course he wants — make analysis

- What we can do is instead of doing this all one by one we can create onboarding funnel and we can see where users are dropping off 


## 3. How will we analyse user's choice user selects a course or courses and the Educator he selects

- we can see distribution graph to see which is the most used/clicked button
- along with time spent on that page to see if users are reading the options or just clicking randomly

## 4. Once we take the user to the explore page after making his choice we need to track what he does next

- we can capture what page users visit and how much time do they spent on the page
- same in flashcard component after users turns the flash card count how much time it takes user to click the action buttons.
  
  
## 5. Creating Whole SignUp -> Study Funnel
- I find post signup to -> study (flashcard) to be most sensitive for long-term user retention.
- For new users we should track how much time they stay in a particular page the reasons are as follow:
	1. If the user stay in a page more than they previously were we can deduce two things 
		1. if it is study screen then we are correctly capturing their attention
		2. if it is any other page that shouldn't take user attention and should be easy to navigate then that page is confusing user
- We can also monitor heat map for pages in this funnel:
	1. we can track rage clicks
	2. Dead buttons
	3. we can also monitor which button/feature is being used
	4. the heat map in study flashcard will tell us how many people click perfect\forgot\difficult\easy
