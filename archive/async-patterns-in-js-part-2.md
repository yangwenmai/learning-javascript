# Asynchronous Patterns in JavaScript (Part 2)

Last time I've examined callbacks, thunks and promises in dealing with async control flows in JavaScript. Now it's time to get over the basics level up our skills. I'll introduce you some of the most exciting techniques in asynchronous programming. A caveat to keep in mind before we dive in is that these patterns are not to replace the previous patterns. We still have to make use of promises, thunks, and even callbacks a lot in our programming practice. These advanced patterns I'm about to talk about are powerful when applied to suitable scenarios. Otherwise they could become cumbersome.

Also bear in mind that fully understanding these patterns is not an easy task. It took me months of practice to be able to talk about them comfortably. I bet some of the code I'm going to show you will blow your mind. Hope you're already excited!

## Generators and Iterators

If you're not already familiar with Generators and Iterators in JavaScript, go to the [MDN documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators) page to learn the details about them. I'll skip the basics.

Generators enable us to pause a function during its execution. This feature helps us to reason about the async control flows in our program easily.

### Redux Saga

Another fancy word from the Redux world! I don't even know what "saga" means. But who cares!

Redux saga takes advantage of generators to manage the side effects of your redux application. Generators can easily be started, paused, resumed, and even cancelled, enabling us to deal with complicated asynchronous logic while writing reasonable synchronous looking code.

Here's a basic example of redux saga:

```javascript
import { call, put, takeEvery, takeLatest } from "redux-saga/effects"
import Api from "..."

// worker Saga: will be fired on USER_FETCH_REQUESTED actions
function* fetchUser(action) {
  try {
    const user = yield call(Api.fetchUser, action.payload.userId)
    yield put({ type: "USER_FETCH_SUCCEEDED", user: user })
  } catch (e) {
    yield put({ type: "USER_FETCH_FAILED", message: e.message })
  }
}

/*
  Starts fetchUser on each dispatched `USER_FETCH_REQUESTED` action.
  Allows concurrent fetches of user.
*/
function* mySaga() {
  yield takeEvery("USER_FETCH_REQUESTED", fetchUser)
}

/*
  Alternatively you may use takeLatest.

  Does not allow concurrent fetches of user. If "USER_FETCH_REQUESTED" gets
  dispatched while a fetch is already pending, that pending fetch is cancelled
  and only the latest one will be run.
*/
function* mySaga() {
  yield takeLatest("USER_FETCH_REQUESTED", fetchUser)
}

export default mySaga
```

Each time the generator function encounters `yield`, it will pause. After the value gets resolved in the yielded expression, the `next()` method from Redux will call `next()` (Notice that these two `next` are different!) on the iterator returned from the `fetchUser` generator for us.

Remember that in the previous post I bashed promises for not being able to be cancelled? Redux saga solves this problem! Although under the hood the `fetchUser` method may still use a promise, but the async flow can be cancelled. Cancelling an async flow is not an easy task to do by yourself. It's better to use a library like redux saga.

### Async Iterator

Suppose that we have a data store, which, when requested, will give us a piece of data every 50 ms. We need to find a smart way to iterate the data stream asynchronously as it keeps emitting values. We can implement an Async Iterator protocol to do this:

```javascript
// This store will emit data every 500 ms
const store = createAsyncStore()

const users = {
  [Symbol.asyncIterator]: function() {
    let i = 0
    return {
      next: async function() {
        i++
        const user = await store.get("user", i)

        if (!user) {
          return { done: true }
        }
        return {
          value: customer,
          done: false
        }
      }
    }
  }
}
```

Compared with regular iterables, the main difference of the `users` iterable is that the `next` method is an async function. We'll handle this difference via the `for await ... of` syntax.

```javascript
;(async function iterateUsers() {
  for await (user of users) {
    doSomethingWithUser(user)
  }
})()
```

Since the `await` keyword needs to be inside an async function, we wrap it in an async function and immediately call that function.

### Async Generator

