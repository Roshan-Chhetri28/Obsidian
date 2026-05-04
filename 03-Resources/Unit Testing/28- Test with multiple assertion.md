## Test with multiple assertions
- we can use multiple expect statement to one of same test
- if we add multiple expectations all of the test should pass else it throws error
**Note**: 
- Technically the expect() does not return true or false. Instead it throws an error. 
- if the expectations is not met the test runner treats thrown errors as failed test that do not throw as passed.
Example where multiple examples makes sense:
```js 
it('should return NaN for non-transformable vlaues', ()=>{
	const nonTransformableValue = 'invalid';
	const nonTransformableValue2 = {};
	
	
	const resultForString = transformToNUmber(nonTransformableValue)
	const resultForObject = transformToNUmber(nonTransformableValue2)
	
	expect(resultForString).toBeNaN();
	expect(resultForObject).toBeNaN();
})
```