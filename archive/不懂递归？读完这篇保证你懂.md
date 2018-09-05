# 不懂递归？读完这篇保证你懂

今年年初我开始学 Haskell 的时候，我被函数式代码的优雅和简洁俘获了。代码居然还能这样写！用指令式代码要写一堆的程序，用递归几行就解决了。这篇文章里，我会把我在 Haskell 里面看到的递归函数翻译成 JS 和 Python，并尽量每一步解释。最后我会尝试解决递归爆栈（Stack Overflow）的问题。

## 递归基础

我从 Python 代码开始，然后展示 JS 实现。

很多解释递归的教程是从解释斐波那契数列开始的，我觉得这样做是在用一个已经复杂的概念去解释另一个复杂的概念，没有必要。我们还是从简单的代码开始吧。

运行这段 Python 代码：

```python
def foo():
    foo()

foo()
```

当然会报错。😱 `foo` 函数会一直调用自己。因为我没有告诉它何时停，它会一直执行下去，直到爆栈。那我们稍作修改再运行一下：

```python
def foo(n):
    if n <= 1:
        return
    foo(n-1)

foo(10)
```

这段代码基本什么都没做，但是这次它不会报错了。我在 `foo` 函数定义初始就告诉它什么时候该停，然后我每次调用的时候都把参数改一下，直到参数满足判断条件，函数停止执行。

如果你理解了上面两段代码，你已经理解递归了。

从上面的代码我总结一下递归的核心构成：

1.  递归函数必须接受参数。
2.  在递归函数的定义初始，应该有一个判断条件，当参数满足这个条件的时候，函数停止执行，并返回值。
3.  每次递归函数执行自己的时候，都需要把当前参数做某种修改，然后传入下一次递归。当参数被累积修改到符合初始判断条件了，递归就停止了。

现在我们来用 Python 写个 max 函数，找出列表中的最大值。是的，我知道 Python 原生有 max 函数，我重新发明个轮子只是为了学习和好玩。

```python
# 不要用这个函数，还是用原生的 max 吧。
def max2(list):
    if len(list) == 1:
         return list[0]
    head, tail = list[0], list[1:]
    return head if head > max2(tail) else max2(tail)

print max2([3,98,345])
# 345
```

`max2`函数接受一个列表作为参数，如果列表长度为 1，函数停止执行并把列表第一个元素返回出去。注意，当递归停止时，它必须返回值。（但是如果你想用递归去执行副作用，而不是纯计算的话，可以不返回值。）如果初始判断条件不满足，把列表的头和尾取出来。接着，我们比较头部元素和尾部列表中最大值的大小（我们先不管尾部列表中最大值是哪个），并把比较结果中更大的那个值返回出去。那我们怎样知道尾部列表中的最大值？答案是我们不用知道。我们已经在 `max2` 函数中定义了比较一个列表的第一个元素和剩下元素中的最大值，并把较大的值返回出去这个行为了。我们只需要把这同一个行为作用于尾部列表，程序会帮我们找到。

下面是 JS 的实现：

```js
const max = xs => {
  if (xs.length === 1) return xs[0];
  const [head, ...tail] = xs;
  return head > max(tail) ? head : max(tail);
};
```

## 更多递归的例子

接下来我展示几个我从 Haskell 翻译过来的递归函数。刚刚已经用很大篇幅解释递归了，这些函数就不解释了。

**reverse**

Python 版：

```python
# Python 内置有 reverse 函数
def reverse2(list):
    if len(list) == 1:
        return list
    head, tail = list[0], list[1:]
    return reverse2(tail) + [head]

print reverse2([1,2,3,4,5,6])
# [6,5,4,3,2,1]
```

JS 版：

```js
const reverse = xs => {
  if (xs.length === 1) return xs;
  const [head, ...tail] = xs;
  return reverse(tail).concat(head);
};
```

**map**

Python 版：

```python
# Python 内置有 map 函数
def map2(f, list):
    if len(list) == 0:
        return []
    head, tail = list[0], list[1:]
    return [f(head)] + map2(f, tail)

print map2(lambda x : x + 1, [2,2,2,2])
# [3,3,3,3]
```

JS 版：

```js
const map = f => xs => {
  if (xs.length === 0) return [];
  const [head, ...tail] = xs;
  return [f(head), ...map(f)(tail)];
};
```

**zipWith**

`zipWith` 接受一个回调函数和两个列表为参数。他会并行遍历两个列表，并把遍历到的元素一一对应，传进回调函数，把每一步遍历的计算结果存在新的列表里，最终返回这个新列表。

Python 版：

```python
def zipWith(f, listA, listB):
    if len(listA) == 0 or len(listB) == 0:
        return []
    headA, tailA = listA[0], listA[1:]
    headB, tailB = listB[0], listB[1:]
    return [f(headA, headB)] + zipWith(f, tailA, tailB)

print zipWith(lambda x, y : x + y, [2,2,2,2], [3,3,3,3,3])
# [5,5,5,5]
# 结果列表长度由参数中两个列表更短的那个决定
```

JS 版：

```js
const zipWith = f => xs => ys => {
  if (xs.length === 0 || ys.length === 0) return [];
  const [headX, ...tailX] = xs;
  const [headY, ...tailY] = ys;
  return [f(headX)(headY), ...zipWith(f)(tailX)(tailY)];
};
```

**replicate**

Python 版：