Sometimes when you deal with a huge set of data from the server, you're not able to process the data all at once. Imagine that there are Gigabytes of images available for processing, downloading them in a bundle is obviously a ridiculous idea. What we can do is to make a stream out of the data and process them gradually.

We'll build a silly application that fetches all the cats images from Flickr, and then display these images one by one, with 2 seconds latency between each display. You'll need a proxy service to make requests to Flickr if you are in China. You also need an API key to proceed with this project. Go [here](https://www.flickr.com/services/api/explore/flickr.photos.search) and click 'Call Method', you'll find an URL at the bottom, then grab the `api_key` from that URL. We need the API key to make requests.

```javascript
getApiKey = () => "898da2e2596df8f27ed691226bdc5357"

// Get the Dom element where the image will be displayed
const imgElm = document.getElementById("img")

// Method to replace image source
const setImg = url => elm => {
  elm.src = url
}

/* A higher order async iterator function, that takes a duration of time and an async iterable, returns a modified iterable with each iteration being delayed with the provided time frame
*/
function delayed(delaySeconds, iterable) {
  const delay = seconds =>
    new Promise(resolve => setTimeout(resolve, seconds * 1000))

  return {
    [Symbol.asyncIterator]: function() {
      const iterator = iterable[Symbol.asyncIterator]()
      const delayedIterable = {
        next: () => {
          return delay(delaySeconds).then(() => iterator.next())
        }
      }
      return delayedIterable
    }
  }
}

// This function will perform a search against the flickr api, and returns an array of image urls of the specified page
function flickrTagSearch(tag, page) {
  const apiKey = getApiKey()
  return fetch(
    "https://api.flickr.com/services/rest/" +
      "?method=flickr.photos.search" +
      "&api_key=" +
      apiKey +
      "&page=" +
      page +
      "&tags=" +
      tag +
      "&format=json" +
      "&nojsoncallback=1"
  )
    .then(response => response.json())
    .then(body => body.photos.photo)
    .then(photos =>
      photos.map(
        photo =>
          `https://farm${photo.farm}.staticflickr.com/` +
          `${photo.server}/${photo.id}_${photo.secret}_q.jpg`
      )
    )
}

function performSearchAndGetRes(tag) {
  return {
    /* This async generator will keep producing image urls.
         When one page is iterated, it will automatically request the next page! */
    [Symbol.asyncIterator]: async function*() {
      let pageIndex = 1
      while (true) {
        const pageData = await flickrTagSearch(tag, pageIndex)
        for (const url of pageData) {
          yield url
        }
        pageIndex = pageIndex + 1
      }
    }
  }
}

// Get the cats urls, which are now in an iterable.
const cats = delayed(2, performSearchAndGetRes("cats"))

// Iterate over the cats iterable and use each url to replace source of the image node
;(async function run() {
  for await (const cat of cats) {
    setImg(cat)(imgElm)
  }
})()
```

Checkout this [online demo](https://codepen.io/leihuang/pen/vrGeVv)! (If it doesn't work, just get a fresh API key, and replace the original one.)

## Observables

The Observer pattern offers a subscription model in which objects subscribe to an event and get notified when the event occurs. Rx.js is the default choice when implementing this pattern. What impresses me the most about Rx.js is that it enables us to specify the dynamic behavior of a value just when we declare it.

Here is a simple example:

```javascript
const { fromEvent } = "rxjs"
import { ajax } from "rxjs/ajax"
const { throttleTime, pluck, timeout, reTry, flatMap } = "rxjs/operators"

const button = document.querySelector("button")
fromEvent(button, "click")
  .pipe(
    throttleTime(1000),
    flatMap(click => ajax("http://userapi.com")),
    pluck("response", "users"),
    timeout(3000),
    reTry(3)
  )
  .subscribe(displayUsers)

const displayUsers = users =>
  // I'll omit the display detail, since I'm lazy.
  users.forEach(user => displayLogic(user))
