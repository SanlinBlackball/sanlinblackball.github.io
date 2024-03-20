---
layout:     post
title:      React 组件渲染
subtitle:   commit 阶段react做了哪些事情
date:       2024-01-27
author:     sanlinblackball
header-img: img/Ju-ITc1Cc0w.jpg
catalog: true
tags:
- React
- Diff
- Fiber
---

## 前言
在React开发中不可避免 我们总要用到`setState` 或者 `useState`相关以此来更新界面，但是在渲染中究竟发生了什么呢？让我们带着疑问一步一步看看。为了让读者看的明白清楚 我内容一分为三。

1.   [Fiber到底是什么](../../../../2023/11/27/React-组件渲染-fiber)
2.  [组件到底是怎么render的](../../../../2023/12/27/React-组件渲染-render)
3.  commit 阶段react做了哪些事情
## 挂载（commitRoot）
### flags （effectTag）
从上一篇文章render我们也对 `flags` 有了小小的了解。需要被操作的 Fiber 都会被打上 `flags` 标记。

```jsx
var NoFlags =
/*                      */
0;

var Placement =
/*                    */
2;
var Update =
/*                       */
4;
var ChildDeletion =
/*                */
16;

```
在以前的版本中还存在
```jsx
var PlacementAndUpdate =
/*           */
6;
var Deletion =
/*                     */
8;
```
根据上面我们很容易发现 `PlacementAndUpdate` 是通过`Placement` 和 `Update` 相加得出的为0 ，在查看源码我们也发现通过按位与运算来判断是何种改变而后对`dom`进行操作。(新版本中)

在React 18中，提交阶段 `commitRoot` 的处理有所改变。

在`React 17`及之前的版本中，React在完成一次更新（也就是构建完current Fiber树）后，会把所有需要执行副作用的fiber节点用链表串联起来，形成`effect list`。这样做的目的是为了在commit阶段能快速找到所有有副作用的fiber节点，进行相应的DOM操作。这个链表的第一个节点存储在`finishedWork.firstEffect`，通过每个节点上的`nextEffect`可以遍历整个链表。

但在`React 18`以后，这个方案改变了，现在React在commit阶段不再使用`effect list`。取而代之的是，在新的Fiber架构中所有fiber节点的child、sibling和return指针形成的树形结构。此时commit阶段的处理方式变为深度优先遍历这个Fiber树。这是为了配合`React 18`新引入的`并发模式`(`concurrent mode`)和其他的新特性。

在`React 18`中，`finishedWork`上的`firstEffect`字段被废弃，原本存放在各个Fiber节点`effectTag`上的副作用标识，改为存放在`flags`字段上，不再使用`effectTag`。   
当读完上面的文字，我们会有一些疑问。在render阶段已经进行了深度遍历。已经找到了需要改变fiber并且打上了标签。如果在`commit阶段`还要进行深度遍历一次。这不是资源浪费吗？

的确，在`React 17`及之前的版本中，React在`render阶段`就已经遍历过一次Fiber树，并在此过程中标记了哪些Fiber节点有副作用，然后将这些有副作用的节点用链表串联起来，在`commit阶段`只需要遍历这个链表即可。这确实会比在`commit阶段`重新遍历整个Fiber树要性能更好。

为什么在React 18中，React仍然改回在`commit阶段`遍历整个Fiber流程，这在初看起来似乎是一种效率损失，但这实际上是React在**为了引入新的并发模式做的权衡和妥协**。

在`React 17`之前，React并没有真正的并发机制。虽然`React 16`开始引入了Fiber机制，可以打断任务和恢复任务，但仍无法同时处理多个任务。而`React 18`带来了`并发模式`（`concurrent mode`），此模式下，React可以在内存中同时处理多个任务。

并发模式下，可能存在多个任务同时在内存中进行，每个任务都可能产生不同的更新和副作用，因此会有多个不同的Fiber树在内存中同时存在。如果还像`React 17`那样只通过链表来管理副作用，那么这个链表将会变得非常复杂，因为需要同时管理多个任务产生的副作用。

相比之下，如果在`commit阶段`遍历整个Fiber树，虽然在性能上可能较17有所损失，但在代码复杂度和可维护性上有了提升。并且，对于绝大多数应用来说，这样的性能损失是可以接受的，因为`commit阶段`通常只占整个更新过程的一小部分时间。

