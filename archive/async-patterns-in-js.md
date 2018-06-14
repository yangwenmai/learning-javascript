# Asynchronous Patterns in JavaScript (Part 1)

Dealing with asynchronous logic is one of the most challenging parts in programming. In JavaScript, we have different ways of coping with async flow control. In this two-part series, I'll take a close look into different patterns of handling async logic. By explaining why these patterns are introduced and what problems they aim to solve, this series provides a higher level of abstraction on this topic. This is the first part, I'll explore callbacks, thunks and Promises.

## Old familiar callbacks

If you are like me, who started to learn JavaScript later than 2016, by which time the ES6 had already been introduced into JavaScript, you've probably never had to use callbacks to solve async problems. Learning and understanding this pattern seems unnecessary if we're never going to use it. However, by understanding this pattern, we'll gain at least these two benefits:

    1. By knowing how JavaScript programmers struggled to tackle complex async problems back in these dark days when Promises don't exist, we'll get a deeper understanding of why we need better solutions for handling async issues.
    2. We'll be spared from making these mistakes again. If you're not careful, you'll probably get into the same trouble that callbacks introduces.

### Callback hell

Say we have 3 tasks: task1, task2, task3. We want them to be executed in sequence, with each one waiting for 2000 ms before being executed. With the help of callbacks, we can achieve this easily:

```javascript
setTimeout(function() {
  task1()
  setTimeout(function() {
    task2()
    setTimeout(function() {
      task3()
    }, 2000)
  }, 2000)
}, 2000)
```

This piece of code is awful to look at and it hurts our brains to try to keep track of each step. This style is famously named callback hell, a named well deserved.

The main disadvantages of callbacks have less to do with readablity. The major issues lie much deeper and cause much bigger problems.

### Inversion of control

As a software design technique, IOC (Inversion of Control) has some valid use cases. However, sometimes it may cause trouble. By allowing callbacks to be executed without our control, we risk our program of running into problems we'd never anticipate. To illustrate what I mean, consider this scenario:

You're building an eCommerce site and the payment process of your site embeds a tracking service from a third party. You pass the payment method to a function that's provided by the third party service, letting them to call the payment method at the right time. Here's the pseudo code:

```javascript
trackCheckout(purchaseInfo, function finish() {
  chargeCreditCard(purchaseInfo)
  showThankYouPage()
})
```

Now you have a problem. What if the third party service calls your `chargeCreditCard()` method more than once? You have no way of preventing them to do that! If for some crazy reasons the tracking service calls the payment method multiple times, you'll have to deal with angry customers and a huge PR crisis!

There's a trust issue about Inversion of Control, we can't trust the callers of our callbacks to call the callbacks as we expect.

### Hard to reason about

When we write code like this:

```javascript
console.log("execute this first part!")

setTimeout(function() {
  console.log("execute this second part!")
}, 1000)
```

We expect the code to work like this:

```javascript
console.log("execute this first part!")
blockTheProgram(1000)
console.log("execute this second part!")
```

What really happens is like this:

```javascript
console.log("execute this first part!")
// Do a lot of other things we don't expect
console.log("execute this second part!")
```

Because of how JavaScript event loop works, async methods are moved to the event queue and the main thread keeps running. We expect that nothing runs until the timer completes. In reality, however, the program keeps running. Our brains are not configured to think as event loop does, oops.

## Thunks

If you have worked with React, you've most probably heard of this bizarre name. There's a library called redux thunk in the React eco system. As a matter of fact, redux thunk implements the same logic as the pattern I'm going to explore.

I'll first steal some contents from the redux thunk docs:

A thunk is a function that wraps an expression to delay its evaluation.

```javascript
// calculation of 1 + 2 is immediate
// x === 3
let x = 1 + 2

// calculation of 1 + 2 is delayed
// foo can be called later to perform the calculation
// foo is a thunk!
let foo = () => 1 + 2
```

That's kind of boring, let's spice things up and introduce time complexity to it:

```javascript
function addAsync(x, y, cb) {
  setTimeout(function() {
    cb(x + y)
  }, 1000)
}

const thunk = cb => addAsync(10, 15, cb)

thunk(console.log)
// 25
```

Every time we call thunk with a callback, that callback will always get 25.

Let's write a thunk factory function:

```javascript
function makeThunk(fn) {
  const args = [].slice.call(arguments, 1)
  return function(cb) {
    args.push(cb)
    fn.apply(null, args)
  }
}
```

Provided with a callback function and extra arguments, the makeThunk function will return a function, which also takes a callback function. When the returned thunk function is called with a callback, the previously provided callback function will be called with the extra arguments plus the later provided callback function.

My brain got twisted to understand this makeThunk function, I suggest you play with it a few times.

Now we can achieve the same effect as the previous code:

```javascript
const thunk = makeThunk(addAsync, 10, 15)
thunk(console.log)
// 25
```

By implementing thunk, we've factored time out of our program and are able to reason about our code in a time independent way. Let's see how easy it is to write synchronous looking code that deals with async tasks.

Suppose we have an ajax method that takes an url and a callback, when the ajax call succeeds, the callback gets called with the resolved value. We can write a thunk to help us get the value out.

```javascript
function ajax(url, cb) {
  const fake_responses = {
    url1: "The first text",
    url2: "The middle text",
    url3: "The last text"
  }
  setTimeout(function() {
    cb(fake_response[url])
  }, 3000)
}

function getFile(url) {
  let res, fn
  ajax(url, function(response) {
    if (fn) fn(response)
    else res = response
  })

  return function(cb) {
    if (res) cb(res)
    else fn = cb
  }
}

const thunk = getFile(url1)
thunk(console.log)
// "The first text"
```

