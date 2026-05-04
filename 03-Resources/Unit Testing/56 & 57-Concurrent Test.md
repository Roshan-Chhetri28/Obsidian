## Concurrent Test
- We can add .concurrent after it function to run test concurrently
```js
it.concurrent('description', ()=>{
	test code
})
```
- Alternatively we can directly add .concurrent to the describe function to run all the test in the suite comcurrent;y
- For small amount of tests this wont create much of a difference but for large tests it can create huge difference in test

<hr>


## Concurrency & Default Behavior

- Even when not adding the `.concurrent` property / annotation, tests that are stored in different files are executed concurrently (i.e., in parallel). This is done by both Vitest and Jest - ensuring that your tests run in a short amount of time.
- With `.concurrent` you can enforce this behavior also inside the individual files (i.e., tests that live in one and the same file are executed concurrently).
- Concurrent execution can reduce the amount of time your tests need to execute. A downside of concurrent execution is, that tests that perform clashing (global) state manipulations may interfere with each other.