所以 React团队是在“在`commit阶段`节约CPU时间”和“支持并发模式，简化代码维护难度”之间做出了权衡。再加上通过对React的调度方式进行优化，从综合效率上来看，并不会造成大的性能损失。
为了便于好理解，这里我们对 `React 18`以前的处理方式进行讲解。去除别的干扰。
### effectList
为了便于好理解 下面我举一个例子。
```jsx
function App() {
  const [count, setCount] = useState(0);
  return (
    <div onClick={() => setCount(count + 1)}>
      <p>{count}</p>
      <span>{count}</span>
    </div>
  );
};
```
当我们 执行例子，点击`div` 时，在 `performSyncWorkOnRoot` 方法 的末尾的 `finishedWork`的 `firstEffect` 我们会发现 当前的effectList 是类似于下面的情况。

![fiber](/img/commitroot_pic_1.png)

在例子中，我们通过 `setCount` 进行重新渲染界面。 `p` 和`span` 因为count发生了变化重新渲染。`div` 因重新创建了 `onClick` 也重新渲染。因为是深度遍历所以 firstEffect为 `p`。
```jsx
function commitRoot(root) {
  const renderPriorityLevel = getCurrentPriorityLevel();
  runWithPriority(
    ImmediateSchedulerPriority,
    commitRootImpl.bind(null, root, renderPriorityLevel),
  );
  return null;
}
```
在commitRoot中同时使用到了渲染优先级和调度优先级,本节不再赘述优先级. 最后的实现是通过commitRootImpl函数。
```jsx
// ... 省略部分无关代码
function commitRootImpl(root, renderPriorityLevel) {

  // ============ 渲染前: 准备 ============
  do {
    flushPassiveEffects();
  } while (rootWithPendingPassiveEffects !== null);


  const finishedWork = root.finishedWork;
  const lanes = root.finishedLanes;

  // 清空FiberRoot对象上的属性
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  root.callbackNode = null;

  if (root === workInProgressRoot) {
    // 重置全局变量
    workInProgressRoot = null;
    workInProgress = null;
    workInProgressRootRenderLanes = NoLanes;
  }

  // 再次更新副作用队列
  let firstEffect;
  if (finishedWork.flags > PerformedWork) {
    // 默认情况下fiber节点的副作用队列是不包括自身的
    // 如果根节点有副作用, 则将根节点添加到副作用队列的末尾
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork;
      firstEffect = finishedWork.firstEffect;
    } else {
      firstEffect = finishedWork;
    }
  } else {
    firstEffect = finishedWork.firstEffect;
  }

  // ============ 渲染 ============
  let firstEffect = finishedWork.firstEffect;
  if (firstEffect !== null) {
    const prevExecutionContext = executionContext;
    executionContext |= CommitContext;
    // 阶段1: dom突变之前
    nextEffect = firstEffect;
    do {
      commitBeforeMutationEffects();
    } while (nextEffect !== null);

    // 阶段2: dom突变, 界面发生改变
    nextEffect = firstEffect;
    do {
      commitMutationEffects(root, renderPriorityLevel);
    } while (nextEffect !== null);
    // 恢复界面状态
    resetAfterCommit(root.containerInfo);
    // 切换current指针
    root.current = finishedWork;

    // 阶段3: layout阶段, 调用生命周期componentDidUpdate和回调函数等
    nextEffect = firstEffect;
    do {
      commitLayoutEffects(root, lanes);
    } while (nextEffect !== null);
    nextEffect = null;
    executionContext = prevExecutionContext;
  }

  // ============ 渲染后: 重置与清理 ============
  if (rootDoesHavePassiveEffects) {
    // 有被动作用(使用useEffect), 保存一些全局变量
  } else {
    // 分解副作用队列链表, 辅助垃圾回收
    // 如果有被动作用(使用useEffect), 会把分解操作放在flushPassiveEffects函数中
    nextEffect = firstEffect;
    while (nextEffect !== null) {
      const nextNextEffect = nextEffect.nextEffect;
      nextEffect.nextEffect = null;
      if (nextEffect.flags & Deletion) {
        detachFiberAfterEffects(nextEffect);
      }
      nextEffect = nextNextEffect;
    }
  }
  // 重置一些全局变量(省略这部分代码)...
  // 下面代码用于检测是否有新的更新任务
  // 比如在componentDidMount函数中, 再次调用setState()

  // 1. 检测常规(异步)任务, 如果有则会发起异步调度(调度中心`scheduler`只能异步调用)
  ensureRootIsScheduled(root, now());
  // 2. 检测同步任务, 如果有则主动调用flushSyncCallbackQueue(无需再次等待scheduler调度), 再次进入fiber树构造循环
  flushSyncCallbackQueue();

  return null;
}
```
上面的代码比较多 我们可以先不用完全理解，在这个代码中我们把这个过程分为三个阶段：  

