## Testing asynchronous code
- we can't use callBack function to test async code
```js
it('should test for callbacks', (done)=>{

	const testuserEmai; = 'test@test.com'
	
	generateToken(testUserEmai, (err, token) =>{
	try{
		expect(token).toBeDefined();	
		//we can add done() to wait for promise to returnand the callback to be run
		expect(token).toBe(2) // to catch error we are using try catch
		done()
	}catch(err){
		done(err)
	}
	
	});

});
```

- and to check for the error we need to use try catch and pass catched error in done
<hr>

## Testing asynchronous code with promises
- expect supports promises out-of-box so we can wrap the promise returning function in expect
	- we can use rejects or resolves before toBe() and don't forget to use return expect..
```js
  
  //testing for promise 
  it('should test for promises', ()=>{
	  const testuserEmai; = 'test@test.com'
	
	return  expect(generateTokenPromise(testUserEmai)).resolves.toBeDefined();

  })
  
```

- Alternative Approach
	- this feels beautiful
```js
  
  //testing for promise 
  it('should test for promises', async()=>{
	const testuserEmai; = 'test@test.com'
	const token = await generateTokenPromise(testUserEmai)
	expect(token).toBeDefined();
  })
  
```
