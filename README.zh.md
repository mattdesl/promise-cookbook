# promise-cookbook（Promise 手册）

这是一份 JavaScript 语言叙述的 Promise 简要介绍，主要面向的读者是前端开发者。

## 目录

- [简介](#简介)
- [问题的提出](#问题的提出)
- [`async`](#async)
- [promises](#promises)
  - [`new Promise()`](#new-promise)
  - [`.then(resolved, rejected)`](#thenresolved-rejected)
  - [`.catch(err)`](#catcherr)
  - [chaining](#chaining)
  - [resolving values](#resolving-values)
  - [`Promise.all()`](#promiseall)
  - [passing the buck](#passing-the-buck)
  - [`throw` and implicit catch](#throw-and-implicit-catch)
- [common patterns](#common-patterns)
  - [memoization](#memoization)
  - [`Promise.resolve` / `Promise.reject`](#promiseresolve--promisereject)
  - [handling user errors](#handling-user-errors)
  - [`Promise` in ES2015](#promise-in-es2015)
- [陷阱](#陷阱)
  - [小模块中的 promise](#小模块中的 promise)
  - [complexity](#complexity)
  - [lock-in](#lock-in)
- [further reading](#further-reading)

## 简介

Promise 是一种为解决异步编程痛苦而生的程序结构。使用 Promises 可以使你的代码更为精简，而且更易扩展。

本文主要着眼于 ES6 的 Promise 语法，但是会使用 [Bluebird](https://github.com/petkaantonov/bluebird)，因为它在浏览器端提供了极好的错误处理功能。CommonJS 语法的程序需要类似 [browserify](https://github.com/substack/node-browserify) 或者 [webpack](http://webpack.github.io/) 这样的工具的辅助才能运行在浏览器端。要了解 CommonJS 和 browserify，请参考 [jam3-lesson-module-basics](https://github.com/Jam3/jam3-lesson-module-basics)。

## 问题的提出

我们从浏览器端载入图片的问题开始。下面的代码展示了一种 Node 风格（error-first）回调的解决方式。

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

<sub>*提示:* 上面的代码使用了 [img](https://www.npmjs.com/package/img) 模块。</sub>

载入单个图片相对简单：

```js
loadImage('one.png', function(err, image) {
  if (err) throw err;
  console.log('Image loaded', image);
});
```

然而，当我们的应用越来越复杂的时候，这么做就不合适了。如果我们使用相同的办法，载入 3 个图片，这么写就显得很不明智了：

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

这种层层嵌套的结构就像一颗圣诞树，这会让你的代码丧失可读性，难以维护。
而且，如果我们想要并行地载入图片，就需要一个[更复杂的解决方案](https://gist.github.com/mattdesl/7b2afa86481fbce87098). 

## async

人们发明了很多种对 error-first 回调的抽象，有些叫做“errbacks”。

解决这个问题的方法之一是用 [async](https://github.com/caolan/async) 模块:

```js
var mapAsync = require('async').map;

var urls = [ 'one.png', 'two.png' ];
mapAsync(urls, loadImage, function(err, images) {
  if (err) throw err;
  console.log('All images loaded', images);
});
```

npm 上还有很多相似的解决方案，比如：

- [async-each](https://www.npmjs.com/package/async-each)
- [async-each-series](https://www.npmjs.com/package/async-each-series)
- [run-series](https://www.npmjs.com/package/run-series)
- [run-waterfall](https://www.npmjs.com/package/run-waterfall)
- [map-limit](https://www.npmjs.com/package/map-limit)

这种方法非常棒。这是一种合适的 [对小模块的解决方案](#小模块中的 promise)，因为它没有引入额外的概念和依赖，也没有 [promise 陷阱](#陷阱)。

然而，在处理大规模应用的回调问题时，promise 可以给你的应用提供一个统一的，可组合的结构。有人已经在 [ES7 async/await](https://jakearchibald.com/2014/es7-async-functions/) 上做了些基础工作。

## promises

让我们用 promise 的方式解决上面的问题。一开始看起来似乎会多一些开销，但是马上你好看到它带来的好处。

### `new Promise()`

下面就是用 pormise 实现的图片加载函数。为区别之前的例子，我们叫它 `loadImageAsync` 。

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

这个函数返回一个 `Promise` 的实例，会在图片加载成功时调用 *resolve*，或者加载出错时调用 *reject*，并抛出 `Error`。
在上面的例子中，使用 `require('bluebird')` 加载 bluebird 这个 Promise 实现。

`Promise` 构造器仅在上面这种需要把一个回调风格的 API 转化成 Promise 风格的 API 的情况下使用。在很多情况下，我们可以使用 `promisify` 或者 `denodeify` 方法把回调风格的函数 转化成对应的 Promise 风格的。

举个例子，在原有的 [`loadImage`](#问题的提出) 函数基础上，上面的代码可以非常简洁：

```js
var Promise = require('bluebird');
var loadImageAsync = Promise.promisify(loadImage);
```

或者直接使用 [img](https://www.npmjs.com/package/img) 模块:

```js
var Promise = require('bluebird');
var loadImage = require('img');
var loadImageAsync = Promise.promisify(loadImage);
```

如果你不用 Bluebird，你可以使用 [es6-denodeify](https://www.npmjs.com/package/es6-denodeify) 作为替代。

### `.then(resolved, rejected)`

每个 `Promise` 实例的原型都有一个 `then()` 方法。这可以让我们处理异步任务的结果。

```js
loadImageAsync('one.png')
  .then(function(image) {
    console.log('Image loaded', image);
  }, function(err) {
    console.error('Error loading image', err);
  });
```

`then` 有两个函数类型参数，二者中的其一可以是 `null` 或 `undefined`。
`resolved` 回调函数将会在 promise 成功时被调用，并且会传递 “resolved value”（在这个例子中就是 `image`）。
`rejected` 回调函数将会在 promise 失败是被调用，并且传递 `Error` 对象。

### `.catch(err)`

Promises 也有一个 `.catch(func)` 用来处理错误，与 `.then(null, func)` 相似，但是意图更明确。

```js
loadImageAsync('one.png')
  .catch(function(err) {
    console.error('Could not load image', err);
  });
```

### chaining

`.then()` 方法*总是返回一个 Promise*，这意味着它可以链式地使用。
上面的代码可以像这样重写。
如果 promise 被拒绝（rejected），下一个 `catch()` 或 `then(null, rejected)` 将会被调用。

在下面的例子中，如果 `loadImageAsync` 方法被拒绝，控制台唯一的输出就会是错误信息。

```js
loadImageAsync('one.png')
  .then(function(image) {
    console.log('Image loaded', image);
    return { width: image.width, height: image.height };
  })
  .then(function(size) {
    console.log('Image size:', size);
  })
  .catch(function(err) {
    console.error('Error in promise chain', err);
  });
```

一般来说，promise 链不应该过长。它们会变得很难维护，那些异步任务应该被分拆为更小的，带名字的函数。

### resolving values

`then()` and `catch()` 回调可以返回一个值，传递给链中的下一个方法。举个例子，我们可以在加载出错时使用默认图片：

```js
loadImageAsync('one.png')
  .catch(function(err) {
    console.warn(err.message);
    return notFoundImage;
  })
  .then(function(image) {
    console.log('Resolved image', image);
  });
```

上面的代码会尝试载入 `'one.png'`，但如果加载失败，就会使用 `notFoundImage`。

有个很酷的用法是，你可以返回一个 `Promise` 实例，并且它会在下一个 `.then()` 触发之前被 resolved。这个 promise 的 resolved value 也会传递给下一个 `.then()`。

```js
loadImageAsync('one.png')
  .catch(function(err) {
    console.warn(err.message);
    return loadImageAsync('not-found.png');
  })
  .then(function(image) {
    console.log('Resolved image', image);
  })
  .catch(function(err) {
    console.error('Could not load any images', err);
  });
```

上面的代码尝试载入 `'one.png'`，如果载入失败就会载入 `'not-found.png'`。

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
    .then(renderGallery)
    .catch(renderEmptyGallery);
}

showUserImages('mattdesl')
  .catch(function(err) {
    showError(err);
  });
```

### `throw` and implicit catch

If you `throw` inside your promise chain, the error will be impliticly caught by the underlying Promise implementation and treated as a call to `reject(err)`. 

In the following example, if the user has not activated their account, the promise will be rejected and the `showError` method will be called.

```js
loadUser()
  .then(function(user) {
    if (!user.activated) {
      throw new Error('user has not activated their account');
    }
    return showUserGallery(user);
  })
  .catch(function(err) {
    showError(err.message);
  });
```

This part of the specification is often viewed as a pitfall of promises. It conflates the semantics of error handling by combining syntax errors, programmer error (e.g. invalid parameters), and connection errors into the same logic. 

It leads to frustrations during browser development: you might lose debugger capabilities, stack traces, and source map details.

![debugging](http://i.imgur.com/Y6RH8ke.png)

For many developers, this is enough reason to eschew promises in favour of error-first callbacks and abstractions like [async](#async).

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

### handling user errors

Functions that return promises should *always* return promises, so the user does not need to wrap them in a `try/catch` block.

Instead of throwing errors on invalid user arguments, you should return a promise that rejects with an error. [Promise.reject()](#promiseresolve--promisereject) can be convenient here.

For example, using our earlier [`loadImageAsync`](#new-promise):

```js
function loadImageAsync(url) {
  if (typeof url !== 'string') {
    return Promise.reject(new TypeError('must specify a string'));
  }

  return new Promise(function (resolve, reject) {
    /* async code */
  });
}
```

Alternatively, you could use `throw` inside the promise function:

```js
function loadImageAsync(url) {
  return new Promise(function (resolve, reject) {
    if (typeof url !== 'string') {
      throw new TypeError('must specify a string');
    }

    /* async code */
  });
}
```

See [here](https://www.w3.org/2001/tag/doc/promises-guide#always-return-promises) for details.

### `Promise` in ES2015

Although this guide uses [bluebird](https://github.com/petkaantonov/bluebird), it should work in any standard Promise implementation. For example, using [Babel](https://babeljs.io/docs/learn-es2015/#promises).

Some other implementations: 

- [pinkie-promise](https://github.com/floatdrop/pinkie-promise)
- [es6-promise](https://www.npmjs.com/package/es6-promise)

For example, in Node/browserify:

```js
// use native promise if it exists
// otherwise fall back to polyfill
var Promise = global.Promise || require('es6-promise').Promise;
```

## pitfalls

In addition to the the issues mentioned in [`throw` and implicit catch](#throw-and-implicit-catch), there are some other problems to keep in mind when choosing promises. Some developers choose not to use promises for these reasons.

### promises in small modules

One sitaution where promises are not yet a good fit is in small, self-contained [npm](https://www.npmjs.com/) modules.

- Depending on `bluebird` or `es6-promise` is a form of vendor lock-in. It can be a problem for frontend developers, where bundle size is a constraint. 
- Expecting the native `Promise` (ES2015) constructor is also a problem, since it creates a peer dependency on these polyfills.
- Mixing different promise implementations across modules may lead to subtle bugs and debugging irks.

Until native Promise support is widespread, it is often easier to use Node-style callbacks and independent [async modules](#async) for control flow and smaller bundle size. 

Consumers can then "promisify" your API with their favourite implementation. For example, using the [xhr](https://www.npmjs.com/package/xhr) module in Bluebird might look like this: 

```js
var Promise = require('bluebird')
var xhrAsync = Promise.promisify(require('xhr'))
```

### complexity

Promises can introduce a lot of complexity and mental overhead into a codebase (evident by the need for this guide). In real-world projects, developers will often work with promise-based code without fully understanding how promises work.

See Nolan Lawson's ["We Have a Problem With Promises"](http://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html) for an example of this.

### lock-in

Another frustration is that promises tend to work best once *everything* in your codebase is using them. In practice, you might find yourself refactoring and "promisifying" a lot of code before you can reap the benefits of promises. It also means that new code must be written with promises in mind — you are now stuck with them!

## further reading

- [JavaScript Promises: There and back again](http://www.html5rocks.com/en/tutorials/es6/promises/)
- [We Have a Problem With Promises](http://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)
- [You're Missing the Point of Promises](https://blog.domenic.me/youre-missing-the-point-of-promises/)

## License

MIT, see [LICENSE.md](http://github.com/mattdesl/promise-cookbook/blob/master/LICENSE.md) for details.