1. 渲染前
2. 渲染
3. 渲染后  


#### 渲染前

```jsx
function commitRootImpl(root, renderPriorityLevel) {
  do {
    flushPassiveEffects();
  } while (rootWithPendingPassiveEffects !== null);
 
  // ...
}
```
在方法起始阶段，我们看到方法先判断 `rootWithPendingPassiveEffects` 是否为 `null`，如果不为 `null` 就会执行 `flushPassiveEffects。`    
这个 `PassiveEffects` 指的就是 那些被推迟执行的被动效应（passive effects），也就是由 useEffect hook 创建的副作用函数。这里我们要清楚一个概念：**useEffect 中的副作用函数是在渲染到真实 DOM 后才会执行的**。
这里的判断执行的是当前是否还有未执行的 useEffect，如果有，就执行它，也就是说在开启新一轮的 commit 阶段时会先等待上一轮的 useEffect 执行完。

```jsx
function commitRootImpl(root, renderPriorityLevel) {
  // ....
  const finishedWork = root.finishedWork;
  const lanes = root.finishedLanes;

  // 清空FiberRoot对象上的属性
  root.finishedWork = null;
  root.finishedLanes = NoLanes;
  root.callbackNode = null;

  if (root === workInProgressRoot) {
    // 重置全局变量
    workInProgressRoot = null;
    workInProgress = null;
    workInProgressRootRenderLanes = NoLanes;
  }
  
  // ...
}
```

接着会重置 render阶段使用到的一些全局变量。表示准备开始渲染工作。

```jsx
function commitRootImpl(root, renderPriorityLevel) {

  // ...

  // 再次更新副作用队列
  let firstEffect;
  if (finishedWork.flags > PerformedWork) {
    // 默认情况下fiber节点的副作用队列是不包括自身的
    // 如果根节点有副作用, 则将根节点添加到副作用队列的末尾
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork;
      firstEffect = finishedWork.firstEffect;
    } else {
      firstEffect = finishedWork;
    }
  } else {
    firstEffect = finishedWork.firstEffect;
  }
  // ...
}
```
 
 上面说过 `render阶段` 已经把带有 `effectTag` 的 fiber 节点连接形成一条链表了，这里再次处理 `effect list` 是因为这条链表目前只有子节点，并没有挂载根节点。如果根节点也存在 `effectTag`，那么就需要把根节点拼接到链表的末尾，形成一条完整的 `effect list`。

 总结来说渲染前做了以下操作:  

1. 处理上次更新的副作用队列
2. 设置全局状态(如: 更新fiberRoot上的属性)
重置全局变量(如: workInProgressRoot, workInProgress等)
3. 再次更新副作用队列: 只针对根节点fiberRoot.finishedWork    

    1. 默认情况下根节点的副作用队列是不包括自身的, 如果根节点有副作用, 则将根节点添加到副作用队列的末尾
    2. 注意只是延长了副作用队列, 但是fiberRoot.lastEffect指针并没有改变.

#### 渲染中
`commitRootImpl`函数中, 渲染阶段的主要逻辑是处理副作用队列, 将最新的 虚拟DOM 节点渲染到真实dom上。
整个渲染过程被分为 3 个阶段:

1. **commitBeforeMutationEffects**   

    dom 变更之前, 处理副作用队列中带有`Snapshot`,`Passive`标记的fiber节点。
2. **commitMutationEffects**

    dom 变更, 界面得到更新. 处理副作用队列中带有`Placement`, `Update`, `Deletion`, `Hydrating`标记的fiber节点。
3. **commitLayoutEffects**

    dom 变更后, 处理副作用队列中带有`Update` | `Callback` 标记的fiber节点。

##### commitBeforeMutationEffects 

