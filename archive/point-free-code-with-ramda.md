# 优雅代码指北 -- 巧用 Ramda

这次给大家介绍一个我用的最多的工具函数库 Ramda，展示怎样用 Ramda 写出既简洁易读，又方便扩展复用的代码。由于时间和精力有限，我就不解释我用到的每一个 Ramda 函数的用法了，大家感兴趣的话可以去查官方文档。

Ramda 有两个特性让它从其它工具库中脱颖而出：

1. 所有 Ramda 函数都已经被柯里化。
2. 所有 Ramda 函数都把数据作为最后一个参数传入。

这两个特征让我们可以很轻松利用 Ramda 写出 "point free" 风格的代码。所谓 point 就是指作为参数传进函数的数据。point free 就是脱离数据的代码风格。通过做到 point free，我们做到了**行为和数据的分离**，这利于我们写出更安全（组合行为时没有副作用），更易扩展（脱离数据的逻辑容易复用），和更易理解（读高阶函数的组合就像读普通英文一样）的代码。


## 一，第一个 point free 例子
如果你写过 React + Redux 项目，你可能经常写这种代码：
```js
function mapStateToProps(state) {
  return {
    board: state.board,
    nextToken: state.nextToken
  }
}
```
我们可以用 Ramda 的 `pick` 函数改写一下:
```js
import { pick } from 'ramda';

function mapStateToProps(state) {
  return pick(['board', 'nextToken'], state)
}
```
继续改写成 point free 风格，可以写成这样：
```js
const mapStateToProps = pick(['board', 'nextToken']);
```
是不是简洁了很多？

## 二，函数组合

**问题：** 有这样一个数组：
```js
const teams = [
  {name: 'Lions', score: 5},
  {name: 'Tigers', score: 4},
  {name: 'Bears', score: 6},
  {name: 'Monkeys', score: 2},
]
```
要求找出分数最高的小组，并取到名字。

**答案：**
```js
import { compose, head, sort, descend, prop } from "ramda";

const teams = [
    {name: 'Lions', score: 5},
    {name: 'Tigers', score: 4},
    {name: 'Bears', score: 6},
    {name: 'Monkeys', score: 2},
  ];
  
const sortByScoreDesc = sort(descend(prop("score")));
const getName = prop("name");
const findTheBestTeam = compose(
  getName,
  head,
  sortByScoreDesc
);

findTheBestTeam(teams) // => Bears
```
稍微感受一下。数据是在最后一步才提供的，提供目标数据之前一直在组合行为，没有改变数据，没有任何副作用。注意 `compose` 是从后往前组合函数，如果习惯从前往后组合函数，用 `pipe`。

再来一个：

**问题：**
把下面这个查询字符串转成对象：
```js
const queryString = "?page=2&pageSize=10&total=203";
```
**答案：**
```js
import { compose, fromPairs, map, split, tail } from "ramda";

const queryString = "?page=2&pageSize=10&total=203";

const parseQs = compose(
  fromPairs,
  map(split("=")),
  split("&"),
  tail
);

const result = parseQs(queryString); // => { page: '2', pageSize: '10', total: '203' }
```

你可能会问，JS 原生都提供 `map` 方法了，为什么还有用 Ramda 的？ 原因是文章开头提到的，Ramda 函数有两个特厉害的属性。这个例子里，给 `map` 传一个回调函数，它会返回一个新函数，等你给它传数据。

## 三，一个数据用两次，怎么 point free？

有时候会遇到这种情景，根据一个数据算出结果，再根据相同数据算出另一个结果，然后把两个结果进行某种运算。比如这个简单例子：

**问题：** 给定一个用户对象，根据用户 id 生成头像地址，并把地址合并到用户对象上。
```js
// 合并前
const user = {
  id: 1,
  name: 'Joe'
}

// 合并后

{
    id: 1,
    name: 'Joe',
    avatar: 'https://img.socialnetwork.com/avatar/1.png'
}
```

**答案一：**
```js
const generateUrl = id => `https://img.socialnetwork.com/avatar/${id || 'default'}.png`;
const getUpdatedUser = user => ({ ...user, avatar: generateUrl(user.id) });

