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
  - [`throw` 和隐式 catch](#throw-和隐式-catch)
- [常见模式](#常见模式)
  - [memoization](#memoization)
  - [`Promise.resolve` / `Promise.reject`](#promiseresolve--promisereject)
  - [handling user errors](#handling-user-errors)
  - [ES2015 中的 `Promise`](#ES2015-中的-`Promise`)
- [陷阱](#陷阱)
  - [小模块中的 promise](#小模块中的-promise)
  - [复杂性](#复杂性)
  - [lock-in](#lock-in)
- [延伸阅读](#延伸阅读)

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

回到最开始的载入多个图片的问题。

`Promise.all()` 方法接受的参数可以是数组，或者是 promise 对象，并返回一个新的 `Promise` 对象，这个新的 `Promise` 对象只会在*所有* promise 对象进入 resolve 状态后变成 resolve 的。下面我们用 `loadImageAsync` 把每个 URL 映射为一个新的 promise 对象，然后传递给 `all()`。

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

这样，加载多个图片的代码看起来要清晰一些了。

### passing the buck

你可能想知道 promise 风格的解决方法与 `async` 方式相比优势在哪。当你需要组合多个 promise 时，你就能体会它的优势了。

我们可以声明多个返回 promise 的具名函数，并且让错误信息冒泡到上层函数。上面的代码可以改写为这样：

```js
function loadImages(urls) {
  var promises = urls.map(loadImageAsync);
  return Promise.all(promises);
}
```

更复杂的例子会像这样：

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

### `throw` 和隐式 catch

如果在 promise 链中 `throw` ，错误会被 Promise 底层代码隐式地 catch，并调用 `reject(err)`。

在下面的例子中，如果用户没有激活他的账号，promise 将会被 rejected，并且 `showError` 方法将会被调用。

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

Promise 标准的这部分经常被视为它的一个陷阱。
它把所有错误处理的语义混淆了。语法错误，编码者错误（比如非法参数），和连接错误被糅合到相同的逻辑里去了。

这会给浏览器端开发带来麻烦：你可能无法调试，无法追查（你想要看到的）调用栈。

![debugging](http://i.imgur.com/Y6RH8ke.png)

对大多数开发者来说，就这个理由就足以让他们抛弃 promise 回归 error-first 回调风格和类似 [async](#async) 这样的工具.

## 常见模式

### memoization

我们在异步任务完成后使用 `.then()`。比如，我们可以缓存第一次请求的结果，resolve 同一个 `Image` 对象，而不是每次都请求同样的 `'not-found.png'` 图片。

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

这在服务端可能更有用，因为浏览器已经有缓存机制。

### `Promise.resolve` / `Promise.reject`

`Promise` 本身也提供 `resolve` 和 `reject` 方法。调用它们时，将会返回一个新的 promise，这个新的 promise 已经是 resolved 或 rejected 状态。

举个例子:

```js
var thumbnail = Promise.resolve(defaultThumbnail);

// 查询数据库
if (userLoggedIn) {
  thumbnail = loadUserThumbnail();
}

// 当 DOM 是 ready 状态时，添加图片到 DOM
thumbnail.then(function(image) {
  document.body.appendChild(image);
});
```

这里的 `loadUserThumbnail` 返回一个 `Promise`，并可以从中取到一个图片。有了 `Promise.resolve`，即使我们不去进行查询数据库的操作，也同样可以获得一个 `Promise`，并且不需要改动后面的代码。

### handling user errors

返回 promise 的函数应该*总是*返回 promise，这样使用它的时候就不需要用 `try/catch` 包裹这些函数。

在出现错误是，你应该使用 reject 返回错误，而不是抛出一个错误。[Promise.reject()](#promiseresolve--promisereject) 在这种场景下非常好用。

举个例子，这里用到了早先定义的 [`loadImageAsync`](#new-promise):

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

或者可以在 promise 函数内部使用 `throw`：

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

点击[这里](https://www.w3.org/2001/tag/doc/promises-guide#always-return-promises) 可以了解更多细节。

### ES2015 中的 `Promise`

尽管本文使用了 [bluebird](https://github.com/petkaantonov/bluebird)，但上述也同样适用于标准的 Promise 实现。比如，[Babel](https://babeljs.io/docs/learn-es2015/#promises) 中的 Promise。

一些其它的实现：

- [pinkie-promise](https://github.com/floatdrop/pinkie-promise)
- [es6-promise](https://www.npmjs.com/package/es6-promise)

举个例子，在 Node/browserify 中：

```js
// 使用原生 promise ，如果不存在否则使用 polyfill
var Promise = global.Promise || require('es6-promise').Promise;
```

## 陷阱

除了 [`throw` 和隐式 catch](#`throw` 和隐式 catch) 这个陷阱，还有一些问题值得注意。

### 小模块中的 promise

有一个不适合使用 promise 的场景，就是它不适合加进一些独立、小巧的 [npm](https://www.npmjs.com/) 模块。

- 当打包大小有限制时，依赖 `bluebird` 或 `es6-promise` 占用一些空间。
- 使用 `Promise` (ES2015) 构造器也会有问题，因为它引入了对那些 polyfill 的依赖。
- 跨模块地混合使用不同的 promise 实现会导致一些微妙的 bug，调试时让人受尽折磨。

除非原生 Promise 普及，在小模块中，建议使用 Node 风格的回调和独立的 [async](#async) 模块来控制异步任务。

你可以使用任何一张你喜欢的 Promise 实现去 "promise 化"你的 API。比如，在 Bluebird 下使用 [xhr](https://www.npmjs.com/package/xhr) 模块：

```js
var Promise = require('bluebird')
var xhrAsync = Promise.promisify(require('xhr'))
```

### 复杂性

Promises 引入了很多的复杂性和额外的智力开销。在实际项目中，开发者经常要面对基于 promise 的代码，但不完全明白 promise 内里的机制。

请看 Nolan Lawson 的 ["We Have a Problem With Promises"](http://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html) 一文，里面给出了很多种 promise 的错误用法。

### lock-in

另一个令人沮丧的事实是，一旦你用了 promise，你就得在整个项目中使用它，以确保它能完美运行。
在实践中，你会发现，想要获得 promise 的诸多益处，需要先重构并 “promise 化” 很多代码。
这也意味着，写新的代码时必须以 promise 的方式思考——你会被这种思维方式折磨的，如果不熟练的话。

## 延伸阅读

- [JavaScript Promises: There and back again](http://www.html5rocks.com/en/tutorials/es6/promises/)
- [We Have a Problem With Promises](http://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)
- [You're Missing the Point of Promises](https://blog.domenic.me/youre-missing-the-point-of-promises/)

## 协议

MIT，请看 [LICENSE.md](http://github.com/mattdesl/promise-cookbook/blob/master/LICENSE.md) 。
