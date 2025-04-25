# React Fiber 结构详细罗列

本文档详细罗列了React Fiber的完整结构，包括所有字段、类型和注释，以便深入理解React的内部工作机制。

## Fiber节点结构

以下是React源码中Fiber节点的完整结构定义（基于React 17版本）：

```typescript
// ReactFiber.js中的Fiber结构定义

export type Fiber = {
  // 标记fiber的类型
  tag: WorkTag,

  // 唯一标识，特别是在列表中用于区分不同元素
  key: null | string,

  // fiber类型，对应于函数/类组件、原生DOM等
  elementType: any,

  // 对于组件，type是函数/类本身
  // 对于原生DOM，type是字符串（如'div'）
  type: any,

  // 与此fiber关联的本地状态
  // 对于类组件，这是组件实例
  // 对于DOM元素，这是DOM节点
  stateNode: any,

  // ===== Fiber树结构 =====

  // 指向父fiber
  return: Fiber | null,

  // 指向第一个子fiber
  child: Fiber | null,

  // 指向下一个兄弟fiber
  sibling: Fiber | null,

  // 在兄弟节点中的索引
  index: number,

  // ref字段
  ref:
    | null
    | (((handle: mixed) => void) & { _stringRef: ?string, ... })
    | RefObject,

  // ===== 输入输出 =====

  // 新的props
  pendingProps: any,

  // 已经处理过的props
  memoizedProps: any,

  // 更新队列
  updateQueue: UpdateQueue<any> | null,

  // 已经处理过的state
  memoizedState: any,

  // 依赖项（用于context, suspense等）
  dependencies: Dependencies | null,

  // 模式标记（如StrictMode, ConcurrentMode等）
  mode: TypeOfMode,

  // ===== 副作用 =====

  // 标记此fiber上的副作用
  flags: Flags,

  // 子树中所有fiber的副作用标记的合并
  subtreeFlags: Flags,

  // 需要删除的子节点列表
  deletions: Array<Fiber> | null,

  // ===== 调度相关 =====

  // 优先级相关
  lanes: Lanes,

  // 子树中所有fiber的优先级的合并
  childLanes: Lanes,

  // 指向workInProgress树中对应的fiber
  // 或指向current树中对应的fiber
  alternate: Fiber | null,

  // 时间相关（仅在开发模式和分析模式下使用）
  actualDuration?: number,
  actualStartTime?: number,
  selfBaseDuration?: number,
  treeBaseDuration?: number,

  // 调试ID
  _debugID?: number,
  _debugSource?: Source | null,
  _debugOwner?: Fiber | null,
  _debugIsCurrentlyTiming?: boolean,
  _debugNeedsRemount?: boolean,

  // 其他内部使用的字段
  _debugHookTypes?: Array<HookType> | null,
};
```

## WorkTag类型

`tag`字段标识了Fiber节点的类型：

```typescript
export type WorkTag =
  | 0  // FunctionComponent - 函数组件
  | 1  // ClassComponent - 类组件
  | 2  // IndeterminateComponent - 初始挂载时未确定的组件类型
  | 3  // HostRoot - 根节点
  | 4  // HostPortal - Portal
  | 5  // HostComponent - 原生DOM元素
  | 6  // HostText - 文本节点
  | 7  // Fragment - Fragment
  | 8  // Mode - 模式，如StrictMode
  | 9  // ContextConsumer - Context.Consumer
  | 10 // ContextProvider - Context.Provider
  | 11 // ForwardRef - forwardRef
  | 12 // Profiler - Profiler
  | 13 // SuspenseComponent - Suspense
  | 14 // MemoComponent - React.memo包装的组件
  | 15 // SimpleMemoComponent - 简化版的React.memo
  | 16 // LazyComponent - React.lazy包装的组件
  | 17 // IncompleteClassComponent - 加载失败的类组件
  | 18 // DehydratedFragment - 服务端渲染相关
  | 19 // SuspenseListComponent - SuspenseList
  | 20 // ScopeComponent - Scope
  | 21 // OffscreenComponent - Offscreen
  | 22 // LegacyHiddenComponent - LegacyHidden
  | 23 // Cache - 缓存组件
  | 24 // TracingMarkerComponent - 追踪标记组件
;
```

## Flags（副作用标记）

`flags`字段标识了Fiber节点上需要执行的副作用：

```typescript
export type Flags = number;

// 副作用标记
export const NoFlags = /*                      */ 0b000000000000000000000000000;
export const PerformedWork = /*                */ 0b000000000000000000000000001;
export const Placement = /*                    */ 0b000000000000000000000000010;
export const Update = /*                       */ 0b000000000000000000000000100;
export const PlacementAndUpdate = /*           */ 0b000000000000000000000000110;
export const Deletion = /*                     */ 0b000000000000000000000001000;
export const ContentReset = /*                 */ 0b000000000000000000000010000;
export const Callback = /*                     */ 0b000000000000000000000100000;
export const DidCapture = /*                   */ 0b000000000000000000001000000;
export const Ref = /*                          */ 0b000000000000000000010000000;
export const Snapshot = /*                     */ 0b000000000000000000100000000;
export const Passive = /*                      */ 0b000000000000000001000000000;
export const PassiveUnmountPendingDev = /*     */ 0b000000000000010000000000000;
export const Hydrating = /*                    */ 0b000000000000100000000000000;
export const HydratingAndUpdate = /*           */ 0b000000000000100000000000100;
// ...更多标记
```

