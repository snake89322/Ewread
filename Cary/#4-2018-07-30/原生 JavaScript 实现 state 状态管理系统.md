# 原生 JavaScript 实现 state 状态管理系统
#article

> [Build a state management system with vanilla JavaScript | CSS-Tricks](https://css-tricks.com/build-a-state-management-system-with-vanilla-javascript/) <br>

在软件工程中，状态管理已经不是什么新鲜概念，但是在 JavaScript 语言中比较流行的框架都在使用相关概念。传统意义上，我们会保持 DOM 本身的状态甚至声明该状态为全局变量。不过现在，我们有很多状态管理的宠儿供我们选择。比如 Redux，MobX 以及 Vuex，使得跨组件的状态管理更为方便。这对于一些响应式的框架非常适用，比如 React 或者 Vue。

然而，这些状态管理库是如何实现的？我们能否自己创造一个？先不讨论这些，最起码，我们能够真实地了解状态管理的通用机制和一些流行的 API。

在开始之前，需要具备 JavaScript 的基础知识。你应该知道数据类型的概念，了解 ES6 相关语法及功能。如果不太了解，[去这里学习一下](https://css-tricks.com/learning-gutenberg-4-modern-javascript-syntax/)。这篇文章并不是要替代 Redux 或者 MobX。在这里我们进行一次技术探索，各持己见就好。

# 前言
在开始之前，我们先看看需要达到的效果。

* [项目 Demo](https://vanilla-js-state-management.hankchizljaw.io)
* [项目 Repo](http://github.com/hankchizljaw/vanilla-js-state-management)

# 架构设计
使用你最爱的 IDE，创建一个文件夹：

```
~/Documents/Projects/vanilla-js-state-management-boilerplate/
```

项目结构类似如下：

```
/src
├── .eslintrc
├── .gitignore
├── LICENSE
└── README.md
```

# Pub/Sub
下一步，进入 `src` 目录，创建 `js` 目录，下面创建 `lib`目录，并创建 `pubsub.js`。

结构如下：

```
/js
├── lib
└── pubsub.js
```

打开 `pubsub.js` 因为我们将要实现一个 [订阅/发布 模块](https://msdn.microsoft.com/en-us/magazine/hh201955.aspx)。全称 “Publish/Subscribe”。在我们应用中，我们会创建一些功能模块用于订阅我们命名的事件。另一些模块会发布相应的事件，通常应用在一个相关的负载序列上。

Pub/Sub 有时候很难理解，如何去模拟呢？想象一下你工作在一家餐厅，你的用户有一个发射装置和一个菜单。假如你在厨房工作，你知道什么时候服务员会清除发射装置（下单），然后让大厨知道哪一个桌子的发射装置被清除了（下单）。这就是一条对应桌号的点菜线程。在厨房里面，一些厨子需要开始作业。他们是被这条点菜线程订阅了，直到菜品完成，所以厨子知道自己要做什么菜。因此，你手底下的厨师都在为相同的点菜线程（称为 event），去做对应的菜品（称为 callback）。

![](https://res.cloudinary.com/css-tricks/image/upload/c_scale,w_1000,f_auto,q_auto/v1531870147/state-management-restaurant_boashk.jpg)

上图是一个直观的解释。

PubSub 模块会预加载所有的订阅并执行他们各自的回调函数。只需要几行代码就能够创建一个非常优雅地响应流。

在 `pubsub.js` 中添加如下代码：

```js
export default class PubSub {
  constructor() {
    this.events = {};
  }
}
```

`this.events` 用来保存我们定义的事件。

然后在 constructor 下面增加如下代码：

```js
subscribe(event, callback) {

  let self = this;

  if(!self.events.hasOwnProperty(event)) {
    self.events[event] = [];
  }

  return self.events[event].push(callback);
}
```

这里是一个订阅方法。参数 `event` 是一个字符串类型， 用于指定唯一的 event 名字用于回调。如果没有匹配的 event 在 `events` 集合中，那么我们创建一个空数组用于之后的检查。然后我们将回调方法 push 到这个 event 集合中。如果存在 event 集合，将回调函数直接 push 进去。最后返回集合长度。

现在我们需要获取对应的订阅方法，猜猜接下来是什么?你们知道的：是 `publish` 方法。添加如下代码：

```js
publish(event, data = {}) {

  let self = this;

  if(!self.events.hasOwnProperty(event)) {
    return [];
  }

  return self.events[event].map(callback => callback(data));
} 
```

这个方法首先检查传递的 event 是否存在。如果不存在，返回空数组。如果存在，那么遍历集合中的方法，并将 data 传递进去执行。如果没有回调方法，那也 ok，因为我们创建的空数组也会适用于 `subscribe` 方法。

这就是 PubSub。接下来看看是什么！

# 核心的存储对象 Store
现在我们已经有了订阅/发布模型，我们想要创建这个应用的依赖：Store。我们一点一点来看。

先看一下这个存储对象是用来干什么的。

Store 是我们的核心对象。每次引入 `@import store from '../lib/store.js'`, 你将会在这个对象中存储你编写的状态位。这个 `state` 的集合，包含我们应用的所有状态，它有一个 `commit` 方法我们称为 **mutations**，最后有一个 `dispatch` 方法我们称为 **actions**。在这个核心实现的细节中，应该有一个基于代理（Proxy-based）的系统，用来监听和广播在 `PubSub` 模型中的状态变化。

我们创建一个新的文件夹 `store` 在 `js` 下面。然后再创建一个 `store.js` 的文件。你的 `js` 目录看起来应该是如下的样子：

```
/js
└── lib
    └── pubsub.js
└──store
    └── store.js
```

打开 `store.js` 并且引入 订阅/发布 模块。如下：

```js
import PubSub from '../lib/pubsub.js';
```

这在 ES6 语法中很常见，非常具有辨识性。

下一步，开始创建对象：

```js
export default class Store {
  constructor(params) {
    let self = this;
  }
}
```

这里有一个自我声明。我们需要创建默认的 `state`，`actions`，以及 `mutations`。我们也要加入 `status` 元素用来判定 Store 对象在任意时刻的行为：

```js
self.actions = {};
self.mutations = {};
self.state = {};
self.status = 'resting';
```

在这之后，我们需要实例化 `PubSub`，绑定我们的 `Store` 作为一个 `events` 元素：

```js
self.events = new PubSub();
```

接下来我们需要寻找传递的 `params` 对象是否包含 `actions` 或者 `mutations`。当 `Store` 初始化时，我们将数据传递进去。包含一个 `actions` 和 `mutations
`  的集合，这个集合用来控制存储的数据：

```js
if(params.hasOwnProperty('actions')) {
  self.actions = params.actions;
}

if(params.hasOwnProperty('mutations')) {
  self.mutations = params.mutations;
}
```

以上是我们默认设置和可能的参数设置。接下来，让我们看看 `Store` 对象如何追踪变化。我们会用 [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) 实现。Proxy 在我们的状态对象中使用了一半的功能。如果我们使用 `get`，每次访问数据都会进行监听。同样的选择 `set` ，我们的监测将作用于数据改变时。代码如下：

```js
self.state = new Proxy((params.state || {}), {
  set: function(state, key, value) {

    state[key] = value;

    console.log(`stateChange: ${key}: ${value}`);

    self.events.publish('stateChange', self.state);

    if(self.status !== 'mutation') {
      console.warn(`You should use a mutation to set ${key}`);
    }

    self.status = 'resting';

    return true;
  }
});
```

在这个 `set` 函数中发生了什么？这意味着如果有数据变化如 `state.name = 'Foo'`，这段代码将会运行。及时在我们的上下文环境中，改变数据并打印。我们可以发布一个 `stateChange` 事件到 `PubSub` 模块。任何订阅的事件的回调函数会执行，我们检查 `Store` 的 status，当前的状态应该是  `mutation`，这意味着状态已经被更新了。我们可以添加一个警告去提示开发者非 `mutation` 状态下更新数据的风险。

# Dispatch 和 commit
我们已经将核心的元素添加到 `Store` 中了，现在我们添加两个方法。`dispatch` 用于执行 `actions`，`commit` 用于执行 `mutations`。代码如下：

```js
dispatch (actionKey, payload) {

  let self = this;

  if(typeof self.actions[actionKey] !== 'function') {
    console.error(`Action "${actionKey} doesn't exist.`);
    return false;
  }

  console.groupCollapsed(`ACTION: ${actionKey}`);

  self.status = 'action';

  self.actions[actionKey](self, payload);

  console.groupEnd();

  return true;
}
```

处理过程如下：寻找 action，如果存在，设置 status，并且运行 action。 `commit` 方法很相似。

```js
commit(mutationKey, payload) {
  let self = this;

  if(typeof self.mutations[mutationKey] !== 'function') {
    console.log(`Mutation "${mutationKey}" doesn't exist`);
    return false;
  }

  self.status = 'mutation';

  let newState = self.mutations[mutationKey](self.state, payload);

  self.state = Object.assign(self.state, newState);

  return true;
}
```

# 创建一个基础组件
我们创建一个列表去实践状态管理系统：

```
~/Documents/Projects/vanilla-js-state-management-boilerplate/src/js/lib/component.js
```

```js
import Store from '../store/store.js';

export default class Component {
  constructor(props = {}) {
    let self = this;

    this.render = this.render || function() {};

    if(props.store instanceof Store) {
      props.store.events.subscribe('stateChange', () => self.render());
    }

    if(props.hasOwnProperty('element')) {
      this.element = props.element;
    }
  }
}
```

我们看看这一串代码。首先，引入 `Store` 类。我们并不想要一个实例，但是更多的检查是放在 `constructor` 中。在 `constructor` 中，我们可以得到一个 render 方法，如果 `Component` 类是其他类的父类，可能会用到继承类的 `render` 方法。如果没有对应的方法，那么会创建一个空方法。

之后，我们检查 `Store` 类的匹配。需要确认 `store` 方法是 `Store` 类的实例，如果不是，则不执行。我们订阅了一个全局变量 `stateChange` 事件让我们的程序得以响应。每次 state 变化都会触发 render 方法。

基于这个基础组件，然后创建其他组件。

# 创建我们的组件
创建一个列表：

```
~/Documents/Projects/vanilla-js-state-management-boilerplate/src/js/component/list.js
```

```js
import Component from '../lib/component.js';
import store from '../store/index.js';

export default class List extends Component {

  constructor() {
    super({
      store,
      element: document.querySelector('.js-items')
    });
  }

  render() {
    let self = this;

    if(store.state.items.length === 0) {
      self.element.innerHTML = `<p class="no-items">You've done nothing yet &#x1f622;</p>`;
      return;
    }

    self.element.innerHTML = `
      <ul class="app__items">
        ${store.state.items.map(item => {
          return `
            <li>${item}<button aria-label="Delete this item">×</button></li>
          `
        }).join('')}
      </ul>
    `;

    self.element.querySelectorAll('button').forEach((button, index) => {
      button.addEventListener('click', () => {
        store.dispatch('clearItem', { index });
      });
    });
  }
};
```

创建一个计数组件：

```js
import Component from '../lib/component.js';
import store from '../store/index.js';

export default class Count extends Component {
  constructor() {
    super({
      store,
      element: document.querySelector('.js-count')
    });
  }

  render() {
    let suffix = store.state.items.length !== 1 ? 's' : '';
    let emoji = store.state.items.length > 0 ? '&#x1f64c;' : '&#x1f622;';

    this.element.innerHTML = `
      <small>You've done</small>
      ${store.state.items.length}
      <small>thing${suffix} today ${emoji}</small>
    `;
  }
}
```

创建一个 status 组件：

```js
import Component from '../lib/component.js';
import store from '../store/index.js';

export default class Status extends Component {
  constructor() {
    super({
      store,
      element: document.querySelector('.js-status')
    });
  }

  render() {
    let self = this;
    let suffix = store.state.items.length !== 1 ? 's' : '';

    self.element.innerHTML = `${store.state.items.length} item${suffix}`;
  }
}
```

文件目录结构如下：

```
/src
├── js
│   ├── components
│   │   ├── count.js
│   │   ├── list.js
│   │   └── status.js
│   ├──lib
│   │  ├──component.js
│   │  └──pubsub.js
└───── store
│      └──store.js
└───── main.js
```

# 完善状态管理
我们已经得到前端组件和主要的 `Store`。现在需要一个初始状态，一些 `actions` 和 `mutations`。在 `store` 目录下，创建一个新的 `state.js` 文件：

```
~/Documents/Projects/vanilla-js-state-management-boilerplate/src/js/store/state.js
```

```js
export default {
  items: [
    'I made this',
    'Another thing'
  ]1
};
```

继续创建 `actions.js`：

```js
export default {
  addItem(context, payload) {
    context.commit('addItem', payload);
  },
  clearItem(context, payload) {
    context.commit('clearItem', payload);
  }
};
```

继续创建 `mutation.js`

```js
export default {
  addItem(state, payload) {
    state.items.push(payload);

    return state;
  },
  clearItem(state, payload) {
    state.items.splice(payload.index, 1);

    return state;
  }
};
```

最后创建 `index.js`：

```js
import actions from './actions.js';
import mutations from './mutations.js';
import state from './state.js';
import Store from './store.js';

export default new Store({
  actions,
  mutations,
  state
});
```

# 最后的集成
最后我们将所有代码集成到 `main.js`中，还有 `index.html` 中：

```
~/Documents/Projects/vanilla-js-state-management-boilerplate/src/js/main.js
```

```js
import store from './store/index.js'; 

import Count from './components/count.js';
import List from './components/list.js';
import Status from './components/status.js';

const formElement = document.querySelector('.js-form');
const inputElement = document.querySelector('#new-item-field');
```

到此一切准备就绪，下面添加交互：

```js
formElement.addEventListener('submit', evt => {
  evt.preventDefault();

  let value = inputElement.value.trim();

  if(value.length) {
    store.dispatch('addItem', value);
    inputElement.value = '';
    inputElement.focus();
  }
});
```

添加渲染：

```js
const countInstance = new Count();
const listInstance = new List();
const statusInstance = new Status();

countInstance.render();
listInstance.render();
statusInstance.render();
```

至此完成了一个状态管理的系统。