# unit-test-notes
现在的网页变得越来越复杂，如果想要保持代码的高质量，<b>单元测试</b>是一个非常好的选择。它可以帮助我们更好梳理业务逻辑，减少bug的出现。
<br>
本文记录了一些单元测试中常见的问题，希望能帮助到各位同学，避免重复踩坑。
<br>
<b>注意</b>，单元测试内容都是基于以下几个框架来编写的：
* [React](https://github.com/facebook/react)
* [Jest](https://github.com/facebook/jest)
* [Enzyme](https://github.com/airbnb/enzyme)

## 问题汇总

### 无法测试React组件中的异步回调内容
我们经常都会在React组件内部使用异步函数来完成业务逻辑，比如获取后台数据。在单元测试中，如果使用enzyme渲染组件，它并不会等待组件内部的异步操作执行完毕。因此在异步回调都还未被执行时，整个测试流程就已经结束了。

```js
export default class Item extends React.Component {
  constructor(props) {
    super(props);
  }

  componentDidMount() {
    new Promise(function(resolve) {
      resolve();
    }).then(function() {
      console.log('component did mount');
    });
  }

  render() {
    return (
      <div />
    );
  }
}
```

```js
import React from 'react';
import { shallow } from 'enzyme';
import Item from './Item';

test('promise callback in Item Component', async () => {
  var wrapper = shallow(
    <Item />
  );
});
```

执行上面代码后可以发现，`shallow`方法虽然会执行`componentDidMount`，但它并不是异步函数，所以"component did mount"不会被打印。那么要如何达到我们的目的呢？
<br>
此处介绍一个trick：

```js
function flushPromises() {
  return new Promise(function(resolve) {
    setImmediate(resolve);
  });
}
```

然后再修改一下单元测试：

```js
import React from 'react';
import { shallow } from 'enzyme';
import Item from './Item';

test('promise callback in Item Component', async () => {
  var wrapper = shallow(
    <Item />
  );
  await flushPromises();
  console.log('unit test finish');
});
```

再执行修改后的代码发现，终端会先打印"component did mount"，然后再打印"unit test finish"。这就是我们想要的结果！但这是什么原因呢？
<br>
其实这跟<b>JavaScript事件循环的task和microtask</b>有关。这里就不再详细叙述了，有兴趣的同学可以通过以下两个链接来了解一下：
* [Testing component with async componentDidMount](https://github.com/airbnb/enzyme/issues/1587)
* [深入理解 JavaScript 事件循环 — task and microtask](https://www.cnblogs.com/dong-xu/p/7000139.html)

另外再介绍一个不是那么"地道"的解决方法给大家：

```js
export default class Item extends React.Component {
  constructor(props) {
    super(props);
  }

  async componentDidMount() {
    await new Promise(function(resolve) {
      resolve();
    });
    console.log('component did mount');
  }

  render() {
    return (
      <div />
    );
  }
}
```

```js
import React from 'react';
import Item from './Item';

test('promise callback in Item Component', async () => {
  var ItemInstance = new Item({});
  await ItemInstance.componentDidMount();
  console.log('unit test finish');
});
```

该方法的思路是把组件当成一个实例来使用，这样就不用去考虑渲染问题。