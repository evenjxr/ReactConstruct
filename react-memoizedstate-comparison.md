# React memoizedState 详解：函数组件与类组件对比

本文档详细介绍React中`memoizedState`的结构和工作原理，并比较函数组件和类组件在`memoizedState`使用上的差异。

## 概述

在React Fiber架构中，`memoizedState`是Fiber节点上的一个关键字段，用于存储组件的状态。虽然函数组件和类组件都使用这个字段，但它们的内部结构和工作原理有着显著差异。

```typescript
// Fiber节点上的memoizedState字段
export type Fiber = {
  // ...其他字段

  // 上次渲染完成后的状态
  memoizedState: any,

  // ...其他字段
};
```

## 类组件的memoizedState

### 基本结构

在类组件中，`memoizedState`直接存储组件实例的`state`对象。它是一个简单的JavaScript对象，与组件的`this.state`完全相同。

```typescript
// 类组件的memoizedState结构示例
memoizedState = {
  count: 0,
  name: 'React',
  isActive: true,
  user: {
    id: 1,
    name: 'User'
  },
  // ...其他状态属性
};
```

### 工作原理

1. **初始化**：
   - 在组件挂载时，React从类组件的构造函数或类属性中获取初始状态
   - 这个初始状态被赋值给Fiber节点的`memoizedState`字段

2. **更新过程**：
   - 当调用`this.setState()`时，React创建更新对象并加入更新队列
   - 在渲染阶段，React处理更新队列，计算新的状态
   - 计算完成后，新状态被赋值给`memoizedState`
   - 在提交阶段，React确保组件实例的`this.state`与`memoizedState`保持同步

3. **状态合并**：
   - 类组件的状态更新是浅合并的
   - 新的状态对象会与现有的`memoizedState`进行合并，而不是完全替换

### 示例

```typescript
class Counter extends React.Component {
  constructor(props) {
    super(props);
    // 初始状态被赋值给memoizedState
    this.state = { count: 0, name: 'Counter' };
  }

  increment = () => {
    // 这个更新会被处理并最终更新memoizedState
    this.setState({ count: this.state.count + 1 });
  };

  render() {
    // 渲染时使用的state实际上是memoizedState的引用
    return <div>{this.state.count}</div>;
  }
}
```

在上面的例子中，当组件挂载时，`{ count: 0, name: 'Counter' }`被赋值给Fiber节点的`memoizedState`。当调用`increment`方法时，React会计算新的状态`{ count: 1, name: 'Counter' }`并更新`memoizedState`。

## 函数组件的memoizedState

### 基本结构

函数组件的`memoizedState`结构完全不同。它不是一个简单的状态对象，而是一个**链表**，存储了组件中所有Hook的状态。

```typescript
// 函数组件的memoizedState是一个Hook链表
// 每个Hook都有自己的memoizedState字段
type Hook = {
  memoizedState: any, // Hook自己的状态
  baseState: any,
  baseQueue: Update<any, any> | null,
  queue: UpdateQueue<any, any> | null,
  next: Hook | null, // 指向下一个Hook
};
```

每个Hook的`memoizedState`字段存储的内容取决于Hook的类型：

1. **useState/useReducer**：存储当前状态值
2. **useEffect/useLayoutEffect**：存储effect对象，包含依赖数组和清理函数
3. **useMemo**：存储`[memoizedValue, deps]`数组
4. **useRef**：存储`{ current: value }`对象
5. **useCallback**：存储`[callback, deps]`数组
6. **useContext**：存储context的当前值

### 工作原理

1. **Hook链表**：
   - 函数组件中的每个Hook按照调用顺序形成一个链表
   - Fiber节点的`memoizedState`指向链表的第一个Hook
   - 每个Hook通过`next`字段链接到下一个Hook

2. **初始化**：
   - 在组件首次渲染时，React为每个Hook创建一个Hook对象
   - 每个Hook对象的`memoizedState`字段存储Hook的初始状态
   - 所有Hook对象形成一个链表，链表头存储在Fiber节点的`memoizedState`中

3. **更新过程**：
   - 在组件重新渲染时，React按照相同的顺序遍历Hook链表
   - 每个Hook的更新逻辑独立处理，更新自己的`memoizedState`
   - Hook的顺序必须保持稳定，这就是为什么Hook不能在条件语句中使用


### 不同Hook的memoizedState结构

#### useState/useReducer

```typescript
// useState的memoizedState直接存储状态值
const [count, setCount] = useState(0);
// Hook.memoizedState = 0

// 对于复杂对象也是直接存储
const [user, setUser] = useState({ name: 'React', age: 10 });
// Hook.memoizedState = { name: 'React', age: 10 }
```

