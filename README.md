# promise-cookbook

This is a brief introduction to using Promises in JavaScript, primarily aimed at frontend developers.

## contents

- [intro](#intro)
- [the problem](#the-problem)
- [async](#async)
- [promises](#promises)
  - [`new Promise()`](#new-promise)
  - [`.then(resolved, rejected)`](#thenresolved-rejected)
  - [chaining](#chaining)
  - [resolving values](#resolving-values)
  - [`.catch(err)`](#catcherr)
  - [`Promise.all()`](#promiseall)
  - [passing the buck](#passing-the-buck)
- [common patterns](#common-patterns)
  - [memoization](#memoization)
  - [`Promise.resolve` / `Promise.reject`](#promiseresolve--promisereject)
- [`Promise` in ES2015](#promise-in-es2015)
- [promises in small modules](#promises-in-small-modules)
- [further reading](#further-reading)

## intro

A Promise is a programming construct that can reduce some of the pains of asynchronous programming. Using Promises can help produce code that is leaner, easier to maintain, and easier to build on.

This lesson will mostly focus on ES6 Promise syntax, but will use [Bluebird](https://github.com/petkaantonov/bluebird) since it provides excellent error handling in the browser. The CommonJS syntax will need a bundler like [browserify](https://github.com/substack/node-browserify) or [webpack](http://webpack.github.io/). See [lesson-module-basics](https://github.com/Jam3/jam3-lesson-module-basics) for further details on modules and CommonJS.

## the problem

To demonstrate, let's take the problem of loading images in the browser. The following shows an implementation using a Node style (error-first) callback:

```javascript
function loadImage(url, callback) {
  var image = new Image();

  image.onload = function() {
    callback(null, image);
  };

  image.onerror = function() {
    callback(new Error('Could not load image at ' + url));
  };

  image.src = url;
}
```

<sub>*Tip:* The above is implemented on npm as [img](https://www.npmjs.com/package/img).</sub>

Loading a single image is relatively easy, and looks like this:

```js
loadImage('one.png', function(err, image) {
  if (err) throw err;
  console.log('Image loaded', image);
});
```

However, as our application grows in complexity, so too does the code. If we were to take the same approach, but load three images, things get a little unwieldy:

```js
loadImage('one.png', function(err, image1) {
  if (err) throw err;

  loadImage('two.png', function(err, image2) {
    if (err) throw err;

    loadImage('three.png', function(err, image3) {
      if (err) throw err;

      var images = [image1, image2, image3];
      console.log('All images loaded', images);
    });
  });
});
```

This tends to create a "Christmas Tree" of functions; and leads to code that is difficult to read and maintain. Further, if we wanted the images to load in parallel, it would need a [more complex solution](https://gist.github.com/mattdesl/7b2afa86481fbce87098). 

## async

One way to solve the problem is with [async](https://github.com/caolan/async) and similar abstractions:

```js
var each = require('async').each;

each(['one.png', 'two.png'], loadImage, function(err, images) {
  if (err) throw err;

  console.log('All images loaded', images);
});
```

Many of these helpers exist independently on npm, like:

- [async-each](https://www.npmjs.com/package/async-each)
- [async-each-series](https://www.npmjs.com/package/async-each-series)
- [run-series](https://www.npmjs.com/package/run-series)
- [map-limit](https://www.npmjs.com/package/map-limit)

In many cases, such as when developing small modules<sup>[1](#promises-in-small-modules)</sup>, this is a good solution. But in a larger scope, promises can provide a more unified and composable structure throughout your application.

## promises

Let's re-implement the above with promises for our control flow. At first this may seem like more overhead, but the benefits will become clear shortly.

### `new Promise()`

Below is how the image loading function would be implemented with promises. We'll call it `loadImageAsync` to distinguish it from the earlier example.

```js
var Promise = require('bluebird')

function loadImageAsync(url) {
  return new Promise(function(resolve, reject) {
    var image = new Image();

    image.onload = function() {
      resolve(image);
    };

    image.onerror = function() {
      reject(new Error('Could not load image at ' + url));
    };

    image.src = url;
  });
}
```

The funtion returns a new instance of `Promise` which is *resolved* to `image` if the load succeeds, or *rejected* with a new `Error` if it fails. In our case we are using bluebird as our Promise implementation.

The `Promise` constructor is typically only needed for edge cases like this, where we are converting a callback-style API into a promise-style API. In many cases it is preferable to use a `promisify` or `denodeify` utility which converts Node style (error-first) functions into their `Promise` counterpart.

For example, the above becomes very concise with our earlier `loadImage` function:

```js
var Promise = require('bluebird');
var loadImageAsync = Promise.promisify(loadImage);
```

Or with the [img](https://www.npmjs.com/package/img) module:

```js
var Promise = require('bluebird');
var loadImage = require('img');
var loadImageAsync = Promise.promisify(loadImage);
```

If you aren't using Bluebird, you can use [es6-denodeify](https://www.npmjs.com/package/es6-denodeify).

### `.then(resolved, rejected)`

Each `Promise` instance has a `then()` method on its prototype. This allows us to handle the result of the async task.

```js
loadImageAsync('one.png')
  .then(function(image) {
    console.log('Image loaded', image);
  }, function(err) {
    console.error('Error loading image', err);
  });
```

It takes two functions, either of which can be `null`. The `resolved` callback is called when the promise succeeds, and it is passed the resolved value (in this case `image`). The `rejected` callback is called when the promise fails, and it is passed the `Error` object we created earlier.

### chaining

The `.then()` method *always returns a Promise*, which means it can be chained. The above could be re-written like so, where the second callback handles any errors in the promise before it:

```js
loadImageAsync('one.png')
  .then(function(image) {
    console.log('Image loaded', image);
  })
  .then(null, function(err) {
    console.error('Error in promise chain', err);
  });
```

In general, you should be wary of long promise chains. They can be difficult to maintain and it would be better to split the tasks into smaller, named functions.

### resolving values

Your callbacks can return a value to pass it along to the next `.then()` method. For example:

```js
loadImageAsync('one.png')
  .then(null, function(image) {
    return notFoundImage;
  })
  .then(function(image) {
    console.log('Resolved image', image);
  }, function(err) {
    console.error('Could not find any images', err);
  });
```

The above code will try to load `'one.png'`, but will fall back to using `notFoundImage` if the load failed. 

The cool thing is, you can return a `Promise` instance, and it will be resolved before the next `.then()` is triggered. The value resolved by that promise will also get passed to the next `.then()`.

```js
loadImageAsync('one.png')
  .then(null, function(image) {
    return loadImageAsync('not-found.png');
  })
  .then(function(image) {
    console.log('Resolved image', image);
  }, function(err) {
    console.error('Could not find any images', err);
  });
```

The above tries to load `'one.png'`, but if that fails it will then load `'not-found.png'`, and only then would it resolve to the image.

### `.catch(err)`

You can also use `.catch()` to handle errors, which is the same as using `.then(null, handler)`.

```js
loadImageAsync('one.png')
  .then(function(image) {
    console.log('Loaded image', image);
  });
  .catch(function(err) {
    console.error('Could not load image', err);
  });
```

### `Promise.all()`

Let's go back to our original task of loading multiple images.

The `Promise.all()` method accepts an array of values or promises and returns a new `Promise` that is only resolved once *all* the promises are resolved. Here we map each URL to a new Promise using `loadImageAsync`, and then pass those promises to `all()`.

```js
var urls = ['one.png', 'two.png', 'three.png'];
var promises = urls.map(loadImageAsync);

Promise.all(promises)
  .then(function(images) {
    console.log('All images loaded', images);
  })
  .catch(function(err) {
    console.error(err);
  });
```

Finally, things are starting to look a bit cleaner.

### passing the buck

You may still be wondering where promises improve on the `async` approach. The real benefits come from composing promises across your application.

We can "pass the buck" by making named functions that return promises, and let errors bubble upstream. The above code would look like this:

```js
function loadImages(urls) {
  var promises = urls.map(loadImageAsync);
  return Promise.all(promises);
}
```

A more complex example might look like this:

```js
function getUserImages(user) {
  return loadUserData(user)
    .then(function(userData) {
      return loadImages(userData.imageUrls);
    });
}

function showUserImages(user) {
  return getUserImages(user)
    .then(renderGallery, renderEmptyGallery);
}

showUserImages('mattdesl')
  .catch(function(err) {
    showError(err);
  });
```

## common patterns

### memoization

We can use `.then()` on a promise even after the asynchronous task is long complete. For example, instead of always requesting the same `'not-found.png'` image, we can cache the result of the first request and just resolve to the same `Image` object.

```js
var notFound;

function getNotFoundImage() {
  if (notFound) {
    return notFound;
  }
  notFound = loadImageAsync('not-found.png');
  return notFound;
}
```

This is more useful for server requests, since the browser already has a caching layer in place for image loading.

### `Promise.resolve` / `Promise.reject`

The `Promise` class also provides a `resolve` and `reject` method. When called, these will return a new promise that resolves or rejects to the (optional) value given to them.

For example:

```js
var thumbnail = Promise.resolve(defaultThumbnail);

//query the DB
if (userLoggedIn) {
  thumbnail = loadUserThumbnail();
}

//add the image to the DOM when it's ready
thumbnail.then(function(image) {
  document.body.appendChild(image);
});
```

Here `loadUserThumbnail` returns a `Promise` that resolves to an image. With `Promise.resolve` we can treat `thumbnail` the same even if it doesn't involve a database query.

## `Promise` in ES2015

Although this guide uses [bluebird](https://github.com/petkaantonov/bluebird), it should work in any standard Promise implementation. For example, using [Babel](https://babeljs.io/docs/learn-es2015/#promises).

Another option is [es6-promise](https://www.npmjs.com/package/es6-promise), which might be used like so in Node/browser:

```js
// use native promise if it exists
// otherwise fall back to polyfill
var Promise = global.Promise || require('es6-promise').Promise;
```

## promises in small modules

When you begin to use promises, it may be tempting to use them for *all* asynchronous code. However, one situation where you may want to question the use of promises is in small, self-contained [npm](https://www.npmjs.com/) modules.

- Depending on `bluebird` or `es6-promise` is a form of vendor lock-in. It can be a problem for frontend developers, where bundle size is a constraint. 
- Expecting the native `Promise` (ES2015) constructor is also a problem, since it creates a peer dependency on these polyfills.
- Mixing different promise implementations across modules may lead to subtle bugs and debugging irks.

Until native Promise support is widespread, it is often easier to use Node-style callbacks and independent [async modules](#async) for control flow. 

Consumers can then "promisify" your API with their favourite implementation. For example, using the [xhr](https://www.npmjs.com/package/xhr) module in Bluebird might look like this: 

```js
var Promise = require('bluebird')
var xhrAsync = Promise.promisify(require('xhr'))
```

## further reading

- [JavaScript Promises: There and back again](http://www.html5rocks.com/en/tutorials/es6/promises/)
- [We Have a Problem With Promises](http://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)
- [You're Missing the Point of Promises](https://blog.domenic.me/youre-missing-the-point-of-promises/)

## License

MIT, see [LICENSE.md](http://github.com/Jam3/jam3-lesson-promises/blob/master/LICENSE.md) for details.