getUpdatedUser(user);
```

这个方案已经足够简洁，但是并没有达到 point free 的要求。数据 user 提前出现了，而我们期待的是在函数组合时不关心数据，哪怕是作为参数。但是，数据在计算过程中需要多次用到，怎样在没有数据（连代表数据的参数都没有）的情况下表达对未来数据的多次操作？Ramda 提供的 `converge` 函数可以解决这个问题：

**答案二：**
```js
import { compose, converge, propOr, assoc, identity } from "ramda";
const user = {
  id: 1,
  name: "Joe"
};
const generateUrl = id => `https://img.socialnetwork.com/avatar/${id}.png`;

const getUrlFromUser = compose(
  generateUrl,
  propOr("default", "id")
);
const getUpdatedUser = converge(assoc("avatar"), [
  getUrlFromUser,
  identity
]);

getUpdatedUser(user)
```

`converge` 函数接受两个参数，第一个参数是最终执行的函数，第二个参数是由作用于传入数据的变形函数组成的数组。在这个例子里，`user` 数组先分别传给 `identity` 和 `getUrlFromUser` 函数，然后把这两个函数的计算结果分别传给 `assoc("avatar")`。`identity` 可能是最无聊的函数，它长这样：
```js
const identity = x => x;
```
我们要保留 `user` 数据不动，然后传给 `assoc("avatar")` 作为第二个参数，所以用了 `identity`。

## 四，方法和数据耦合在一起，怎么 point free ？

有些时候方法就在数据上。比如用 jQuery 选中 DOM 元素后，对 DOM 元素进行操作的方法。假设 DOM 上有个 `<div id = "el1"></div>`，用 jQuery 选中元素后，执行某个动画效果：
```js
 $('#el1')
   .animate({left:'250px'})
   .animate({left:'10px'})
   .slideUp()
```

jQuery 的方法全在选中 DOM 元素后生成的对象上，方法是没法离开数据的。但这并不影响我们在数据还没给到之前组合行为。Ramda 提供了 `invoker` 函数解决类似问题：
```js
import { invoker, compose, constructN } from "ramda";

const animate = invoker(1, "animate");
const slide = invoker(0, "slideUp");
const jq = constructN(1, $);

const animateDiv = compose(
  slide,
  animate({ left: "10px" }),
  animate({ left: "250px" }),
  jq
);

animateDiv("#el1");
animateDiv("#el2");
```
`invoker` 函数接受3个参数。第一个参数表示要在对象上执行的函数接受多少个参数，第二个参数表示要在对象上执行的函数的名字，第三个参数是目标对象。`constructN` 是用来实例化一个构造函数或者类。

## 五，强大的 lens
`lens` 是从函数式编程语言借来的一个概念，它相当于对某个属性的聚焦。比如 `lensProp('a')`，就是对 a 属性的聚焦，不管这个 a 属性由哪个对象提供。聚焦之后，我们可以很方便的读取属性（`view`）和改变属性（`over`）注意，Ramda 中所有改变值的操作都不是真的在原数据基础上改，而是返回改了指定属性的新值。

举个很简单的例子。
```js
import {lensProp, view, over, toUpper} from 'ramda';

const person = {
  firstName: 'Fred',
  lastName: 'Flintstone'
}

const fLens = lensProp('firstName')

const firstName = view(fLens, person) // => 'Fred'

const result = over(fLens, toUpper, person)
// => {firstName: 'FRED', lastName: 'Flintstone'}
```

上面例子还不能看出 `lens` 有什么用。来看下实际使用场景：

**问题：**

用 React 写一个简单 counter demo，点击 + 和 — 按钮时，计数器对应加 1 和减 1。

`lens` 用在 React 的 `setState` 里非常方便：

```js
import {inc, dec, lensProp, over} from 'ramda'

const countL = lensProp('count')
const transformCount = over(countL)
const incCount = transformCount(inc)
const decCount = transformCount(dec)

// ... 其它细节

state = {
  count: 0
}