#### useEffect/useLayoutEffect

```typescript
// useEffect的memoizedState存储effect对象
useEffect(() => {
  console.log('Effect ran');
  return () => console.log('Cleanup');
}, [dep1, dep2]);

// Hook.memoizedState = {
//   tag: HookEffectTag,
//   create: () => { console.log('Effect ran'); return () => console.log('Cleanup'); },
//   destroy: undefined | cleanup function,
//   deps: [dep1, dep2],
//   next: nextEffect
// }
```

#### useMemo

```typescript
// useMemo的memoizedState存储[计算结果, 依赖数组]
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
// Hook.memoizedState = [computedValue, [a, b]]
```

#### useRef

```typescript
// useRef的memoizedState存储{ current: initialValue }对象
const refObject = useRef(initialValue);
// Hook.memoizedState = { current: initialValue }
```

#### useCallback

```typescript
// useCallback的memoizedState存储[回调函数, 依赖数组]
const memoizedCallback = useCallback(() => doSomething(a, b), [a, b]);
// Hook.memoizedState = [callback, [a, b]]
```

### 示例

```typescript
function Counter() {
  // 第一个Hook: useState
  const [count, setCount] = useState(0);

  // 第二个Hook: useState
  const [name, setName] = useState('Counter');

  // 第三个Hook: useEffect
  useEffect(() => {
    document.title = `${name}: ${count}`;
  }, [count, name]);

  // 第四个Hook: useRef
  const prevCount = useRef(count);

  // 组件渲染逻辑
  return <div>{count}</div>;
}
```

在上面的例子中，函数组件的`memoizedState`是一个包含4个Hook的链表：

```typescript
// Fiber.memoizedState (链表头) -> Hook1 -> Hook2 -> Hook3 -> Hook4

// Hook1 (useState)
{
  memoizedState: 0, // count的值
  next: Hook2
}

// Hook2 (useState)
{
  memoizedState: 'Counter', // name的值
  next: Hook3
}

// Hook3 (useEffect)
{
  memoizedState: {
    tag: HookEffectTag,
    create: () => { document.title = `${name}: ${count}`; },
    deps: [count, name],
    next: null
  },
  next: Hook4
}

// Hook4 (useRef)
{
  memoizedState: { current: 0 }, // prevCount的值
  next: null
}
```

## 关键差异对比

| 特性 | 类组件 | 函数组件 |
|------|--------|----------|
| **memoizedState结构** | 单一对象，直接存储组件状态 | Hook链表，每个Hook有自己的memoizedState |
| **状态组织** | 所有状态集中在一个对象中 | 状态分散在各个独立的Hook中 |
| **状态更新** | 通过setState进行浅合并 | 每个useState独立更新，完全替换而非合并 |
| **状态访问** | 通过this.state访问 | 通过各个Hook的返回值访问 |
| **状态依赖** | 状态之间可能存在隐式依赖 | 通过依赖数组显式声明依赖关系 |
| **稳定性要求** | 状态结构可以动态变化 | Hook的调用顺序必须稳定 |

## 内部实现差异

### 类组件的状态处理

```typescript
// 类组件状态更新的简化实现
function processUpdateQueue(workInProgress, instance) {
  const queue = workInProgress.updateQueue;
  let newState = workInProgress.memoizedState;

  // 处理更新队列中的所有更新
  const firstUpdate = queue.firstUpdate;
  if (firstUpdate !== null) {
    let update = firstUpdate;
    do {
      // 计算新状态
      const payload = update.payload;
      let partialState;

      if (typeof payload === 'function') {
        // 函数式更新
        partialState = payload.call(instance, newState);
      } else {
        // 对象式更新
        partialState = payload;
      }

      // 浅合并状态
      newState = Object.assign({}, newState, partialState);

      update = update.next;
    } while (update !== null);
  }

  // 更新memoizedState
  workInProgress.memoizedState = newState;

  // 在提交阶段，确保instance.state与memoizedState同步
  instance.state = newState;
}
```

### 函数组件的Hook处理

