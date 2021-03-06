# 精读《React Hooks》

## 引言

React Hooks 是 React 16.7.0-alpha 版本推出的新特性，想尝试的同学安装此版本即可。

React Hooks 要解决的问题是状态共享，是继 render-props 和 higher-order components 之后的第三种状态共享方案，不会产生 JSX 嵌套地域问题。

状态共享可能描述的不恰当，称为状态逻辑复用会更恰当，因为只共享数据处理逻辑，不会共享数据本身。

为了更快理解 React Hooks 是什么，先看下面的一段代码：

```
function App() {
  return (
    <Toggle initial={false}>
      {
        ({on, toggle}) => {
          <Button type="primary" onClick={toggle}>Open Modal</Button>
          <Modal visible={on} onOk={toggle} onCancel={toggle}/>
        }
      }
    </Toggle>
  )
}
```

恰巧，React Hooks 解决的也是这个问题：

```
function App() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <Button type="primary" onClick={() => {setOpen(true)}}>
        Open Modal
      </Button>
      <Modal
        visible={open}
        onOk={() => {setOpen(false)}}
        onCancel={() => {setOpen(false)}}
      />
    </>
  )
}
```

可以看到，React Hooks 就像一个内置的打平 renderProps 库，我们可以随时创建一个值，与修改这个值的方法。看上去像 function 形式的 setState，其实这等价于依赖注入，与使用 setState 相比，这个组件是没有状态的。

## 概述

React Hooks 带来的好处不仅是“更 FP,更新粒度更细，代码更清晰”，还有如下三个特性：

- 多个状态不会产生嵌套，写法还是平铺的（renderProps 可以通过 compose 解决，不但使用略为繁琐，而且因为强制封装一个新对象而增加了实体数量）
- Hooks 可以引用其他 Hooks
- 更容易将组建的 UI 与状态分离

第二点展开说一下：Hooks 可以引用其他 Hooks

```
import { useState, useEffect } from 'react';

// 底层 Hooks，返回布尔值：是否在线
function useFriendStatusBoolean(friendID) {
  const [isOnline, setIsOnline] = useState(null);

  function handleStatusChange(status) {
    setIsOnline(status.isOnline);
  }

  useEffect(() => {
    ChatAPI.subscribeToFriendsStatus(friendID, handleStatusChange);

    return () => {
      ChatAPI.unsubscribeFromFriendStatus(friendId, handleStatusChange);
    }
  })

  return isOnline;
}

// 上层 Hooks，根据在线状态返回字符串：Loading... or Online or Offline
function useFriendStatusString(props) {
  const isOnline = useFriendStatusBoolean(props.friend.id);

  if (isOnline === null) {
    return "Loading...";
  }
  return isOnline ? "Online" : "Offline"
}

// 使用了底层 Hooks 的 UI
function FriendListItem(props) {
  const isOnline = useFriendStatusBoolean(props.friend.id)

  return (
    <li style={{color: isOnline ? "green" : "black"}}>{props.friend.name}</li>
  )
}

// 使用了上层 Hooks 的 UI
function FriendListStatus(props) {
  const statu = useFriendStatusString(props.friend.id);

  return <li>{statu}</li>
}
```

这个例子中，有两个 Hooks：useFriendStatusBoolean 与 useFriendStatusString，useFriendStatusString 是利用 useFriendStatusBoolean 生成的新 Hook，这两个 Hook 可以给不同的 UI：FriendListItem、FriendListStatus 使用，而因为两个 Hooks 数据是联动的，因此两个 UI 的状态也是联动的。

顺带一提，这个例子也可以用来理解 “有状态的组件没有渲染，有选人的组件没有状态”

- useFriendStatusBoolean 与 useFriendStatusString 是有状态的组件（使用 useState），没有渲染（返回非 UI 的值），这样就可以作为 Custom Hooks 被任何 UI 组件调用
- FriendListItem 与 FriendListStatus 是有渲染的组件（返回了 JSX），没有状态（没有使用 useState），这就是一个纯函数 UI 组件

### 利用 useState 创建 Redux

Redux 的精髓就是 Reducer，而利用 React Hooks 可以轻松创建一个 Redux 机制：

```
// 这就是 Redux
function useReducer(reducer, initialState) {
  const [state, setState] = useState(initialState);

  function dispatch(action) {
    const nextState = reducer(state, action);
    setState(nextState);
  }

  return [state, dispatch];
}
```

这个自定义 Hook 的 value 部分当做 redux 的 state，setValue 部分当作 redux 的 dispatch，合起来就是一个 redux。而 react-redux 的 connect 部分做的事情与 Hook 调用一样：

```
// 一个 Action
function useTodos() {
  const [todos, dispatch] = useReducer(todosReducer, []);

  function handleAddClick(text) {
    dispatch({type: "add", text});
  }

  return [todos, { handleAddClick }];
}


// 绑定 Todos 的 UI
function TodosUI() {
  const [todos, actions] = useTodos()

  return (
    <>
      {
        todos.map((todo, index) => {
          <div>{todo.text}</div>
        })
      }
      <button onClick={actions.handleAddClick}>Add Todo</button>
    </>
  )
}
```

useReducer 已经作为一个内置 Hooks 了

不过这里需要注意的是，每次 useReducer 或者自己的 Custom Hooks  都不会持久化数据，所以比如我们创建两个 App，App1 与 App2

```
function App1() {
  const [todos, actions] = useTodos();

  return <span>todo count: {todos.length}</span>
}
function App2() {
  const [todos, actions] = useTodos();

  return <span>todo count: {todos.length}</span>
}

function All() {
  return (
    <>
      <App1/>
      <App2/>
    </>
  )
}
```