When called with an url, the getFile method triggers the ajax method to be called immediately and then returns a thunk function. When the returned thunk is called with a callback, if the response from the ajax call is ready, the callback will be called immediately with the response, otherwise, the callback will be delegated to the ajax function to be called. Either way, the callback provided to the thunk will be called with the response.

Thunks still make use of callbacks, so the problems with Inversion of control persist. However, they provide us a way of writing code that eliminates time as a complicating factor. We're able to write synchronous looking async code now.

Since we are on this topic, let's examine how redux thunk works. First let's take a look at the source code!

```javascript
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === "function") {
      return action(dispatch, getState, extraArgument)
    }

    return next(action)
  }
}

const thunk = createThunkMiddleware()
thunk.withExtraArgument = createThunkMiddleware

export default thunk
```

Yes, the redux thunk library is just 14 lines of code!

For the sake of simplicity, we'll omit the extraArgument part. The createThunkMiddleware will be called twice before returns a thunk. The Redux library handles the first two calls, so we'll ignore them. Normally when we call an action creator in redux, it will return an action object.

```javascript
function increment() {
  return {
    type: "INCREMENT",
    payload: 1
  }
}
```

When we handle with async task, we can return a function instead of an object:

```javascript
function asyncInc() {
  return dispatch => {
    setTimeout(
      dispatch(() => ({
        type: "INCREMENT",
        payload: 1
      })),
      1000
    )
  }
}
```

The returned action will be processed by the thunk middleware. When the thunk middleware detects that we are passing a function as an action, it will call the action function for us, and returns the value. Again, thunks help to eliminate time as a complicating factor.

## Promise

The basic syntax of promises looks like this:

```javascript
const promise1 = new Promise(function(resolve, reject) {
  setTimeout(resolve, 100, "foo")
})

promise1.then(console.log)
// foo
```

You can chain promises easily:

```javascript
const delay = ms => new Promise(resole => setTimeout(resolve, ms))

delay(100)
  .then(() => delay(200))
  .then(() => delay(700))
  .then(() => console.log("All done"))

// After 1 second, print "All done"
```

I assume that you've already known the basic usage of promises. So I'll jump directly to the interesting part.

### Problems that promises solve

    1. Promises solve Inversion of Control. We are in control of calling a method when a time dependent state resolves.
    2. Promises only resolve once, and stay immutable once resolved. Less surprises in our program.
    3. The promises chain gets executed in order. We can easily reason about how the promises chain cascades down. The code runs as we expect.
    4. Error handling becomes much easier than callbacks.

### Problems introduced by promises

The downside of promises are generally overlooked by the JS community. This is too big a topic to be covered in this article. So I suggest go read this [article](https://medium.com/@avaq/broken-promises-2ae92780f33). Here is a TLDR version of the article:

    1. Promises are eagerly evaluated: when they are constructed, they immediately begin work. They don’t care whether users are interested in the result of the operation.
    2. Promises can't be cancelled. Sometimes we need to cancel an async operation because of UI events. Say a user goes back to previous page before the ajax call completes. In this case, the user is not interested at the result of the promise, we need to cancel it. Promises don't support this feature.
    3.Promises don't force users to provide a rejection handler, which may cause headaches when exceptions arise.

### Async/Await

The async/await syntax introduced by ES7 is a great improvement in asynchronous programming. In essence, async/await is just a syntax sugar around promises. However, it enables us to write terse and synchronous looking async code.

Checkout this example:

```javascript
async getSomething(url) {
    const something = await fetchSomethingFromAPI(url);
    return something
}
```

Much more like synchronous code than promises!

### Pitfalls to avoid in async/await

Let's say we have two async dependencies in our function:

```javascript
async getObj() {
    const a = await fetchA();
    const b = await fetchB();
    return {a, b}
}
```

I'll confess that I've being writing this kind of code for a while, until I find the gotcha. Although this piece of code looks elegant and all, it has a performance issue. the fetchB function will wait for the fetchA function to complete! That is unnecessary since fetchB doesn't depend on fetchA. They can run in parallel. Here's the fix:

```javascript
async getObj() {
    const aPromise = fetchA();
    const bPromise = fetchB();
    const a = await aPromise;
    const b = await bPromise;
    return {a,b}
}
```

Here's another even worse scenario. Say you fetch a list of users based on a list of ids.

Maybe you want to write like this:

```javascript
async getUsers(ids) {
    const users = ids.map(id => await fetchUser(id));
    return users
}
```

This will cause the main function to wait between each fetchUser call. Again, this is unnecessary and can be refactored to run in parallel. The fix:

```javascript
async getUsers(ids) {
    const fetchUserPromises = ids.map(id => fetchUser(id));
    const users = await Promise.all(fetchUserPromises);
    return users;
}
```

### An alternative to error handling in async/await

Error handling in async/await is a pain to deal with. Usually we'll use try/catch like this:

```javascript
async getSomething(url) {
    try {
        const something = await fetchSomethingFromAPI(url);
        return something
    } catch(err) {
        console.log(err)
    }
}
```

This has two down sides. First, the try/catch syntax is a pain to write again and again. It really negatively affects the experience of writing code. Second, try/catch will catch every exception in the block, some other exceptions which are not usually caught by promises will be caught.

The alternative approach I'll introduce is borrowed from Golang. In Golang, you can deal with errors like this:

```go
data, err := db.Query("SELECT ...")
if err != nil { return err }
```

We can implement this style in JavaScript like this:

```javascript
function to(promise) {
   return promise.then(data => {
      return [null, data];
   })
   .catch(err => [err]);

async getSomething(url) {
    const [err, something] = await to(fetchSomethingFromAPI(url));
    return err ? err : something
```

That's all for today. Next time we will explore more advanced fancy patterns in asynchronous programming. Stay tuned!