```typescript
// 函数组件Hook处理的简化实现
function renderWithHooks(current, workInProgress, Component, props) {
  // 当前正在处理的Fiber
  currentlyRenderingFiber = workInProgress;

  // 重置Hook链表
  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;

  // 设置Hook上下文
  ReactCurrentDispatcher.current =
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount  // 首次渲染
      : HooksDispatcherOnUpdate; // 更新渲染

  // 调用函数组件，这将触发Hook的调用
  let children = Component(props);

  // 重置Hook上下文
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;

  // 此时workInProgress.memoizedState已经是完整的Hook链表
  return children;
}

// useState的简化实现
function useState(initialState) {
  // 获取当前Hook或创建新Hook
  const hook = getOrCreateCurrentHook();

  if (isMount) {
    // 首次渲染，初始化Hook状态
    const state = typeof initialState === 'function'
      ? initialState()
      : initialState;

    hook.memoizedState = state;
    hook.baseState = state;

    // 创建更新队列
    const queue = {
      pending: null,
      dispatch: dispatchAction.bind(null, currentlyRenderingFiber, hook.queue),
      lastRenderedReducer: basicStateReducer,
      lastRenderedState: state
    };
    hook.queue = queue;

    // 返回状态和更新函数
    return [state, queue.dispatch];
  } else {
    // 更新渲染，处理更新队列
    // ...处理更新逻辑

    // 返回最新状态和更新函数
    return [hook.memoizedState, hook.queue.dispatch];
  }
}
```

## 实际影响和最佳实践

### 状态设计差异

1. **类组件**：
   - 倾向于将相关状态组合在一个对象中
   - 状态更新是浅合并的，只需更新变化的部分
   - 状态之间的依赖关系隐含在代码逻辑中

   ```typescript
   class UserProfile extends React.Component {
     state = {
       user: {
         name: '',
         email: '',
         preferences: {}
       },
       isLoading: true,
       error: null
     };

     updateEmail = (email) => {
       // 只更新email，其他状态保持不变
       this.setState({
         user: {
           ...this.state.user,
           email
         }
       });
     };
   }
   ```

2. **函数组件**：
   - 倾向于将不相关的状态拆分为多个useState
   - 每个状态更新是完全替换的，需要手动处理对象合并
   - 状态之间的依赖关系通过依赖数组显式声明

   ```typescript
   function UserProfile() {
     // 状态拆分为多个独立的useState
     const [user, setUser] = useState({ name: '', email: '', preferences: {} });
     const [isLoading, setIsLoading] = useState(true);
     const [error, setError] = useState(null);

     const updateEmail = (email) => {
       // 需要手动合并对象
       setUser({
         ...user,
         email
       });
     };
   }
   ```

### 性能优化差异

1. **类组件**：
   - 使用`shouldComponentUpdate`或`PureComponent`优化渲染
   - 需要手动管理哪些状态变化应该触发重新渲染

   ```typescript
   class OptimizedComponent extends React.PureComponent {
     // PureComponent会对props和state进行浅比较
     // 只有当它们变化时才会重新渲染
   }
   ```

2. **函数组件**：
   - 使用`React.memo`、`useMemo`和`useCallback`优化渲染
   - 通过依赖数组精确控制计算和副作用的执行时机

   ```typescript
   const OptimizedComponent = React.memo(function({ value }) {
     // 只有当计算结果需要变化时才重新计算
     const expensiveResult = useMemo(() => {
       return computeExpensiveValue(value);
     }, [value]);

     // 只有当依赖变化时才创建新的回调函数
     const handleClick = useCallback(() => {
       doSomething(value);
     }, [value]);

     return <div onClick={handleClick}>{expensiveResult}</div>;
   });
   ```

### 最佳实践

1. **类组件**：
   - 将相关状态组合在一起，形成有意义的状态对象
   - 使用函数式更新避免状态依赖：`this.setState(state => ({ count: state.count + 1 }))`
   - 小心处理嵌套对象的更新，确保正确合并

2. **函数组件**：
   - 使用多个`useState`管理独立的状态片段
   - 对于复杂的状态逻辑，考虑使用`useReducer`
   - 注意Hook的调用顺序，不要在条件语句中使用Hook
   - 正确设置依赖数组，避免遗漏依赖或包含不必要的依赖

## 总结

类组件和函数组件的`memoizedState`在结构和工作原理上有着根本的不同：

- **类组件**：`memoizedState`是一个单一对象，直接存储组件的整个状态。状态更新是浅合并的，所有状态集中管理。

- **函数组件**：`memoizedState`是一个Hook链表，每个Hook有自己的状态。状态更新是独立的，各个Hook的状态分散管理。

这种差异反映了React团队对组件状态管理的不同设计理念：从集中式的状态管理转向更加分散和声明式的状态管理。理解这些差异有助于更好地设计组件状态，并在适当的场景选择合适的组件类型和状态管理策略。