```jsx
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    const current = nextEffect.alternate;
    const flags = nextEffect.flags;
    // 处理`Snapshot`标记
    if ((flags & Snapshot) !== NoFlags) {
      commitBeforeMutationEffectOnFiber(current, nextEffect);
    }
    // 处理`Passive`标记
    if ((flags & Passive) !== NoFlags) {
      // Passive标记只在使用了hook, useEffect会出现. 所以此处是针对hook对象的处理
      if (!rootDoesHavePassiveEffects) {
        rootDoesHavePassiveEffects = true;
        scheduleCallback(NormalSchedulerPriority, () => {
          flushPassiveEffects();
          return null;
        });
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```
我们再看看 `commitBeforeMutationEffectOnFiber` 做了什么事情
```
function commitBeforeMutationLifeCycles(
  current: Fiber | null,
  finishedWork: Fiber,
): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
    case Block: {
      return;
    }
    case ClassComponent: {
      if (finishedWork.flags & Snapshot) {
        if (current !== null) {
          const prevProps = current.memoizedProps;
          const prevState = current.memoizedState;
          const instance = finishedWork.stateNode;

          const snapshot = instance.getSnapshotBeforeUpdate(
            finishedWork.elementType === finishedWork.type
              ? prevProps
              : resolveDefaultProps(finishedWork.type, prevProps),
            prevState,
          );
          instance.__reactInternalSnapshotBeforeUpdate = snapshot;
        }
      }
      return;
    }
    case HostRoot: {
      if (supportsMutation) {
        if (finishedWork.flags & Snapshot) {
          const root = finishedWork.stateNode;
          clearContainer(root.containerInfo);
        }
      }
      return;
    }
    case HostComponent:
    case HostText:
    case HostPortal:
    case IncompleteClassComponent:
      return;
  }
}
```
从源码中可以看出, 与`Snapshot`标记相关的类型只有`ClassComponent`和`HostRoot`.

