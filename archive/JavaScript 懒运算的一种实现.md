# JavaScript 懒运算的一种实现

在学习 Haskell 时，我遇到了这种写法：

```haskell
sum (takeWhile (<10000) (filter odd (map (^2) [1..]))) 
```
这段代码的意思是，找出自然整数中小于 10000 的同时是乘方数和奇数的数字，再把这些数加总。由于 Haskell 的懒运算特性，上面的程序并不会立马生成从 1 到 无限大的自然数列表，而是会等待 `takeWhile` 指令，再生成符合条件的列表。如果用 JS 来写，很难写出这么简洁高表达性的代码。一个可能的思路就是写个 `while` 循环，然后找到符合条件的数进行加总。这个比较简单，我就不演示了。

但是如果我们要用高阶函数来模拟 Haskell 的写法，就要想个办法实现懒运算了。提到懒，首先想到的就是 `generator` 。没人踢它一脚告诉它 `next()`，它会一直坐那儿不动的。现在我们就来用 `generator` 来实现一个懒运算。

首先定义一个生成从 1 到无穷大自然数的 `generator` :

```js
const numbers = function*() {
  let i = 1;
  while (true) {
    yield i++;
  }
};
```

由于只有在 `generator` 执行后生成的 `iterable` 上执行 `next()` 方法，`yield` 才会执行，所以我们要做的主要工作就是实现不同的 `next` 方法，达到目的。

我们需要先创建一个类 `Lazy`，`Lazy` 封装了我们的各种目标操作 :

```js
class Lazy(iterator, callback) {
    this.iterator = iterator;
    this.callback = callback;
// ...其它细节

    next() {
        return this.iterable.next();
    }
 // ...其它细节  
}
```

我们先定义一个 `map` 方法，它会把每次 `next` 返回的值根据提供的回调函数进行修改：

```js
class Lazy(iterator, callback) {
    // ...其它细节
    map(callback){
        return new LazyMap(this, callback);
    }
    // ...其它细节
}

class LazyMap extends Lazy {
  next() {
    const item = this.iterable.next();

    const mappedValue = this.callback(item.value);
    return { value: mappedValue, done: item.done };
  }
}
```

再定义 `filter` 方法，它会让 `next` 只返回符合判断条件的值：

```js
class Lazy(iterator, callback) {
    // ...其它细节
   filter(callback) {
    return new LazyFilter(this, callback);
  }
    // ...其它细节
}

class LazyFilter extends Lazy {
  next() {
    while (true) {
      const item = this.iterable.next();

      if (this.callback(item.value)) {
        return item;
      }
    }
  }
}
```

最后，定义 `takeWhile`，它会限制 `next` 执行的条件，一旦条件不满足，则停止执行 `next` 并返回历史执行结果：

```js
takeWhile(callback) {
    const result = [];
    let value = this.next().value;
    while (callback(value)) {
      result.push(value);
      value = this.next().value;
    }
    return result;
  }
```

主要的方法都定义完了，现在把它们合并起来：

```js
class Lazy {
  constructor(iterable, callback) {
    this.iterable = iterable;
    this.callback = callback;
  }

  filter(callback) {
    return new LazyFilter(this, callback);
  }

  map(callback) {
    return new LazyMap(this, callback);
  }

  next() {
    return this.iterable.next();
  }

  takeWhile(callback) {
    const result = [];
    let value = this.next().value;
    while (callback(value)) {
      result.push(value);
      value = this.next().value;
    }
    return result;
  }
}

class LazyFilter extends Lazy {
  next() {
    while (true) {
      const item = this.iterable.next();

      if (this.callback(item.value)) {
        return item;
      }
    }
  }
}

class LazyMap extends Lazy {
  next() {
    const item = this.iterable.next();

    const mappedValue = this.callback(item.value);
    return { value: mappedValue, done: item.done };
  }
}

const numbers = function*() {
  let i = 1;
  while (true) {
    yield i++;
  }
};
```

现在用我们写的 `Lazy` 和 `numbers` 函数来实现文章开头的 Haskell 代码：

```js
new Lazy(numbers())
  .map(x => x ** 2)
  .filter(x => x % 2 === 1)
  .takeWhile(x => x < 10000)
  .reduce((x, y) => x + y);
  // => 16650
```