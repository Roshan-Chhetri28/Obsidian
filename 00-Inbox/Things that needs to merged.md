(Sorted by Date Created)

| Branch                   | Repos                                    | Description                                                                                                  | Needs Migration                                    |
| ------------------------ | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------- |
| feat/SelfStudy           | Backend, Frontend, Webhook, React Native | Added Self Study tier which provides everything else except Videos                                           | Yes: database/selfStudy.js                         |
| feat/outRankNotification | Backend, Reactnative                     | push notification for Leaderboard when ever use gets out ranked 1/day                                        | No                                                 |
| fix/videoDeepLink        | Backend                                  | The Apps route changed which caused the deeplink to go obsolete and didnt open the video tab                 | No                                                 |
| fix/whatsappPush         | Backend                                  | The Admin was getting notification for his own reply                                                         | No                                                 |
| feat/videoDecoupling     | backend                                  | Soft delete Unnecessary notes and heading                                                                    | node database/migrations/removeUnnecessaryNotes.js |
| feat/emiPushNotification | Backend, ReactNative, Razorpay           | push notification on 7, 3, 1, 0, day before course expiry date and 1, 3, 7 days after course expires         | Yes: node scripts/backfillEmiReminders.js          |
| Prod                     | chat                                     | Production CI CD                                                                                             | No                                                 |
| feat/subscribers-chat    | chat                                     | Introduced a new service, friendshipGate, to check if two users are friends before allowing direct messages. | No                                                 |
