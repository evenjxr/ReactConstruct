# React Fiber中的Props结构详解

本文档详细解释React Fiber中的`pendingProps`和`memoizedProps`结构，包括它们的定义、用途、内部结构以及在React渲染过程中的作用。

## Props在Fiber节点中的位置

在Fiber节点结构中，有两个与props相关的重要字段：

```typescript
export type Fiber = {
  // ...其他字段
  
  // 新的、待处理的props
  pendingProps: any,
  
  // 已经处理过的props
  memoizedProps: any,
  
  // ...其他字段
};
```

## pendingProps和memoizedProps的基本定义

### pendingProps

`pendingProps`表示**待处理的props**，是即将应用于组件的新props。这些props是从父组件传递下来的，但尚未被当前组件处理和"记忆"。

### memoizedProps

`memoizedProps`表示**已记忆的props**，是上一次渲染完成后使用的props。这些props已经被组件处理过，并且与当前屏幕上显示的内容相对应。

## Props的内部结构

Props本身是一个JavaScript对象，其结构取决于组件的定义和使用方式。一个典型的props对象可能如下所示：

```javascript
// 典型的props对象结构
{
  className: "container",
  style: { color: "red", fontSize: "16px" },
  onClick: function handleClick() { /* ... */ },
  children: [
    { type: "h1", props: { children: "标题" } },
    { type: "p", props: { children: "段落内容" } }
  ],
  // ...其他属性
}
```

### 特殊的children属性

`children`是一个特殊的prop，它代表组件的子元素。它可以是：

- 单个React元素
- React元素数组
- 原始类型（如字符串、数字）
- 函数（如渲染props模式）
- null或undefined（无子元素）

React内部会对`children`进行特殊处理，以支持各种渲染场景。

## pendingProps和memoizedProps在渲染过程中的作用

React的渲染过程可以分为两个主要阶段：协调阶段（Reconciliation）和提交阶段（Commit）。`pendingProps`和`memoizedProps`在这个过程中扮演着重要角色。

### 协调阶段

1. **开始渲染时**：
   - 从父组件接收新的props，存储在`pendingProps`中
   - 将`pendingProps`与`memoizedProps`进行比较，判断是否需要更新

2. **处理组件时**：
   - 对于类组件：将`pendingProps`传递给`render`方法
   - 对于函数组件：将`pendingProps`作为参数传递给函数

3. **完成组件处理后**：
   - 如果组件成功渲染，将`pendingProps`复制到`memoizedProps`
   - 这表示当前的props已经被"记忆"，成为下一次渲染的比较基准

### 提交阶段

在提交阶段，React使用`memoizedProps`来更新DOM或调用生命周期方法。此时，`memoizedProps`已经是最新的props值。

## Props比较和优化

React使用`pendingProps`和`memoizedProps`的比较来决定是否需要更新组件：

```javascript
// React内部的浅比较逻辑（简化版）
function shallowEqual(oldProps, newProps) {
  // 引用相同，直接返回true
  if (oldProps === newProps) {
    return true;
  }
  
  // 获取所有属性名
  const keys = Object.keys(newProps);
  
  // 属性数量不同，返回false
  if (keys.length !== Object.keys(oldProps).length) {
    return false;
  }
  
  // 逐个比较属性值
  for (let i = 0; i < keys.length; i++) {
    const key = keys[i];
    if (newProps[key] !== oldProps[key]) {
      return false;
    }
  }
  
  return true;
}

// 在处理Fiber节点时的判断（简化版）
if (shallowEqual(workInProgress.memoizedProps, workInProgress.pendingProps)) {
  // props没有变化，可以跳过更新
} else {
  // props发生变化，需要更新组件
}
```

### React.memo和PureComponent

`React.memo`和`PureComponent`就是利用props比较来优化性能的：

- `React.memo`：对函数组件的props进行浅比较
- `PureComponent`：对类组件的props和state进行浅比较

如果props没有变化，这些组件会跳过重新渲染。

## Props在不同类型组件中的处理

### 类组件

在类组件中，props通过构造函数传入，并可以通过`this.props`访问：

```javascript
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    // 构造函数中可以访问props
    this.state = {
      derivedValue: props.initialValue
    };
  }
  
  render() {
    // 通过this.props访问props
    return <div>{this.props.message}</div>;
  }
}
```

在Fiber中，类组件实例的`props`属性实际上指向的是对应Fiber节点的`memoizedProps`。

### 函数组件

在函数组件中，props作为函数的参数传入：

```javascript
function MyComponent(props) {
  // 直接使用props参数
  return <div>{props.message}</div>;
}
```

函数组件没有实例，所以没有持久化的`this.props`。每次渲染时，函数都会接收新的props参数。

## Props的不可变性

在React中，props是不可变的（immutable）。组件不应该修改接收到的props：

```javascript
// 错误的做法
function WrongComponent(props) {
  // 不应该修改props
  props.value = 42;
  return <div>{props.value}</div>;
}

// 正确的做法
function CorrectComponent(props) {
  // 创建新的状态，而不是修改props
  const [value, setValue] = useState(props.initialValue);
  return <div>{value}</div>;
}
```

这种不可变性有助于保持组件行为的可预测性，并支持React的优化策略。

## Props的传递和透传

### Props传递

Props从父组件传递到子组件：

```javascript
function Parent() {
  return <Child name="John" age={30} />;
}

function Child(props) {
  // 接收name和age props
  return <div>{props.name}, {props.age}</div>;
}
```

### Props透传

有时需要将所有props透传给子组件：

```javascript
function Wrapper(props) {
  // 使用展开运算符透传所有props
  return <div className="wrapper"><Child {...props} /></div>;
}
```

在Fiber中，这种透传会直接影响`pendingProps`的值，但不会创建新的props对象引用，从而可能触发优化。

## 默认Props

组件可以定义默认props，当父组件没有提供特定prop时使用：

```javascript
function Button({ type = "button", children }) {
  return <button type={type}>{children}</button>;
}

// 类组件的默认props
Button.defaultProps = {
  type: "button"
};
```

在Fiber处理过程中，默认props会在创建`pendingProps`时被合并。

## 总结

`pendingProps`和`memoizedProps`是React Fiber中处理组件属性的关键字段：

- `pendingProps`：新的、待处理的props，来自父组件的最新传值
- `memoizedProps`：已处理的props，对应当前屏幕显示的内容

React通过比较这两个字段来决定是否需要更新组件，这是React性能优化的重要基础。理解这两个字段的结构和作用，有助于更好地理解React的渲染机制和优化策略。