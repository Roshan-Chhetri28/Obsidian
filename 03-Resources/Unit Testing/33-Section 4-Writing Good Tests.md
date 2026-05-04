##  33 Module introduction
- What to & Not To Test
	- What Should be & Shouldn't be tested
	- Good test are short and concise
	- Testing & Code Improvements work together Iteratively\

	<hr>
##  34 what to test and what not to 
- don't test things we can't change like third party module
	- example is the query selector
		- like if the query selector is returning the right
	- or the node module which is we cannot change
	- we can test: if there is form selector on UI 
	- don't test backend code implicitly in frontend test it in backend it self
	- things to test in frontend
		- reaction to different responses & errors

<hr> 

##  35 Writing Good Tests
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
- Keep your number of exceptions  (assertion) low
## 36 Only test one Thing

- A test should only test one feature
	- Example: 
		- say you have a function that does two things (this should is so wrong to do) validate input or transform it.
		- We don't want to put this things in same test; The test should only check either validation or transformation
			- in this too we only should validate if a string for being empty or having enough characters in different tests
		- Make it as granular  as possible

<hr>

##  37 Splitting functions for Easier testing & Better Code

- There is not much to write but we can see it in following example (we have this functionality every where in our code so making it granular will help all of us out):
- this is my own code: 
	- see generateInvoicePDF() in invoicePage.jsx line 180- 570
	- the generateInvoicePDF() can be written as following
```js
const generateInvoicePDF = async (invoice) => {

if (!invoice) {

console.error('generateInvoicePDF: invoice is undefined or null');

dispatch(setAlert('Unable to generate PDF: Invoice data is missing', 'error'));

return;

}

  


// the fecth Billing address can be written as the follwing whcih will make it easier to write test on
const userBillingAddress = await fetchUserBillingAddress()

// the code ranging from 204 - 213 can be written as following:
 const structure = definePDFStructure()

// the code ranging from 215 - 251 can be written as following
const img = importImageFromLocal()

const logoPos=setPosition()

and so on
```