```python
def replicate(n,x):
    if n <= 0:
        return []
    return [x] + replicate(n-1,x)

print replicate(4, 'hello')
# ['hello', 'hello', 'hello', 'hello']
```

JS 版：

```js
const replicate = (n, x) => {
  if (n <= 0) return [];
  return [x, ...replicate(n - 1, x)];
};
```

**reduce**

Python 不鼓励用 reduce，我就不写了。

JS 版：

```js
const reduce = (f, acc, arr) => {
  if (arr.length === 0) return acc;
  const [head, ...tail] = arr;
  return reduce(f, f(head, acc), tail);
};
```

**quickSort**

用递归来实现排序算法肯定不是最优的，但是如果处理数据量小的话，也不是不能用。

Python 版：

```python
def quickSort(xs):
    if len(xs) <= 1:
         return xs
    pivot, rest = xs[0], xs[1:]
    smaller, bigger = [], []
    for x in rest:
        smaller.append(x) if x < pivot else bigger.append(x)
    return quickSort(smaller) + [pivot] + quickSort(bigger)

print quickSort([44,14,65,34])
# [14, 34, 44, 65]
```

JS 版：

```js
const quickSort = list => {
  if (list.length === 0) return list;
  const [pivot, ...rest] = list;
  const smaller = [];
  const bigger = [];
  rest.forEach(x => (x < pivot ? smaller.push(x) : bigger.push(x)));

  return [...quickSort(smaller), pivot, ...quickSort(bigger)];
};
```

## 解决递归爆栈问题

由于我对 Python 还不是特别熟，这个问题只讲 JS 场景了，抱歉。

每次递归时，JS 引擎都会生成新的 frame 分配给当前执行函数，当递归层次太深时，可能会栈不够用，导致爆栈。ES6引入了尾部优化（TCO），即当递归处于尾部调用时，JS 引擎会把每次递归的函数放在同一个 frame 里面，不新增 frame，这样就解决了爆栈问题。

然而，V8 引擎在短暂支持 TCO 之后，放弃支持了。那为了避免爆栈，我们只能在程序层面解决问题了。 为了解决这个问题，大神们发明出了 `trampoline` 这个函数。来看代码：

```js
const trampoline = fn => (...args) => {
  let result = fn(...args);
  while (typeof result === "function") {
    result = result();
  }
  return result;
};
```

给`trampoline`传个递归函数，它会把递归函数的每次递归计算结果保存下来，然后只要递归没结束，它就不停执行每次递归返回的函数。这样做相当于把多次的函数调用放到一次函数调用里了，不会新增 frame，当然也不会爆栈。

注意，为了配合 `trampoline` 函数，我们应该让递归函数返回一个函数。比如，正常递归是这样：

```js
const factorial = (n, total) => {
  if (n === 1) return total;
  return factorial(n - 1, n * total);
};
```

配合 `trampoline` 要改成：

```js
const factorial = (n, total) => {
  if (n === 1) return total;
  return () => factorial(n - 1, n * total);
};
```

先别高兴太早。仔细看 `trampoline` 函数的话，你会发现它也要求传入的递归函数符合尾部调用的情况。那不符合尾部调用的递归函数怎么办呢？（ 比如我刚刚写的 JS 版 `quickSort`，最后 return 的结果里，把两个递归调用放在了一个结果里。这种情况叫 **binary recursion**，暂译二元递归，翻译错了勿怪 ）

这个问题我也纠结了很久了，然后直接去 Stack Overflow 问了，然后真有大神回答了。要解决把二元递归转换成尾部调用，需要用到一种叫 Continuous Passing Style (CPS) 的技巧。来看怎么把 `quickSort` 转成尾部调用:

```js
const identity = x => x;

const quickSort = (list, cont = identity) => {
  if (list.length === 0) return cont(list);

  const [pivot, ...rest] = list;
  const smaller = [];
  const bigger = [];
  rest.forEach(x => (x < pivot ? smaller.push(x) : bigger.push(x)));

  return quickSort(smaller, a =>
    quickSort(bigger, b => cont([...a, pivot, ...b])),
  );
};

tramploline(quickSort)([5, 1, 4, 3, 2]) // -> [1, 2, 3, 4, 5]
```

如果上面的写法难以理解，推荐去看 Kyle Simpson 的[这章内容](https://github.com/getify/Functional-Light-JS/blob/master/manuscript/ch8.md/#chapter-8-recursion)。我不能保证比他讲的更清楚，就不讲了。

## 屠龙之技

虽然我将要讲的这个概念在 JS 中根本都用不到，但是我觉得很好玩，就加进来了。如果既不用迭代，也不用递归，能不能实现循环？ 答案是可以的，用 Y Combinator.

JS 实现：

```js
function y(le) {
  return (function(f) {
    return f(f);
  })(function(f) {
    return le(function(x) {
      return f(f)(x);
    });
  });
}

const factorial = y(function(fac) {
  return function(n) {
    return n <= 2 ? n : n * fac(n - 1);
  };
});

factorial(5); // 120
```

看到这段代码时我感到很激动，惊叹计算机科学的精妙和优雅。很多人把程序员职业当做是搬砖的，但是我不这么看。我在学习 CS 的过程中感受更多的是人类智力的不可思议和计算机科学中体现的普遍认识论规律。

如果想了解更多关于 Y Combinator，参考我[另一篇文章](https://juejin.im/post/5b853c4b6fb9a01a157296c0).
