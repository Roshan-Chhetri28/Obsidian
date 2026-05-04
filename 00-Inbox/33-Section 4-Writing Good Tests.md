## Module introduction
- What to & Not To Test
	- What Should be & Shouldn't be tested
	- Good test are short and concise
	- Testing & Code Improvements work together Iteratively\

	<hr>
## what to test and what not to 
- don't test things we can't change like third party module
	- example is the query selector
		- like if the query selector is returning the right
	- or the node module which is we cannot change
	- we can test: if there is form selector on UI 
	- don't test backend code implicitly in frontend test it in backend it self
	- things to test in frontend
		- reaction to different responses & errors

<hr> 

## Writing Good Tests
- use AAA pattern
- Test only one thing/ behaviour 
- Focus on essence of the thing we are testing
	- example:
```js
it('should summarize all numbers values in an arrary')
 const numbers = [1,2] **
 
 const result = add(numbers);
 
 expect(result).toBe(expectedResult);
```
-  ** In above code we can just check with two numbers without complicating the test
- Keep your number of exceptions  (assertion) low\
