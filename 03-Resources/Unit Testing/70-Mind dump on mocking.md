## Where is mocking used?
- Mocking is used where there is need to test our code which integrates third party tools without invoking thirdParty tools
- The best practice is to write mocks in the "__ mocks __ "  folder so it is easier to manage it 
## What is mock?
- it is as the name suggest creating a fake in place of real one
	- we don't want to invoke real function which gives us a side effect like writing to production database or to the file system. so we use mock to simulate how this all would have behaved but without consequence  