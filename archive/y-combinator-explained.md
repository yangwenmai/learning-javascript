# 你可能还不懂 Y Combinator

我在之前的一篇文章[《不懂递归？读完这篇保证你懂》](https://juejin.im/post/5b5bce21f265da0f6b7710fb)最后扔了一段代码没有解释。当时是略微懂，但是还是不能说很明白，现在读了更多资料之后再尝试着解释一下。那段代码用 ES6 可以简写如下：

```js
const Y = f => (g => g(g))(g => f(x => g(g)(x)));
```

如果你第一次读这段代码，你可能看半天也不知道这究竟是个什么玩意（我就是这样）。现在我就来一步步拆解这个函数，看能不能让你从一头雾水到略懂（完全懂还是有些困难）。

上面那段代码叫 Y Combinator。为了理解它，我们要先从 Lambda 演算说起。

## 1. Lamda 演算
Lambda 演算 (lambda calculus) 是阿伦佐·丘奇（Alonzo Church）发明的一种用来表达数学计算的逻辑语言。它由三个部分构成：1. 变量绑定。2. 函数定义。3. 函数应用（其实就是把定义好的函数进行调用啦）。用 JS 来表达，最简单的 Lambda 演算是 `identity` 函数：
```js
const identity = x => x;

identity(1) // -> 1
```
没有更多内容了，就这么简单。不要小看了这么简单的逻辑，lambda 演算是图灵完整的！你可能还不相信，心里嘀咕着，既然它图灵完整，那你用它写个 for 循环试试？本文的目的，就是既不用迭代，也不用递归，来实现循环。

## 2. U Combinator
先来看这个 Lambda 演算 (它叫 U Combinator)：
```
U = λg.(g g)
```
它的意思是，给 λ 传入一个函数 g，它会返回 g 作用于 g 的结果。翻译到 JS 就是：
```js
const U = g => g(g)
```
函数 U 接受一个函数为参数，返回的结果是把这个函数作用于自身。把上面定义的 `identity` 函数传入 U 执行： `U(identity)`，`identity` 函数会永远不停重复被调用，直到爆栈。U Combinator 体现了用 Lambda 演算实现循环的最核心概念 —— 自我应用（self-application）。把一个函数作为参数传给自己，不停执行，就实现了循环。

## 3. 拆解 Y Combinator

我们先忘掉一开始的那段神秘 JS 代码，回到 Lambda 演算。下面是 Haskell Curry （这位大神的名成了名词 -- 编程语言 Haskell，姓成了动词 -- 函数柯里化，currying ）发明的 Y Combinator：
```
Y = λf.(λx.f (x x)) (λx.f (x x))
```

这个定义的意思是，λ 接受函数 f 为参数，返回一个匿名函数的自我应用。这个匿名函数是 `λx.f (x x)`, 意思是接受 x 为参数，把 x 作用于 x，再把结果传给 f。匿名函数 `λx.f (x x)` 以自己作为实参，进行计算。

把上面的 Lambda 演算翻译成 JS，就是这样：

```js
const Y = f => (g => f(x => g(g)(x)))(g => f(x => g(g)(x)));
```

最后部分左边可以简化成 `g => g(g)`，因为右边作为参数传进来后，效果是一样的，于是我们得到了文章一开始的那段定义：

```js
const Y = f => (g => g(g))(g => f(x => g(g)(x)));
```

## 4. 应用 Y Combinator
其实在 JS 里面根本用不到 Y Combinator，因为 JS 已经可以实现迭代和递归了。学 Lambda 演算纯是强化 CS 基础和更深刻理解函数式编程。但是我们既然都已经学了，还是用它来干些事吧。我已经在之前的文章里用 Y Combinator 实现了 factorial 函数，这里就不演示了。这次演示一个稍微“实用”点的例子。

我们要实现的一个效果是，在网页加载时弹出提示框，要求用户接受条款（输入 yes），如果用户没有输入 yes，继续不停弹，直到用户输入 yes 并提交为止。用户输入 yes 后，再弹提示，显示用户拒绝了多少次。

代码太简单了就直接全上了：
```js
const Y = f => (g => g(g))(g => f(x => g(g)(x)));

Y(ask => count =>
  prompt("Accept terms?") !== "yes"
    ? ask(count + 1)
    : alert("At last! It took " + count + " times.")
)(1);
```

效果见这里：[CodePen](https://codepen.io/leihuang/pen/QVEGPo)

**参考：**
1. [Fixed-point combinators in JavaScript: Memoizing recursive functions](http://matt.might.net/articles/implementation-of-recursive-fixed-point-y-combinator-in-javascript-for-memoization/)
2. [Essentials: Functional Programming's Y Combinator - Computerphile](https://www.youtube.com/watch?v=9T8A89jgeTI)
3. [Lambda Calculus - Computerphile](https://www.youtube.com/watch?v=eis11j_iGMs)