## TypeOfMode（模式类型）

`mode`字段标识了Fiber节点的渲染模式：

```typescript
export type TypeOfMode = number;

export const NoMode = /*            */ 0b000000;
export const StrictMode = /*        */ 0b000001;
export const BlockingMode = /*      */ 0b000010;
export const ConcurrentMode = /*    */ 0b000100;
export const ProfileMode = /*       */ 0b001000;
export const DebugTracingMode = /*  */ 0b010000;
export const OffscreenMode = /*     */ 0b100000;
```

## UpdateQueue（更新队列）

更新队列是一个环形链表，用于存储组件的状态更新：

```typescript
export type UpdateQueue<State> = {
  baseState: State,
  firstBaseUpdate: Update<State> | null,
  lastBaseUpdate: Update<State> | null,
  shared: {
    pending: Update<State> | null,
  },
  effects: Array<Update<State>> | null,
};

export type Update<State> = {
  eventTime: number,
  lane: Lane,
  tag: 0 | 1 | 2 | 3,
  payload: any,
  callback: (() => mixed) | null,
  next: Update<State> | null,
};

// 更新标记
export const UpdateState = 0;
export const ReplaceState = 1;
export const ForceUpdate = 2;
export const CaptureUpdate = 3;
```

## Lanes（优先级模型）

`lanes`字段用于表示Fiber节点的优先级：

```typescript
export type Lanes = number;
export type Lane = number;

// 优先级常量
export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;

export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;
export const SyncBatchedLane: Lane = /*                 */ 0b0000000000000000000000000000010;

export const InputDiscreteHydrationLane: Lane = /*      */ 0b0000000000000000000000000000100;
export const InputDiscreteLanes: Lanes = /*             */ 0b0000000000000000000000000011000;

export const InputContinuousHydrationLane: Lane = /*    */ 0b0000000000000000000000000100000;
export const InputContinuousLanes: Lanes = /*           */ 0b0000000000000000000000011000000;

// ...更多优先级常量
```

## Hook结构

Hook是函数组件中使用的特殊函数，它们允许在函数组件中使用状态和其他React特性：

```typescript
export type Hook = {
  memoizedState: any,
  baseState: any,
  baseQueue: Update<any, any> | null,
  queue: UpdateQueue<any, any> | null,
  next: Hook | null,
};

// 不同类型的Hook有不同的memoizedState结构

// useState/useReducer Hook的queue结构
type UpdateQueue<S, A> = {
  pending: Update<S, A> | null,
  dispatch: (A) => void | null,
  lastRenderedReducer: ((S, A) => S) | null,
  lastRenderedState: S | null,
};

// useEffect Hook的memoizedState结构
type Effect = {
  tag: HookEffectTag,
  create: () => (() => void) | void,
  destroy: (() => void) | void,
  deps: Array<mixed> | null,
  next: Effect | null,
};

// HookEffectTag
export const NoEffect = /*             */ 0b00000;
export const UnmountSnapshot = /*      */ 0b00001;
export const UnmountMutation = /*      */ 0b00010;
export const MountMutation = /*        */ 0b00100;
export const UnmountLayout = /*        */ 0b01000;
export const MountLayout = /*          */ 0b10000;
export const MountPassive = /*         */ 0b100000;
export const UnmountPassive = /*       */ 0b1000000;
```

## Dependencies（依赖项）

`dependencies`字段用于存储Context和Suspense的依赖项：

```typescript
export type Dependencies = {
  lanes: Lanes,
  firstContext: ContextDependency<mixed> | null,
  responders: Set<ResponderInstance> | null,
};

export type ContextDependency<T> = {
  context: ReactContext<T>,
  observedBits: number,
  next: ContextDependency<mixed> | null,
};
```

## Fiber树结构

Fiber节点通过以下字段构成一棵树：

- `return`: 指向父Fiber节点
- `child`: 指向第一个子Fiber节点
- `sibling`: 指向下一个兄弟Fiber节点

这种结构允许React使用深度优先遍历算法遍历整棵树。

## 双缓冲技术

React维护两棵Fiber树：

1. **current树**: 当前屏幕上显示的内容对应的Fiber树
2. **workInProgress树**: 正在构建的新Fiber树

每个Fiber节点通过`alternate`字段连接到另一棵树中对应的节点。当workInProgress树构建完成后，React会通过一次性操作（称为"commit"）将其变为current树。

## 总结

React Fiber是一种复杂的数据结构，它包含了大量字段来支持React的各种功能：

- 组件类型标识
- 树形结构导航
- 状态管理
- 副作用标记
- 优先级调度
- 双缓冲渲染

通过这种精心设计的数据结构，React能够实现增量渲染、优先级调度、并发模式等高级特性，为用户提供流畅的交互体验。