Summary of what i understood
1. the test can be run through npm run test 
2. to write test we need to ==import **test**== package in every file 
	1. so instead of doing this we can add ==--globals== flag to the test script in package lock so that we don't have to import test in every test file
	2. we can also ==import "it"== which is ==synonym to test== 