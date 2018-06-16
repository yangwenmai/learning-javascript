# How To Write Recursion Like A Pro

When I started learning programming, one of the most confusing concept to understand is recursion. Now, when I'm finally the pro programmer who I'd been dreaming of becoming when I was a newbie, I feel like that these confusions and struggles are unnecessary. If I had a teacher like the present me, I would have learned it easily! Recently, as I'm learning Haskell, I get enamored by how elegant coding could be in Haskell, especially when writing recursion. In this post, I'll show you how to write recursion functions step by step. In the end, I'll show you some recursion functions that I translated from Haskell, both in JavaScript and in Python!

## Recursion basics

I'll start by explaining things in Python, then I'll implement the same logic in JavaScript.

Many tutorials usually explain recursion by introducing Fibonacci numbers. I think that's counter-productive. We don't need another complicated concept to explain an already complicated concept. I'll start with some common reasoning.

Let's take a look at this piece of code:

```python
def foo():
    foo()

foo()
```

Open your terminal or any REPL editors, and run this code. You'll get an error! Don't be angry with me yet. This is to be expected. Let's reason about this code for a bit. This `foo` function was defined to call itself, since there's no command to tell it to stop calling itself, it will keep calling itself infinitely, until the runtime quits with stack overflow, which, will be translated to human language as "maximum recursion depth exceeded" in Python runtime.

Let's modify the code to make it stop after meeting a condition:

```python
def foo(n):
    if n <= 1:
        return
    foo(n-1)

foo(10)
```

This code does nothing, but this time it will not throw an error!

There are three major changes that should draw your attention in the second `foo` function:

1. The `foo` function takes an argument.
2. In the beginning of the function, there is a condition against which the argument will be checked. If the condition is met, the function stops running.
3. Each time the function calls itself, it will modify the argument.

Believe it or not, that's all recursion is about! We'll consolidate our understanding by writing some recursion functions. All of the functions I'm going to present you are inspired by Haskell!

Let's first write our own version of `max` in Python. I suggest you use the built-in `max` method in your code, this is just for practice.

```python
def max2(list):
    if len(list) == 1:
         return list[0]
    head, tail = list[0], list[1:]
    return head if head > max2(tail) else max2(tail)

print max2([3,98,345])
# 345
```

The `max2` function takes a list. If the length of the list is 1, the function will stop running and yield the first element of the list. Notice that when a recursion stops, it must yield a value. (If the purpose of your function is to perform side effects rather than calculation, then you can choose to not yield a value.) Otherwise, we'll take the first element from the list and name it head, and name the rest of the list tail.

We compare head with the largest element from the tail list, which we don't know yet, so we put the tail to the `max2` function again. We don't care how the rest calls to `max2` anymore, because we know they will be stopped and yield a value. If the head is bigger than the max number from the tail list, then the head is the max number of the original list. Otherwise, the result of `max2(tail)` is the max number.

If this is still confusing to you, I suggest you write the process down and observe each step.

Here's the JavaScript version:

```javascript
const max = xs => {
  if (xs.length === 1) return xs[0]
  const [head, ...tail] = xs
  return head > max(tail) ? head : max(tail)
}
```

## More recursions

`reverse` takes a list as an argument, and returns a new list with the order of all elements being reversed.

```python
# python
# Python has a built-in reverse,
# so I name this function reverse2
def reverse2(list):
    if len(list) == 1:
        return list
    head, tail = list[0], list[1:]
    return reverse2(tail) + [x]

print reverse2([1,2,3,4,5,6])
# [6,5,4,3,2,1]
```

```javascript
// JavaScript
const reverse = xs => {
  if (xs.length === 1) return xs
  const [head, ...tail] = xs
  return reverse(tail).concat(head)
}
```

`map` takes a function and a list, applies the function to each element of the list and returns the result.

```python
# python
# Python has a built-in map,
# so I name this function map2
def map2(f, list):
    if len(list) == 0:
        return []
    head, tail = list[0], list[1:]
    return [f(head)] + map2(f, tail)

print map2(lambda x : x + 1, [2,2,2,2])
# [3,3,3,3]
```

```javascript
// JavaScript
const map = f => xs => {
  if (xs.length === 0) return []
  const [head, ...tail] = xs
  return [f(head), ...map(f)(tail)]
}
```

`zipWith` takes a function and two lists, it iterates over these two lists in parallel and zips each element from two lists together in every iteration, with the provided function being applied to each zip.

```python
# Python
def zipWith(f, listA, listB):
    if len(listA) == 0 or len(listB) == 0:
        return []
    headA, tailA = listA[0], listA[1:]
    headB, tailB = listB[0], listB[1:]
    return [f(headA, headB)] + zipWith(f, tailA, tailB)

print zipWith(lambda x, y : x + y, [2,2,2,2], [3,3,3,3,3])
# [5,5,5,5]
# The result list will only be as long as the shorter list
```

```javascript
// JavaScript
const zipWith = f => xs => ys => {
  if (xs.length === 0 || ys.length === 0) return []
  const [headX, ...tailX] = xs
  const [headY, ...tailY] = ys
  return [f(headX)(headY), ...zipWith(f)(tailX)(tailY)]
}
```

`replicate` takes a number and an arbitrary element, and returns a list with the length of the number, with each element being that arbitrary element.

```python
# Python
def replicate(n,x):
    if n <= 0:
        return []
    return [x] + replicate(n-1,x)

print replicate(4, 'hello')
# ['hello', 'hello', 'hello', 'hello']
```

```javascript
// JavaScript
const replicate = (n, x) => {
  if (n <= 0) return []
  return [x, ...replicate(n - 1, x)]
}
```

`filter` takes a predicate function and a list, and returns a new list that filters out all the elements that fail to pass the predicate function.

```python
# Python
# Python has a built-in filter,
# so I name this function filter2
def filter2(f, list):
    if len(list) == 0:
        return []
    head, tail = list[0], list[1:]
    return [head] + filter2(f, tail) if f(head) else filter2(f, tail)

print filter2(lambda x : x > 4, [3,2,4,5,6])
# [5,6]
```

```javascript
const filter = (f, list) => {
  if (list.length === 0) return []
  const [head, ...tail] = list
  return f(head) ? [head, ...filter(f, tail)] : filter(f, tail)
}
```

Here's the most complicated one so far. Quick sort in a very common sorting algorithm in computer science. This `quickSort` function takes a list of numbers, and returns a list of numbers sorted with ascending order. The function first takes out the first number of the list and marks it as a pivot, then it iterates over the rest of the list, comparing each element number with the pivot number. If the element number is smaller, it will be appended to the new list called smaller, otherwise, it will be appended to another new list called bigger. Then the function enters recursion, it applies `quickSort` to both the smaller list and the bigger list, and concatenates the results with pivot, with pivot in the middle. The smaller list and bigger list will be divided in every recursion, until the length of them reaches one, upon which the recursion stops and returns the remaining list.

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

```javascript
const quickSort = list => {
  if (list.length === 0) return list
  const [pivot, ...rest] = list
  const smaller = []
  const bigger = []
  for (x of rest) {
    x < pivot ? smaller.push(x) : bigger.push(x)
  }

  return quickSort(smaller)
    .concat(pivot)
    .concat(quickSort(bigger))
}
```