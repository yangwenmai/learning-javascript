# 函数式编程能干什么（一）-- 写个烧脑异步控制流

这次要写的小项目灵感源自月影老师的[漫谈 JS 函数式编程（一）](https://www.h5jun.com/post/js-functional-1.html)。月影老师在文章末尾演示了借助 Ramda 的 `lift` 函数，实现一个烧脑异步效果。几个月前读到月影老师的这篇文章时，我正学到 Applicative Functor，看到了 `lift` 函数，于是决定自己写一个，然后花了一个上午写出来了。

在这篇文章里我将会解释几个本次项目会用到的函数式编程概念，然后深入具体代码逐行解释。函数式编程的核心概念基本都来自范畴论，而范畴论确实比较难懂，所以我在解释时就尽量少涉及严谨理论，而是用灵活的类比来帮助大家理解。如果你去谷歌一些范畴论的概念，你能找到的定义大部分都过于学术化。比如你想知道 Applicative Functor 是什么，维基百科会告诉你那是介于 Functor 和 Monad 之间的数据类型。然后你又顺藤摸瓜去看什么是 Monad，到这一步你基本就没学下去的勇气了。

开始之前说明一下，我即将展示的代码真的很难懂，我尽量解释。我的前两篇发表在掘金的文章都有人批评不够接地气，感觉是在炫。我本意非如此，我开始学编程还没多久，我真的只是在通过写作的方式加深自己的理解。如果你觉得为了实现同样的效果，没必要写这么难懂的代码，欢迎给出你的方案。我是真不知道其它写法该怎么写，欢迎大家交流。

## 一，两个范畴论概念

**1. Functor**

我能想到的最简单定义：如果一个数据类型知道怎样把它的子元素映射到其它值，那它就是个 `Functor`。说人话，比如给个数组`[3]`，我们可以把它的子元素映射到其它值: `[3].map(x => x + 1)`，返回的结果是`[4]`。很显然，JS 里的数组就是个 `Functor`。当然 `Functor` 有很多，数组只是大家最熟悉的一个。这篇文章里，为了避免引入不必要的复杂度，我就拿数组当做 `Functor` 来用。

**2. Applicative Functor**

可能是全网最简单的定义：Applicative Functor 就是由映射函数组成的 Functor。来看这个 Functor（从现在开始我们就不叫它数组了）：`const fns = [x => x + 1, x => x + 2, x => x + 3]` ，所谓映射函数就是把一个值映射到另一个值的函数…… 那这个 Applicative Functor 是干嘛的呢？我们再创建个数组，哦不，Functor：`const xs = [1, 2, 3]`。然后这样：`fns.map(fn => xs.map(fn))`。你可以理解为把第一个 Funcor 遍历一遍，然后把每一个函数当做第二个 Functor 的每一个元素的映射函数。啊，解释还是太累，看计算结果吧。上一步计算的结果是`[[2, 3, 4], [3, 4, 5] [4, 5, 6]]`

## 二，`lift` 函数

我们刚刚其实已经定义了一个很重要的函数 `ap`,  它其实就是把 Applictive Functor 作用于其它 Functor 的函数：`const ap = fns => xs => fns.map(f => xs.map(f));` 但这样是不够的，因为经过这次运算，我们得到的是一个二维数组，我们得把结果转成一维数组。先来定义个工具函数来把二维数组转成一维数组：
```js
const flatten = arr => arr.reduce((flat, next) => flat.concat(next), []);
```
最后的 `ap` 函数定义为：
```js
const ap = fns => xs => flatten(fns.map(f => xs.map(f)));
```

所谓 `lift` 就是把一个柯里化的函数提升到 Applicative Functor，然后再把这个 Applicative Functor 作用于其它 Functor。来看定义：
```js
const lift = f => (head, ...tail) =>
  tail.reduce((fns, xs) => ap(fns)(xs), head.map(f));
```
举个例子。先给 `lift` 传个函数 `const add = x => y => x + y`，为了方便，我们只给 `lift` 传两个 Functor：`const arr1 = [1, 2, 3, 4]; const arr2 = [4, 3, 2, 1]` 然后 `reduce` 函数的第一步计算是 `[1, 2, 3, 4].map(x => y => x + y)`，结果是 `[y => y + 1, y => y + 2, y => y + 3, y => y + 4]`，接着，第二步计算是 `ap([y => y + 1, y => y + 2, y => y + 3, y => y + 4])([4, 3, 2, 1])`，计算结果是 `[ 5, 4, 3, 2, 6, 5, 4, 3, 7, 6, 5, 4, 8, 7, 6, 5 ]`。你可以理解为把两个数组进行嵌套 for 循环然后把每次循环元素相加。

## 三，一些辅助函数

函数式编程最重要的部分是函数组合，在这部分我来解释我们将要用到的辅助函数。

**partial:**

```js
function partial(func, argArr) {
  return function() {
    const allArguments = argArr.concat(Array.prototype.slice.call(arguments));
    return func.apply(this, allArguments);
  };
}
```
Partial Application 也是函数式编程中比较重要的概念。它的意思是，给一个函数所需的部分参数，这个函数会接收这些参数，并返回一个新函数，等你给它传剩下的参数，等你把剩下的参数传进去，函数才会执行。如果你用 Haskell 等函数式编程语言，你是不需要这个函数的，因为 Haskell 默认所有函数都柯里化了。

**chunk:**

```js
const chunk = (array, size) =>
  array.reduce((chunked, item) => {
    const last = chunked[chunked.length - 1];
    if (!last || last.length === size) {
        chunked.push([item]);
        return chunked;
    }
    last.push(item);
    return chunked;
  }, []);
```
这个不用多解释了，`chunk` 函数会把一个一维数组转换成二维数组，目标二维数组的子数组长度可定制。

**asyncPipe:**

```js
const asyncPipe = (...fns) => x => fns.reduce(async (y, f) => f(await y), x);
```

如果你看过我另一篇文章[如何在 JS 代码中消灭 for 循环](https://juejin.im/post/5b5a9451f265da0f6a036346)，你可能记得我写过一个 `pipe` 函数：
```js
const pipe = (...fns) => (...args) => fns.reduce((fx, fy) => fy(fx), ...args);
```
`pipe` 函数是非常重要的一个函数，它是用来函数组合的。给它传入多个函数，它会把传入的函数从左到右一个个执行，并把前一个函数的执行结果传给下一个函数。提个题外话，EcmaScript 可能会原生支持 `pipe` 操作符，它长这样 `|>`，目前这个提案到了 `stage 1`，我非常希望它最终能进入语言标准。

`asyncPipe` 基本和 `pipe` 长一样，区别是它接受多个异步函数，逐个执行，当前函数执行完后，再把结果传给下一个函数，依次从左往右执行。

**wait:**
```js
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));
```
给 `wait` 函数传入毫秒数，它会让程序暂停指定毫秒时间，再继续执行函数。

## 四，开始写逻辑

1. 首先我们要定义对指定某 DOM 节点，等待某段时间后，改变颜色这个行为：
```js
const setColor = async (node, color) => {
  await wait(500);
  node.style.backgroundColor = color;
};
```

这里我们等待 500ms 后再变色。

2. 取到我们想变色的节点：
```js
const items = Array.from(document.querySelectorAll("#list > li > span"));
```

3. 指定我们想变的颜色：
```js
const color = ["red", "#ffa600", "#224de3", "#eee"];
```

4. 提供我们想要的目标属性组合：
```js
const combine = node => color => [node, color];
```

5. 提供我们想要的行为组合：
```js
const tasks = lift(combine)(items, color).map(comb => partial(setColor, comb));
```

注意，上一步定义的任务组合，并没有区分不同灯，而我们目标是把每个灯变四个颜色，等待某段时间后，再开始下一个灯的变色。所以，我们需要把上一个任务组合拆成由多个子任务，每个子任务代表一盏灯，每盏灯变4次颜色：

```js
const chunkedTasks = chunk(tasks, 4);
```

6. 然后，我们就能写主要逻辑了：
```js
async function run(pause) {
  for (let i = 0; i < chunkedTasks.length; i++) {
    await asyncPipe(...chunkedTasks[i])();
    await wait(pause);
  }

  return run(pause);
}
```

7. 最后执行主逻辑，并传入每盏灯之间的等待时间：
```js
run(1000);
```

8. 优化。主逻辑为了实现循环不停执行变色，使用了递归。为了避免递归爆栈，我们使用 `trampoline` 函数来避免每次递归都新增执行帧。
```js
const trampoline = fn => (...args) => {
  let result = fn(...args);
  while (typeof result === "function") {
    result = result();
  }
  return result;
};

trampoline(run)(1000);
```

至此，全部逻辑就写完了。

比较我这个版本的效果和原版的效果，不同在于：

1. 没借助第三方库，全手写辅助函数。
2. 每盏灯之间和每次变色之间的时间都能定制，而原版本中，每盏灯和每次变色之间的时间都是相同的。
3. 所有灯变色完成后，回到第一盏灯，无限循环。

完整代码和效果在[此处](https://jsfiddle.net/leihuang/nrcexgvL/13/)

## 四，更变态的需求

现在我们实现了比原版本要求更高的需求，但是到这里就完了吗？不不不，还有更有挑战的。如果我想要让每盏灯不同颜色之间的等待时间可随意定制，然后让每盏灯之间的等待时间也可随意定制，我们现在的代码结构能做到吗？现在没时间继续写，敬请期待……
