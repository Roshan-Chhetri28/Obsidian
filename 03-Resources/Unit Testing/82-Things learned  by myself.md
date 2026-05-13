we can use expect.any(dataType) to replace a variable value

```js
// Assert

expect(JSON.parse(response.text)).toMatchObject({

	id: expect.any(Number),
	name: expect.any(String),
	user_id: 1,
	course_id: 2,
	date: expect.any(String),
	hide: 0,
	trial: 0,
	priority: 0,
	updated_at: expect.any(String),
	deleted_at: null,
})
```