increase = () => {
  this.setState(incCount);
};

decrease = () => {
  this.setState(decCount);
};

// ... 其它细节
```

`lens` 与 React 的配合，最能发挥作用的情景是在写函数式组件的时候。有兴趣可以参考这个 [Demo](https://codesandbox.io/s/843yn2j570)

## 六，更高阶的函数组合

前面提到的内容都是常规的函数组合，Ramda 还提供了 monad 的组合。抱歉要扔术语了，如果有时间，未来我可能会解释什么是 monad。大家常用的 Promise 就是个 monad，通过运行 Promise 得到值之后，你并不能在 Promise 外面操作值，而是必须在 `then` 方法里面处理。这就是 monad 的最大特征（它里层会返回同样的 Type，一层层嵌套，必须通过某种 flatMap 机制将里层的值取出）。

首先，最常见的是组合 Promise。来看例子。

假设我们先根据用户 email 地址请求得到用户信息，然后再根据用户 ID 得到用户粉丝数。
```js
// 获取用户信息的异步函数
const ajaxForUserInfo = userEmail => fetch(/* post request */); 

// 获取用户粉丝的异步函数
const ajaxForUserFollowers = id => fetch(/* post request*/);

const fetchUserFollowers = async userEmail => {
  const userInfo = await ajaxForUserInfo(userEmail);
  const userFollowers = await ajaxForUserFollowers(userInfo.id);
  return userFollowers;
};
```

用 Ramda 提供的 `composeP`，可以组合上面两个 Promise：

```js
import {composeP} from 'ramda';

const ajaxForUserFollowers = composeP(
  ajaxForUserFollowers,
  ajaxForUserInfo
);

const fetchUserFollowers = async email => {
  const userFollowers = await ajaxUserFollowers(email);
  return userFollowers;
};
```
上面例子只有两个 Promise 需要组合。如果是多个的话，组合的优势更明显。

**高能预警：** 

上面的内容已经足以覆盖大部分函数组合的需求。接下来要讲的一种函数组合算是比较硬核的函数式编程了，感兴趣的可以接着看，没兴趣的可以跳过了。

除了对 Promise 的组合，Ramda 还提供更泛的 monad 组合，叫 Kleisli Composition。暂时不用知道这玩意是什么，知道它是对各种 monad 的组合就行了。我们以 Maybe Monad 为例来看 Kleisli Composition：

可能读者在看到函数组合时会有疑问，如果某个函数有可能返回空值，还怎么组合？在每个后续函数前都做空值判断？那就真不优雅了。函数式编程提供了 Maybe Monad 进行空值处理，Maybe 可以和其它 monad 正常组合。

来看例子：

假设我们有这样一个 JSON 字符串需要解析 
```json
'{"user": {"address": {"state": "in"}}}'
```

我们需要取到用户的任何一个深度的地址，而每一层获取地址都可能失败。所以，每一次取值，我们都要做空值处理。

```js
// 解析 JSON 的函数，做了错误处理
function parse(s) {
  try {
    return JSON.parse(s);
  } catch (e) {
    return null;
  }
}
```

```js
import Maybe from "folktale/maybe";
import { compose, composeK, toUpper } from "ramda";

// 解析 JSON 可能返回 null，所以把结果放到 Maybe 里面
const maybeParseJson = json => Maybe.fromNullable(parse(json));

// 获取属性也可能返回空值，把结果放到 Maybe 里面
const maybeProp = prop => obj => Maybe.fromNullable(obj[prop]);

// 传给 toUpper 的值可能不是字符串，也要把结果放到 Maybe
const maybeUpper = compose(
  Maybe.of,
  toUpper
);

// composeK 代表 Kleisli composition
const getStateCode = composeK(
  maybeUpper,
  maybeProp("city"),
  maybeProp("state"),
  maybeProp("address"),
  maybeProp("user"),
  maybeParseJson
);

const s = '{"user": {"address": {"state": "in"}}}';

getStateCode(s).getOrElse("Error ocurred"); 

// city 属性不存在，程序会返回 'Error ocurred'
```