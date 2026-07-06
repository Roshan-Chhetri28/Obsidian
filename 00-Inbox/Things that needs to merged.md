(Sorted by Date Created)

| Branch                   | Repos                                    | Description                                                                                                  | Needs Migration                                    | Status                                      |
| ------------------------ | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------- | ------------------------------------------- |
| feat/SelfStudy           | Backend, Frontend, Webhook, React Native | Added Self Study tier which provides everything else except Videos                                           | Yes: database/selfStudy.js                         | On Hold                                     |
| feat/outRankNotification | Backend, Reactnative                     | push notification for Leaderboard when ever use gets out ranked 1/day                                        | No                                                 | On Hold                                     |
| fix/videoDeepLink        | Backend                                  | The Apps route changed which caused the deeplink to go obsolete and didnt open the video tab                 | No                                                 | Merged, Needs to be checked in Prod it self |
| fix/whatsappPush         | Backend                                  | The Admin was getting notification for his own reply                                                         | No                                                 |                                             |
| feat/videoDecoupling     | backend                                  | Soft delete Unnecessary notes and heading and Hading Removed while adding video                              | node database/migrations/removeUnnecessaryNotes.js | Merged, checked                             |
| feat/emiPushNotification | Backend, ReactNative, Razorpay           | push notification on 7, 3, 1, 0, day before course expiry date and 1, 3, 7 days after course expires         | Yes: node scripts/backfillEmiReminders.js          |                                             |
| Prod                     | chat                                     | Production CI CD                                                                                             | No                                                 |                                             |
| feat/subscribers-chat    | chat                                     | Introduced a new service, friendshipGate, to check if two users are friends before allowing direct messages. | No                                                 |                                             |
Merge and check the following
- [x] 1. fix/videoDeepLink  
- [x] 2. feat/videoDecoupling
- [ ] 3. feat/emiPushNotification
- [ ] 4. fix/whatsappPush
- [ ] 5. Prod