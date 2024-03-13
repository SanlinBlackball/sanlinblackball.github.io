---
layout:     post
title:      React 组件渲染
subtitle:   组件到底是怎么render的
date:       2023-12-27
author:     sanlinblackball
header-img: img/Ju-ITc1Cc0w.jpg
catalog: true
tags:
- React
- Diff
- Fiber
---
## 前言
在React开发中不可避免 我们总要用到`setState` 或者 `useState`相关以此来更新界面，但是在渲染中究竟发生了什么呢？让我们带着疑问一步一步看看。为了让读者看的明白清楚 我内容一份为三。

1.   [Fiber到底是什么](../../../../2023/11/27/React-组件渲染-fiber)
2.  组件到底是怎么render的
3.  commit 阶段react做了哪些事情


## render
`render` 是我们React中最关键的一步，它和页面的渲染息息相关，那到底他做了什么呢？

上面我们提到了在渲染的时候 在缓存中会存在一个(首次1个后续两个)`VDOM`，`render`经过对比前后两次的DOM的区别，并通过打标记的方式对发生的改变的地方进行增删改。简单说，render 过程就是 React 「对比旧 Fiber 树和新的 element」 然后「为新的 element 生成新 Fiber 树」的一个过程。让我们更详细一些。

根据源码我们得知 从源码中看，React 的整个核心流程开始于 `performSyncWorkOnRoot` 函数，在这个函数中，会根据`root`的`mode`来决定是同步模式还是异步模式，然后会根据`root`的`current`指针来找到当前的`Fiber`，然后会根据`current`指针的`tag`来进行不同的处理。

```js
function performSyncWorkOnRoot(root) {
  .....
  var exitStatus = renderRootSync(root, lanes);
  var finishedWork = root.current.alternate;
  root.finishedWork = finishedWork;
  root.finishedLanes = lanes;
  commitRoot(root, workInProgressRootRecoverableErrors, workInProgressTransitions);
  .....
}
```
上面就是整个核心流程起始地，我们发现，在 `performSyncWorkOnRoot`中我们主要执行了 `renderRootSync` 函数和 `commitRoot` 函数,一个是 `对比旧 Fiber 树和新的 element`，另外一个是用于`执行 Fiber 树中的更新操作并将更新应用到真实的 DOM 上（执行副作用操作）`。当然这里看不懂没有问题，我们会详细讲解一下这两个方法。

回到正题，我们继续 `render` 话题。整个 `render` 过程的重点在 `workLoopSync` ,代码如下。

从 `workLoopSync` 函数里我们可以看到，这里用了一个循环来不断调用 `performUnitOfWork` 方法，直到 workInProgress 为 null。所以这里的重点就是`performUnitOfWork`。而在`performUnitOfWork` 主要使用了 `beginWork` 和 `completeUnitOfWork` 来分别模拟这个“递”和“归”。而这两个方法使得递归过程可以中断。

`render` 过程是深度优先的遍历，`beginWork` 函数会为遍历到的每个 `Fiber` 节点生成他的所有 `子Fiber` 并返回第一个`子Fiber` ，这个 `子Fiber` 将赋值给 `workInProgress`，在下一轮循环继续处理，直到遍历到叶子节点，这时候就需要“归”了。

`completeUnitOfWork` 就会为叶子节点做一些处理，然后把叶子节点的兄弟节点赋值给 `workInProgress` 继续“递”操作，如果连兄弟节点也没有的话，就会往上处理父节点。流程可以查看上图。

### beginWork
#### 目的
更新当前节点`workInProgress`，获取新的 `children`。

生成他们对应的 Fiber，并最终返回第一个子节点。