1. 对于`ClassComponent`类型节点, 调用了`instance.getSnapshotBeforeUpdate`生命周期函数。
2. 对于`HostRoot`类型节点, 调用`clearContainer`清空了容器节点(即div#root这个 dom 节点)。

在处理完 `Snapshot` 以后, 接下来就是处理 `Passive` 标记了。但是要注意这里并不是立即执行，而是把它放在 `scheduleCallback` 的回调当中，`scheduleCallback`方法会以一个优先级异步执行它的回调函数。
我们说清楚一些就是，如果存在 `Passive` ，则把 `rootDoesHavePassiveEffects` 置为 true，并且调度 `flushPassiveEffects`，而整个 commit阶段 是**同步执行** 的，所以 `useEffect` 的回调函数其实会在 commit阶段 **完成后** 再异步执行。

我们按照上面说的做一个小测试 （答案在最底部）
```jsx
function App() {
  console.log(1);
  useEffect(() => {
    console.log(2);
  });
  console.log(3);
  Promise.resolve(() => {
    console.log(4);
  });
  return <div>test</div>;
}
```
##### commitMutationEffects

这个阶段 dom 变更, 界面得到更新， 处理副作用队列中带有`ContentReset`, `Ref`, `Placement`, `Update`, `Deletion`, `Hydrating` 标记的fiber节点.

```
// ...省略部分无关代码
function commitMutationEffects(
  root: FiberRoot,
  renderPriorityLevel: ReactPriorityLevel,
) {
  // 处理Ref
  if (flags & Ref) {
    const current = nextEffect.alternate;
    if (current !== null) {
      // 先清空ref, 在commitRoot的第三阶段(dom变更后), 再重新赋值
      commitDetachRef(current);
    }
  }
  // 处理DOM突变
  while (nextEffect !== null) {
    const flags = nextEffect.flags;
    const primaryFlags = flags & (Placement | Update | Deletion | Hydrating);
    switch (primaryFlags) {
      case Placement: {
        // 新增节点
        commitPlacement(nextEffect);
        nextEffect.flags &= ~Placement; // 注意Placement标记会被清除
        break;
      }
      case PlacementAndUpdate: {
        // Placement
        commitPlacement(nextEffect);
        nextEffect.flags &= ~Placement;
        // Update
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Update: {
        // 更新节点
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      case Deletion: {
        // 删除节点
        commitDeletion(root, nextEffect, renderPriorityLevel);
        break;
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```

处理 DOM 有以下集中情况:

1. **新增**: commitPlacement -> insertOrAppendPlacementNode -> appendChild    

2. **更新**: commitWork -> commitUpdate    


3. **删除**: commitDeletion -> removeChild

最终会调用`appendChild`, `commitUpdate`, `removeChild`, 它们是HostConfig协议(源码在 ReactDOMHostConfig.js 中)中规定的标准函数, 在渲染器react-dom包中进行实现. 这些函数就是直接操作 DOM, 所以执行之后, 界面也会得到更新。

**注意**: `commitMutationEffects`执行之后, 在`commitRootImpl`函数中切换当前fiber树(`root.current = finishedWork`),保证fiberRoot.current指向代表当前界面的fiber树.

##### commitLayoutEffects
dom 变更后, 处理副作用队列中带有Update, Callback, Ref标记的fiber节点。
```jsx
// ...省略部分无关代码
function commitLayoutEffects(root: FiberRoot, committedLanes: Lanes) {
  while (nextEffect !== null) {
    const flags = nextEffect.flags;
    // 处理 Update和Callback标记
    if (flags & (Update | Callback)) {
      const current = nextEffect.alternate;
      commitLayoutEffectOnFiber(root, current, nextEffect, committedLanes);
    }
    if (flags & Ref) {
      // 重新设置ref
      commitAttachRef(nextEffect);
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```
核心逻辑都在`commitLayoutEffectOnFiber->commitLifeCycles`函数中.
```jsx
// ...省略部分无关代码
function commitLifeCycles(
  finishedRoot,
  current,
  finishedWork,
  committedLanes,
): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
    case Block:
      {
        {
          commitHookEffectListMount(Layout | HasEffect, finishedWork);
        }

        schedulePassiveEffects(finishedWork);
        return;
      }
    case ClassComponent: {
      const instance = finishedWork.stateNode;
      if (finishedWork.flags & Update) {
        if (current === null) {
          // 初次渲染: 调用 componentDidMount
          instance.componentDidMount();
        } else {
          const prevProps =
            finishedWork.elementType === finishedWork.type
              ? current.memoizedProps
              : resolveDefaultProps(finishedWork.type, current.memoizedProps);
          const prevState = current.memoizedState;
          // 更新阶段: 调用 componentDidUpdate
          instance.componentDidUpdate(
            prevProps,
            prevState,
            instance.__reactInternalSnapshotBeforeUpdate,
          );
        }
      }
      const updateQueue: UpdateQueue<*> | null =
        (finishedWork.updateQueue: any);
      if (updateQueue !== null) {
        // 处理update回调函数 如: this.setState({}, callback)
        commitUpdateQueue(finishedWork, updateQueue, instance);
      }
      return;
    }
    case HostComponent: {
      const instance: Instance = finishedWork.stateNode;
      if (current === null && finishedWork.flags & Update) {
        const type = finishedWork.type;
        const props = finishedWork.memoizedProps;
        // 设置focus等原生状态
        commitMount(instance, type, props, finishedWork);
      }
      return;
    }
  }
}
```

1. **FunctionComponent**  

    1. 会把 `HookLayout` 这个 tag 类型传给 `commitHookEffectListMount` 方法，也就是说这里会执行 `useLayoutEffect` 的回调函数。

    1. 接着会执行 `schedulePassiveEffects` 方法，在这里会分别注册 `useEffect` 销毁函数和回调函数，其实也就是把 `effect` 分别推进 `pendingPassiveHookEffectsUnmount` 和 `pendingPassiveHookEffectsMount` 这两个数组中，**后续** 取出来执行。
2. **ClassComponent**
    1. 如果 `current` 为空，也就是这个节点是首次 render，则会执行它的 `componentDidMount` 生命周期方法，否则会执行 `componentDidUpdate` 方法。
    2. 处理 `update` 回调函数 如: `this.setState({}, callback)`。

3. **HostComponent**

    1. 如果有 `Update` 标记, 需要设置一些原生状态(如: `focus` 等)

#### 渲染后
执行完上面内容以后, 渲染任务就已经完成了。 在渲染完成后, 需要做一些重置和清理工作:

1. 清除副作用队列

      1. 由于副作用队列是一个链表, 由于单个fiber对象的引用关系, 无法被gc回收.
      2. 将链表全部拆开, 当fiber对象不再使用的时候, 可以被gc回收.
2. 检测更新   

      1. 在整个渲染过程中, 有可能产生新的update(比如在 `componentDidMount` 函数中, 再次调用setState())。
      2. 如果是常规(异步)任务, 不用特殊处理, 调用 `ensureRootIsScheduled` 确保任务已经注册到调度中心即可。
      3. 如果是同步任务, 则主动调用 `flushSyncCallbackQueue` (无需再次等待 scheduler 调度), 再次进入 `fiber` 树构造循环。


答案
```jsx
输出顺序为 1, 3, 4, 2
```