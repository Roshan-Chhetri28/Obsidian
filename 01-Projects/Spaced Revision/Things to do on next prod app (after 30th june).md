
Run migrations:
- ~~node database/bookmarks.js~~
- ~~node database/migrations/addVideoTopicId.js~~
- node database/selfStudy.js
-  node database/migrations/removeUnnecessaryNotes.js 
- node scripts/backfillEmiReminders.js (add .env as well)
The posthog events has been changed so fix the dashboards and backfill accordingly