React 通常会同时存在两个 Fiber 树，一个是`当前视图对应`的，一个则是`根据最新状态正在构建中`的。这两棵树的节点一一对应，通过对比这两个数进行更新。当然，当首次渲染的时候，当前视图对应的Fiber 树 必然为 null。
如果首次渲染，我们会根据正在构建的节点的组件类型做不同的处理。生成一棵 Fiber 树。这个不是我们关注的重点。这部分逻辑并没有太大的争议。我们将关注点放在 `非首次渲染` 上。
react 使用了哪些手段来优化更新效率呢？

当前**节点对应组件的 props 和 context 没有发生变化** 并且**当前节点的更新优先级不够** ，如果这两个条件均满足的话可以直接copy `当前视图对应` 的子节点并返回。如果不满足则同首次渲染走一样的逻辑（当然这些都是一些浅比较）。说完上面这些话题我们不得不拿出我们经典的话题。

``` jsx
function Son() {
  console.log('child render!');
  return <div>Son</div>;
}


function Parent(props) {
  const [count, setCount] = React.useState(0);

  return (
    <div onClick={() => {setCount(count + 1)}}>
      count:{count}
      <Son />
    </div>
  );
}


function App() {
  return (
    <Parent/>
  );
}

``` 
上面的例子 很明显  Son 组件不依赖任意于 props 和 context，但是当我们点击父组件的时候 会发现 Son 组件也会重新渲染。不是说  `节点对应组件的 props 和 context 没有发生变化` 就不会渲染了吗？ 我们先说一下解决方法，这个原因我们留一个悬疑待会再讲解
``` jsx
function Son() {
  console.log('child render!');
  return <div>Son</div>;
}


function Parent(props) {
  const [count, setCount] = React.useState(0);

  return (
    <div onClick={() => {setCount(count + 1)}}>
      count:{count}
      {props.children}
    </div>
  );
}


function App() {
  return (
    <Parent>
      <Son/>
    </Parent>
  );
}

``` 
上面的代码 会发现 Son 组件不会重新渲染。是因为 Son 组件是在 App 组件中生成并作为 props 传入 Parent 的，因为不管 Parent 组件状态怎么变化都不会影响到 App 组件，因此 App 和 Son 组件就只会在首次渲染时会执行一遍，也就是说 Parent 获取到的 props.children 的引用一直都是指向同一个对象，这样一来 Son 组件的 props 也就不会变化了。但是如果我将 Son 组件 放在 Parent 组件中 当 Parent 发生了setState 时 
```jsx
<Son/>
```
```jsx
React.createElement(Son, null)
```
相当于重新创建了 Son，由于props的引用改变，oldProps !== newProps。于是重新渲染了。