```

The code above converts user clicks into a stream, which gets mapped to a stream of ajax requests. When the ajax call resolves, we get the result and display the result. We only allow one request to be dispatched within 1 second by calling `throttleTime(1000)`. If the request doesn't resolve after 3 seconds, we'll cancel the request and throw an error. By calling `reTry(3)`, we let Rx.js to retry 3 times in total if the request fails.

Notice how easy it is to implement these functionality with just a few utility functions.

I've written a post on [Rx.js](https://leihuang.me/posts/rxjs-in-action). Checkout if you want to learn more.

## Communicating Sequential Process (CSP)

Communicating Sequential Process (CSP) is borrowed from Golang and ClojureScript. It's a software mechanism that models concurrency with channels. Channels are like a pipe that has no buffer size, so as to enable back pressure message passing. Imagine there's a pipe that has a receiver and a pusher at each end. Since the pipe has no buffer size, the pusher can't push new messages if the receiver is not ready to receive. On the contrary, the receiver can't get messages out if the pusher is not ready to push.

By implementing channels, we can model each part of our software in separate independent sequential processes that don't need to know about each other. All they care about is sending and receiving messages through the channel. Process A doesn't care about the implementation details of process B. This is a huge improvement in software design. Writing decoupled and easy to reason about code becomes much easier.

We all know that JavaScript is single threaded, it can't have separate processes running independently. However, we can simulate this behavior even in a single thread. Luckily for us, generators can easily model the back pressure mechanism by pausing the execution.

Here's a piece of simple demo code:

```javascript
// The channel creator function will be provided by a library like js-csp
const ch = chan()

function* process1() {
  yield put(ch, "Hello")
  const msg = yield take(ch)
  console.log(msg)
}

function* process2() {
  const greeting = yield take(ch)
  yield put(ch, greeting + "World")
  console.log("done!")

  // Hello World
  // done!
}
```

We first created a channel by calling the `chan` function. The `put` and `take` functions will, well, put and take messages through the channel. Surprise! When the process1 function encounters the `put` expression, it will pause and wait for a `take` call from elsewhere. When process2 calls `take`, if the channel has something to take, which, in our case, means that the process1 has already called `put`, it will keep running. When process2 encounters `put`, it will pause. After process2 called `take` previously, process1 got unblocked and continued until it meets `take`.

You can see that these two processes run independently. If the channel has nothing to take, the `take` expression in either process will pause. If the message in a channel hasn't been taken, the process that puts that message to the channel will pause, too. Both `take` and `put` keep agnostic of the states of the outer world.

Here's another more interesting example:

```javascript
// We are still using the js-csp library
scp.go(function*() {
  while (true) {
    yield csp.put(ch, Math.random())
  }
})

csp.go(function*() {
  yield csp.take(csp.timeout(500))
  const num = yield csp.take(ch)
  console.log(num)
})
```

The `csp.go` function provided by the `js-csp` library models go routines. I'll not go deep into this concept. In our case, we just need to know that the go routine automatically creates a channel for us!

The first go routine will run infinitely. Don't be scared, it will not run any faster than the taker of the channel from elsewhere! The second go routine takes messages from the channel every 500 milliseconds, which means that the first go routine will also put values into the channel at this rate.

### Redux saga, again

The redux saga library really makes use of generators to their fullest potential. Apart from using generators to deal with regular side effects, it also implements CSP channels! If you understand the two pieces of demo code above, you'll grasp the same idea in redux saga easily. Read the docs [here](https://redux-saga.js.org/docs/advanced/Channels.html).

## The End

In this two series of posts, we've covered most of async patterns in JavaScript. These patterns are not mutually exclusive,. You can use them all to suit your needs, even callbacks! (Checkout this [project](https://github.com/callbag/callbag) for inspiration) There's another pattern from Haskell I'd like to explore, which is called "future". However, I'm not yet proficient in purely functional programming. I've always wanted to try this in production projects, but that can cause problems in collaboration and code review. After all, not everyone in our team are ready to fully embrace this fantasy land. Maybe in the future I'll try it in my side projects. If it goes well, I'll write about it!

Thanks for reading!