这两个实例同时渲染时，并不是共享一个 todos 列表，而是分别存在两个独立 todos 列表。也就是 React Hooks 只提供状态处理方法，不会持久化状态。

如果要真正实现一个 Redux 功能，也就是全局维持一个状态，任何组件 useReducer 都会访问到同一份数据，可以和 useContext 一起使用。

大体思路是利用 useContext 共享一份数据，作为 Custom Hooks 的数据源。具体实现可以参考 redux-react-hook

### 利用useEffect 代替一些生命周期

在 useState 位置附近，可以使用 useEffect 处理副作用

```
useEffect(() => {
  const subscription = props.source.subscribe()

  return () => {
    subscription.unsubscribe()
  }
})
```

useEffect 的代码既会在初始化的时候执行，也会在后续每次 render 的时候执行，而返回值在析构时执行。这个更多带来的是便利，对比一下 React 版 G2 调用流程：

```
class Component extends React.PureComponent<Props, State> {
  private chart: G2.Chart = null;
  private rootDomRef: React.ReactInstance = null;

  componentDidMount() {
    this.rootDom = ReactDOM.findDOMNode(this.rootDomRef) as HTMLDivElement;

    this.chart = new G2.Chart({
      container: document.getElementById("chart"),
      forceFit: true,
      height: 300
    });
    this.freshChart(this.props);
  }

  componentWillReceiveProps(nextProps: Props) {
    this.freshChart(nextProps);
  }

  componentWillUnmount() {
    this.chart.destroy();
  }

  freshChart(props: Props) {
    this.chart.render();
  }

  render() {
    return <div ref={(ref) => {this.rootDomRef = ref}}/>;
  }
}
```

用 React Hooks 可以这么做：

```
function App() {
  const ref = React.useRef(null);
  let chart: G2.Chart = null;

  React.useEffect(() => {
    if (!chart) {
      chart = new G2.Chart({
        container: ReactDOM.findDOMNode(ref.current) as HTMLDivElement,
        width: 500,
        height: 500
      })
    }

    chart.render()

    return () => {
      chart.destroy();
    }
  })

  return <div ref={ref}/>
}
```

可以看到将细碎的代码片段结合成了一个完整的代码块，更方便维护。

现在介绍了 useState useContext useEffect useRef 等常用 hooks，更多可以查阅官方介绍。

## 精读

Hooks 带来的约定

Hook 函数必须是以 use 命名开头，因为这样才方便 eslint 做检查，防止用 condition 判断包裹 useHook 语句

为什么不能用 condition 包裹 useHook 语句，详情可以看官方文档。

React Hooks 并不是通过 Proxy 或者 getters 实现的，而是用过数组实现的，每次 useState 都会改变下标，如果 useState 被包裹在 condition 中，那每次执行的下标就可能对不上，导致 useState 导出的 setter 更新错数据。

### 状态与 UI 的界限会越来越清晰

因为 React Hooks  的特性，如果一个 Hook 不产生 UI，那么它可以永远被其他 Hook 封装，虽然允许有副作用，但是被包裹在 useEffect 里，总体来说还是挺函数式的。而 Hooks 要集中在 UI 函数顶部写，也很容易养成书写无状态 UI 组件的好习惯，践行“状态与 UI 分离”这个理念会更容易。

不过这个理念稍微有点蹩脚的地方，那就是“状态”到底是什么

```
function App() {
  const [count, setCount] = useCount();

  return <span>{count}</span>;
}
```

我们知道 useCount 算是无状态的，因为 React Hooks 本质就是 renderProps 或者 HOC 的另一种写法，换成 renderProps 就好理解了：

```
<Count>{(count, setCount) => {return <App count={count} setCount={setCount}/>}}</Count>

function App(props) {
  return <span>{props.count}</span>
}
```

可以看到 App 组件是无状态的，输出完全由输入（Props）决定

那么有状态 无 UI 的组件就是 useCount 了

```
function useCount() {
  const [count, setCount] = useState(0)

  return [count, setCount];
}
```

有状态的地方应该指 useState(0) 这句，不过这句和无状态 UI 组件 App 的 useCount() 很像，既然 React 把 useCount 成为自定义 HOOk，那么 UseState 就是官方 Hook，具有一样的定义，因此可以认为 useCount 是无状态的，useState 也是一层 renderProps，最终的状态其实是 useState 这个 React 内置的组件。

我们看 renderProps 嵌套的表达

```
<UseState>
  {
    (count, setCount) => {
      return (
        <UseCount>
          {" "}
          {
            (count, setCount) => {
              return <App count={count} setCount={setCount}/>
            }
          }
        </UseCount>
      )
    }
  }
</UseState>
```

能确定的是，App 一定有 UI，而上面两层父级组件一定没有 UI。为了最佳实践，我们尽量避免 App 自己维护状态，而其父级的 RenderProps 组件可以维护状态（也可以不维护状态，做个二传手）。因此可以考虑在“有状态的组件没有渲染，有渲染的组件没有状态”这句话后面加一句：没渲染的组件也可以没状态。

## 总结

把 React Hooks 仿作更便捷的 RenderProps 去用吧，虽然写法看上去是内部维护了一个状态，但其实等价于注入、Connect、HOC、或者 renderProps，那么如此一来，使用 renderProps 的门槛会大大降低，因为 Hooks 用起来实在是太方便了，我们可以抽象大量 Custom Hooks，让代码更加 FP，同时也不会增加嵌套层级。