如果不理解可以查看卡颂老师的[文章](https://mp.weixin.qq.com/s/rTQNpWBC7re9ykpwba8IAg)。

#### 生成子节点

我们通过上面得到 workInProgress 的 children 之后，接下来需要为这些 children  生成 Fiber ，这就是 `reconcileChildFibers` 做的事情，这也是我们经常提到的 diff 的过程。
当然更新也分两种情况 更新的是单节点 （ 例如`<Son/>` ） 调用`reconcileSingleElement` ， 或者是数组（例如`map((item,index)=><Son key = {index} />)` ）调用`reconcileChildrenArray`。

##### 单节点
```jsx
  function reconcileSingleElement(returnFiber, currentFirstChild, element, lanes) {
      var key = element.key;
      var child = currentFirstChild;

      while (child !== null) {
        // 首先比较 key 是否相同
        if (child.key === key) {
          var elementType = element.type;

          if (elementType === REACT_FRAGMENT_TYPE) {

            if (child.tag === Fragment) {
              deleteRemainingChildren(returnFiber, child.sibling);
              var existing = useFiber(child, element.props.children);
              existing.return = returnFiber;

              {
                existing._debugSource = element._source;
                existing._debugOwner = element._owner;
              }

              return existing;
            }
          } else {
            // 然后比较 elementType 是否相同
            if (child.elementType === elementType || ( // Keep this check inline so it only runs on the false path:
             isCompatibleFamilyForHotReloading(child, element) ) || // Lazy types should reconcile their resolved type.
            // We need to do this after the Hot Reloading check above,
            // because hot reloading has different semantics than prod because
            // it doesn't resuspend. So we can't let the call below suspend.
            typeof elementType === 'object' && elementType !== null && elementType.$$typeof === REACT_LAZY_TYPE && resolveLazy(elementType) === child.type) {
              deleteRemainingChildren(returnFiber, child.sibling);

              var _existing = useFiber(child, element.props);

              _existing.ref = coerceRef(returnFiber, child, element);
              _existing.return = returnFiber;

              {
                _existing._debugSource = element._source;
                _existing._debugOwner = element._owner;
              }

              return _existing;
            }
          } // Didn't match.


          deleteRemainingChildren(returnFiber, child);
          break;
        } else {
          deleteChild(returnFiber, child);
        }
         // 遍历兄弟节点，看能不能找到 key 相同的节点
        child = child.sibling;
      }

      if (element.type === REACT_FRAGMENT_TYPE) {
        var created = createFiberFromFragment(element.props.children, returnFiber.mode, lanes, element.key);
        created.return = returnFiber;
        return created;
      } else {
        var _created4 = createFiberFromElement(element, returnFiber.mode, lanes);

        _created4.ref = coerceRef(returnFiber, currentFirstChild, element);
        _created4.return = returnFiber;
        return _created4;
      }
    }
```
上面代码看起来可能生涩一些 接下来我用一些白话，给大家讲解一下。
按照react 的渲染规则---尽可能复用旧节点的原则 会遍历旧节点，对每个遍历到的节点会做下面两个判断

1. key 是否相同
2. key 相同的情况下，elementType 是否相同.

我们按照这两个判断在丰富一下 
1. 如果 key 不相同，则直接调用 `deleteChild` 将这个 child `标记` 为删除，注意这里是标记，可能只是我们还没有找到那个对的节点，所以要继续执行`child = child.sibling`;遍历兄弟节点，直到找到那个对的节点。

2. 如果 key 相同，elementType 相同，那就是最理想的情况，找到了可以复用的节点，直接调用 `deleteRemainingChildren` 把剩余的兄弟节点标记删除，然后直接复用 child 返回。

3. 如果 key 相同，但 elementType 不同，这是最悲情的情况，我们找到了那个节点，可惜的是这个节点的 elementType 已经变了，那我们也不需要再找了，把 child 及其所有兄弟节点标记删除，跳出循环。直接创建一个新的节点。 

注意：  

1. `deleteRemainingChildren` 把 child 及其所有兄弟节点标记删除  

2. `deleteChild` 只删除当前节点。

3. 复用使用的是 `useFiber`  

4. 重新生成新的 Fiber 使用的 `createFiberFromXXX` 

##### 多节点（数组）
```jsx
function reconcileChildrenArray(returnFiber, currentFirstChild, newChildren, lanes) {
      
      ...

      var resultingFirstChild = null;
      var previousNewFiber = null;
      var oldFiber = currentFirstChild;
      var lastPlacedIndex = 0;
      var newIdx = 0;
      var nextOldFiber = null;

      for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
        if (oldFiber.index > newIdx) {
          nextOldFiber = oldFiber;
          oldFiber = null;
        } else {
          nextOldFiber = oldFiber.sibling;
        }

        var newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx], lanes);

        // 跳出循环
        if (newFiber === null) {
          
          if (oldFiber === null) {
            oldFiber = nextOldFiber;
          }

          break;
        }

        if (shouldTrackSideEffects) {
          if (oldFiber && newFiber.alternate === null) {
            deleteChild(returnFiber, oldFiber);
          }
        }

        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);

        if (previousNewFiber === null) {
          // TODO: Move out of the loop. This only happens for the first run.
          resultingFirstChild = newFiber;
        } else {
          // TODO: Defer siblings if we're not at the right index for this slot.
          // I.e. if we had null values before, then we want to defer this
          // for each null value. However, we also don't want to call updateSlot
          // with the previous one.
          previousNewFiber.sibling = newFiber;
        }

        previousNewFiber = newFiber;
        oldFiber = nextOldFiber;
      }

      if (newIdx === newChildren.length) {
        // We've reached the end of the new children. We can delete the rest.
        deleteRemainingChildren(returnFiber, oldFiber);

        if (getIsHydrating()) {
          var numberOfForks = newIdx;
          pushTreeFork(returnFiber, numberOfForks);
        }

        return resultingFirstChild;
      }

      if (oldFiber === null) {
        // If we don't have any more existing children we can choose a fast path
        // since the rest will all be insertions.
        for (; newIdx < newChildren.length; newIdx++) {
          var _newFiber = createChild(returnFiber, newChildren[newIdx], lanes);

          if (_newFiber === null) {
            continue;
          }

          lastPlacedIndex = placeChild(_newFiber, lastPlacedIndex, newIdx);

          if (previousNewFiber === null) {
            // TODO: Move out of the loop. This only happens for the first run.
            resultingFirstChild = _newFiber;
          } else {
            previousNewFiber.sibling = _newFiber;
          }

          previousNewFiber = _newFiber;
        }

        if (getIsHydrating()) {
          var _numberOfForks = newIdx;
          pushTreeFork(returnFiber, _numberOfForks);
        }

        return resultingFirstChild;
      } // Add all children to a key map for quick lookups.
      ......
    }
```

```jsx
function updateSlot(returnFiber, oldFiber, newChild, lanes) {
      
      var key = oldFiber !== null ? oldFiber.key : null;

      if (typeof newChild === 'string' && newChild !== '' || typeof newChild === 'number') {
        
        if (key !== null) {
          return null;
        }

        return updateTextNode(returnFiber, oldFiber, '' + newChild, lanes);
      }

      if (typeof newChild === 'object' && newChild !== null) {
        switch (newChild.$$typeof) {
          case REACT_ELEMENT_TYPE:
            {
              if (newChild.key === key) {
                return updateElement(returnFiber, oldFiber, newChild, lanes);
              } else {
                return null;
              }
            }

          ...
        }
        ...
      }

      

      return null;
    }
```
这个看起来就更复杂了，我们还是老样子拆解一下。

从上面我们可以看到，方法内有两个循环。

**第一轮循环中逻辑如下：**



同时遍历 `oldFiber` 和 `newChildren，`判断 `oldFiber` 和 `newChild` 的 key 是否相同。

    如果 key 相同。
        如果elementType相同
            复用 oldFiber 返回。
        如果不同
            新建 Fiber 返回。
    如果 key 不同则
        直接跳出循环。

可以看出，第一轮循环只要新旧的 key 不一样，就会跳出循环

跳出循环后，要先执行两个判断

newChildren 已经遍历完了：这种情况说明新的 children 全都已经处理完了，只要把 oldFiber 和他所有剩余的兄弟节点删除然后返回头部的 Fiber 即可。

![fiber](/img/fiber_array_map_1.jpg)


已经没有 oldFiber ：这种情况说明 children 有新增的节点，给这些新增的节点逐一构建 Fiber 并链接上，然后返回头部的 Fiber 即可。

![fiber](/img/fiber_array_map.jpg)

如果以上两种情况都不是，则进入第二轮循环。第二轮循环比第一轮更复杂一些，给大家拆解一下。

```jsx
var existingChildren = mapRemainingChildren(returnFiber, oldFiber); // Keep scanning and use the map to restore deleted items as moves.

      for (; newIdx < newChildren.length; newIdx++) {
        var _newFiber2 = updateFromMap(existingChildren, returnFiber, newIdx, newChildren[newIdx], lanes);

        if (_newFiber2 !== null) {
          if (shouldTrackSideEffects) {
            // 副本
            if (_newFiber2.alternate !== null) {
              // The new fiber is a work in progress, but if there exists a
              // current, that means that we reused the fiber. We need to delete
              // it from the child list so that we don't add it to the deletion
              // list.
              existingChildren.delete(_newFiber2.key === null ? newIdx : _newFiber2.key);
            }
          }

          lastPlacedIndex = placeChild(_newFiber2, lastPlacedIndex, newIdx);

          if (previousNewFiber === null) {
            resultingFirstChild = _newFiber2;
          } else {
            previousNewFiber.sibling = _newFiber2;
          }

          previousNewFiber = _newFiber2;
        }
      }
```
在执行第二轮循环之前，先把剩下的旧节点和他们对应的 key 或者 index 做成映射(Map)，方便查找。

第二轮循环沿用了第一轮循环的 newIdx，很容易看出第二轮的循环是在第一轮的基础上对newChildren进行处理。

在代码中我们可以发现 第二轮循环先调用了`updateFromMap` 来处理节点，参数为`existingChildren, returnFiber, newIdx`等 

```jsx
function updateFromMap(existingChildren, returnFiber, newIdx, newChild, lanes) {

    ...

    if (typeof newChild === 'object' && newChild !== null) {
      switch (newChild.$$typeof) {
        case REACT_ELEMENT_TYPE:
          {
            var _matchedFiber = existingChildren.get(newChild.key === null ? newIdx : newChild.key) || null;

            return updateElement(returnFiber, _matchedFiber, newChild, lanes);
          }

        .....
      }
    }

    return null;
  }
```

也就是说我们需要用 `newChild` 的 `key` 去 `existingChildren` 中找对应的 Fiber。上面我们也用到了 `updateElement` 这个函数，我们可以得知这个函数主要的目的 

      能找到 key 相同的，(这个节点只是位置变了)，可以复用的

      找不到 key 相同的，则说明这个节点应该是新增的。

```jsx
function placeChild(newFiber, lastPlacedIndex, newIndex) {
    newFiber.index = newIndex;

    ...

    var current = newFiber.alternate;

    if (current !== null) {
      var oldIndex = current.index;

      if (oldIndex < lastPlacedIndex) {
        // This is a move.
        newFiber.flags |= Placement;
        return lastPlacedIndex;
      } else {
        // This item can stay in place.
        return oldIndex;
      }
    } else {
      // This is an insertion.
      newFiber.flags |= Placement;
      return lastPlacedIndex;
    }
  }
```
举一个例子   

    旧         1,2,3,4,5,

    新         1,2,5,3,4
              <!-- 第一次循环 -->
               1,2
              <!-- 第二次循环 -->
                <!-- 找到5 -->
                1,2,5
                <!-- 找到3 -->
                1,2,5,3
                <!-- 找到4 -->
                1,2,5,3,4
按照上面的说法我们应该如上面操作 但是很明显可以看出遍历完`1、2`，以后如果直接移动 `5` 到最后就不用这么多步骤了，或者移动`3、4` 到`5`的前面也比现在这个方案好。

那问题的根本就是   
**到底哪个节点才是移动了的?**   
这就需要一个参照点，我们要保证在参照点左边都是排好序了。而这个参照点就是 `lastPlacedIndex`。有了它，我们在遍历 newChildren 的时候可能会出现下面两种情况

      Fiber(新建或者复用) 对应的老 index < lastPlacedIndex，这就说明这个 Fiber 的位置不对，因为 lastPlacedIndex 左边的应该全是已经遍历过的 newChild 生成的 Fiber。因此这个 Fiber 是需要被移动的，打上 'flag'。

      如果 Fiber 对应的老 index >= lastPlacedIndex，那就说明这个 Fiber 的相对位置是 ok 的，可以不用移动，但是我们需要更新一下参照点，把参照点更新成这个 Fiber 对应的老 index。

这么说可能有点生涩，我们举一个例子来说明一下吧。

    旧         a,b,c,d

    新         c,a,b,d,e


    lastPlacedIndex = 0
    <!-- 开始操作 -->
      <!-- 操作C -->
        newC.index = 0 oldC.index = 2
        oldC.index > lastPlacedIndex
          lastPlacedIndex = 2   
      <!-- 操作A -->
        newA.index = 1 oldA.index = 0
        oldA.index < lastPlacedIndex
          oldA.flag = Placement
      <!-- 操作B -->
        newB.index = 2 oldB.index = 1
        oldB.index < lastPlacedIndex
          oldB.flag = Placement
      <!-- 操作D -->
        newD.index = 3 oldD.index = 3
        oldD.index > lastPlacedIndex
          lastPlacedIndex = 3
       <!-- 操作E -->
        newE.index = 3 oldE.index = 0
        oldE.index < lastPlacedIndex
          oldE.flag = Placement

从上面我们可以看出 a b e 发生了移动，但是更要以最高效方法应该是 移动 c e。 从这里我们也得到一个注意事项   
 **尽量避免把节点从后面提到前面** 

### completeUnitOfWork

#### 目的
1. **调用completeWork**：当一个Fiber节点被处理完毕（即没有更多的子节点），completeUnitOfWork会将渲染的结果提交到实际的DOM节点。
2. **回溯到父节点**：如果没有更多的兄弟节点需要处理，completeUnitOfWork会通过子->父链表回溯到父节点，并继续上述过程，直到回溯到根节点。

#### 处理当前节点
这里主要调用方法为 `completeWork` 主要两种情况：
首次渲染和更新节点
##### 首次渲染
     创建真实 DOM。

     如果有子节点的话将子节点的真实 DOM 插入到刚刚创建的 DOM 中。

     处理真实 DOM 的 props 等。

这里我们以HostComponent为例子讲解下：    


```jsx
  var type = workInProgress.type;
  // 为fiber创建对应DOM节点
  var instance = createInstance(type, newProps, rootContainerInstance, currentHostContext, workInProgress);
  // 将子孙DOM节点插入刚生成的DOM节点中
  appendAllChildren(instance, workInProgress, false, false);
            workInProgress.stateNode = instance;
  
  // 处理prop 
  if (finalizeInitialChildren(instance, type, newProps, rootContainerInstance)) {
      markUpdate(workInProgress);
  }

```

##### 非首次渲染

当 update 时，Fiber 节点已经存在对应 DOM 节点，所以不需要生成 DOM 节点。需要做的主要是处理DOM 节点的 props，这里主要就是一些真实 DOM 的 onClick、onChange等回调函数的注册，style 等，这些处理完之后的 props 也会记录到 workInProgress.updateQueue 中，并在 commit 阶段更新到 DOM 节点上。

#### 回溯
开始我们提到是 beginWork 是深度优先的更新，也就意味着进入 `completeUnitOfWork` 后,一定还需要回到 `beginWork` 中继续处理其他的节点。

```jsx
    var siblingFiber = completedWork.sibling;

    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      workInProgress = siblingFiber;
      return;
    } // Otherwise, return to the parent


    completedWork = returnFiber; // Update the next thing we're working on in case something throws.

    workInProgress = completedWork;
```
可以看到，当处理完当前节点之后，React 会判断当前节点是否具有兄弟节点，如果有的话则将兄弟节点设置为当前的 workInProgress 回到主流程继续 `beginWork`。

而如果没有兄弟节点的话，就意味着同父节点下的所有子节点都已经处理完毕，则接下来就会处理他们的父节点。

大致流程就是：`beginWork` 执行到当前节点没有 child 的时候，进入 `completeUnitOfWork` 处理当前节点，处理完后如果当前节点有兄弟节点则回到 `beginWork` 继续处理兄弟节点，如果没有兄弟节点则继续在 `completeUnitOfWork` 处理当前节点的父节点，直到回溯到根结点上。