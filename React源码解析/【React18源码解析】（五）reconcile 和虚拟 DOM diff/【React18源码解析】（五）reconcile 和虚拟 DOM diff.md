React 的更新可以分为三个阶段：

1. Schedule 调度阶段：决定要如何执行更新，执行哪些更新。

2. Render 渲染阶段：构建新的 Fiber 树，虚拟 DOM diff，标记副作用。

3. Commit 提交阶段：将变更真正提交到 DOM。

`reconcile` 是 render 阶段的核心部分之一，用于比较当前 Fiber 树与最新更新之间的差异，生成新的 Fiber 树，并标记副作用（如插入、更新、删除），以供 commit 阶段使用。

# 如何进入 reconcile

render 阶段的 `workLoop` -> `beginWork` 中，会根据 Fiber 节点的类型调用对应的 update 逻辑。

不同类型节点的 update 逻辑基本保持一致的“套路”：通过当前 Fiber 节点上的信息获取 `nextChildren`，即子节点的虚拟 DOM 对象，然后调用 `reconcile`。（具体过程上一章讲过，可以看[这里](url)）

```ts
reconcileChildren(current, workInProgress, nextChildren, renderLanes);
```

## reconcileChildren

```ts
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes
) {
  if (current === null) {
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes
    );
  } else {
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes
    );
  }
}
```

**参数**：

- `current`：旧 Fiber 节点。
- `workInProgress`：当前正在构建的新 Fiber 节点。
- `nextChildren`：当前 Fiber 节点子节点的虚拟 DOM 对象。
- `renderLanes`：本次渲染优先级。

**实现**：

通过 `current` 是否为 `null` 判断是否是首次挂载：

- 首次挂载调用 `mountChildFibers`，直接构建一组新的子 Fiber 节点，不涉及旧 Fiber 的复用。

- 非首次调用 `reconcileChildFibers`，会尝试复用旧 Fiber 节点。

**调用这两个函数返回当前 Fiber 节点的子节点，放在 `workInProgress.child` 上**。

这两个函数其实是通过高阶函数 `ChildReconciler` 创建的：

```ts
export const reconcileChildFibers = ChildReconciler(true);
export const mountChildFibers = ChildReconciler(false);
```

## ChildReconciler

```ts
function ChildReconciler(shouldTrackSideEffects) {
  function deleteChild() {}
  function deleteRemainingChildren() {}
  function useFiber() {}
  function placeChild() {}
  function reconcileChildrenArray() {}
  function reconcileSingleTextNode() {}
  function reconcileSingleElement() {}
  ...
  function reconcileChildFibers() {}

  return reconcileChildFibers;
}
```

`ChildReconciler` 是一个工厂函数。

**参数**：

- `shouldTrackSideEffects`：由是否首次挂载确定，`false` 表示首次挂载，不需要打删除（`Deletion`）标记。

**实现**：

其中包含了各个类型节点的 `reconcile` 处理函数以及 `reconcile` 过程中使用到的工具函数。这些函数中会根据 `shouldTrackSideEffects` 做区分处理。

最终返回 `reconcileChildFibers` 函数，它其实就是 `mountChildFibers` 和 `reconcileChildFibers` 函数。其中会调用各类型节点的处理函数和工具函数。

## reconcileChildFibers

```ts
function reconcileChildFibers(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
  lanes: Lanes
): Fiber | null {
  if (typeof newChild === "object" && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return placeSingleChild(
          reconcileSingleElement(
            returnFiber,
            currentFirstChild,
            newChild,
            lanes
          )
        );
      case REACT_PORTAL_TYPE:
        return placeSingleChild(
          reconcileSinglePortal(returnFiber, currentFirstChild, newChild, lanes)
        );
      case REACT_LAZY_TYPE:
        if (enableLazyElements) {
          const payload = newChild._payload;
          const init = newChild._init;
          return reconcileChildFibers(
            returnFiber,
            currentFirstChild,
            init(payload),
            lanes
          );
        }
    }

    if (isArray(newChild)) {
      return reconcileChildrenArray(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes
      );
    }

    if (getIteratorFn(newChild)) {
      return reconcileChildrenIterator(
        returnFiber,
        currentFirstChild,
        newChild,
        lanes
      );
    }

    throwOnInvalidObjectType(returnFiber, newChild);
  }

  if (
    (typeof newChild === "string" && newChild !== "") ||
    typeof newChild === "number"
  ) {
    return placeSingleChild(
      reconcileSingleTextNode(
        returnFiber,
        currentFirstChild,
        "" + newChild,
        lanes
      )
    );
  }

  return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

**参数**：

- `returnFiber`：当前正在构建的 Fiber 节点。
- `currentFirstChild`：当前正在构建的 Fiber 节点对应的旧节点的第一个子节点（链表头部），首次挂载为 `null`，非首次通过 `current.child` 获取。
- `newChild`：当前 Fiber 节点子节点的虚拟 DOM 对象。
- `lanes`：本次渲染优先级。

**实现**：

1. `newChild` 为标准 ReactElement 对象，根据 `newChild.$$typeof` 执行不同类型的 `reconcile`。

> `newChild.$$typeof` 是 React 内部用来标识不同 **React 元素类型**的一个唯一标识，它是一个 Symbol。

常见的 `$$typeof` 值有：

```ts
export const REACT_ELEMENT_TYPE = Symbol.for("react.element");
export const REACT_PORTAL_TYPE = Symbol.for("react.portal");
export const REACT_FRAGMENT_TYPE = Symbol.for("react.fragment");
export const REACT_STRICT_MODE_TYPE = Symbol.for("react.strict_mode");
export const REACT_PROFILER_TYPE = Symbol.for("react.profiler");
export const REACT_SUSPENSE_TYPE = Symbol.for("react.suspense");
export const REACT_LAZY_TYPE = Symbol.for("react.lazy");
```

2. `newChild` 为文本节点，执行文本类型的 `reconcile`。

3. 二者都不是，则 `deleteRemainingChildren` 删除无效的旧 children。

以上操作，都会返回新的子 Fiber 节点。

# 不同类型节点的 reconcile

根据 `newChild` 的类型分别处理，下面看几个最常见的 `reconcile`。

## reconcileSingleElement

处理单个 ReactElement 与旧 Fiber 进行 diff 的逻辑。

```ts
function reconcileSingleElement(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  element: ReactElement,
  lanes: Lanes
): Fiber {
  const key = element.key;
  let child = currentFirstChild;
  while (child !== null) {
    if (child.key === key) {
      const elementType = element.type;
      if (elementType === REACT_FRAGMENT_TYPE) {
        if (child.tag === Fragment) {
          deleteRemainingChildren(returnFiber, child.sibling);
          const existing = useFiber(child, element.props.children);
          existing.return = returnFiber;
          return existing;
        }
      } else {
        if (child.elementType === elementType) {
          deleteRemainingChildren(returnFiber, child.sibling);
          const existing = useFiber(child, element.props);
          existing.ref = coerceRef(returnFiber, child, element);
          existing.return = returnFiber;
          return existing;
        }
      }
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  if (element.type === REACT_FRAGMENT_TYPE) {
    const created = createFiberFromFragment(
      element.props.children,
      returnFiber.mode,
      lanes,
      element.key
    );
    created.return = returnFiber;
    return created;
  } else {
    const created = createFiberFromElement(element, returnFiber.mode, lanes);
    created.ref = coerceRef(returnFiber, currentFirstChild, element);
    created.return = returnFiber;
    return created;
  }
}
```

**参数**：

- `returnFiber`：当前正在构建的 Fiber 节点。
- `currentFirstChild`：当前正在构建的 Fiber 节点对应的旧节点的第一个子节点（链表头部），首次挂载为 `null`，非首次通过 `current.child` 获取。
- `element`：当前 Fiber 节点子节点的虚拟 DOM 对象。
- `lanes`：本次渲染优先级。

**实现**：

### 遍历寻找可复用

从旧的第一个子节点开始依次取 `sibling` 遍历旧的子 Fiber ，从中寻找可复用节点。

检查遍历到的旧 `fiber.key` 和新 ReactElement 的 `key` 是否相同。

#### key 匹配

`key` 匹配，再检查 `type` 是否匹配，即旧 `fiber.elementType` 和新 ReactElement 的 `type` 是否相同，其中 `Fragment` 为特殊 case，用 `tag` 判断。

##### type 匹配

`key` 和 `type` 都匹配：

1. `deleteRemainingChildren` 将当前遍历到的旧 Fiber **之后的**兄弟节点都打上删除标记。因为当前匹配成功，后面不需要了。

2. `useFiber` 复用这个旧 Fiber，避免重新分配内存。

3. 通过 `existing.return = returnFiber` 建立复用后节点的父子关系。

4. 直接返回这个复用的旧节点 `existing` 作为新的 `workInProgress.child`。

##### type 不匹配

`key` 匹配，`type` 不匹配：

1. `deleteRemainingChildren` 将当前遍历的旧 Fiber **以及之后的**兄弟节点打上删除标记。因为有过 `key` 匹配的，后续 `key` 也不会匹配了，一定不能复用，直接删除所有旧 Fiber。

2. `break` 跳出循环：已经确定后续旧节点都不能复用了，则不再遍历了。

#### key 不匹配

`key` 不匹配，`deleteChild` 将当前遍历到的这个旧 Fiber 打上删除标记，继续遍历后面的兄弟节点。

### 没找到可复用则创建新 Fiber

所有 `key` 都不匹配，以及 `key` 匹配但 `type` 不匹配的情况，没有可复用的旧 Fiber，则创建新 Fiber：

1. 根据是否为 `Fragment` 调用不同创建函数，普通节点调用 `createFiberFromElement`。

2. 得到新 Fiber 后建立 `return` 父子关系。

3. 返回新创建的 Fiber 作为 `workInProgress.child`。

> 过程中用到的工具函数 `deleteRemainingChildren` 、`deleteChild` 、`useFiber` 、`createFiberFromElement` 的实现在本文最后一节中有解析，可以跳过去看。

### placeSingleChild

`reconcileSingleElement` 返回复用或新建的子节点，调用 `placeSingleChild` 判断是否为新建节点（`alternate` 属性为 `null` 则为新建），新节点则打上 `Placement` 标记，供 commit 阶段插入 DOM 用。

```ts
function placeSingleChild(newFiber: Fiber): Fiber {
  if (shouldTrackSideEffects && newFiber.alternate === null) {
    newFiber.flags |= Placement;
  }
  return newFiber;
}
```

> React 在 render 阶段仅构建 Fiber 树，并为节点打上一些副作用标记，比如 `Placement` 插入、`Deletion` 删除、`Update` 更新，等 commit 阶段再去做真正的 DOM 操作。
>
> 其中 `Placement` 和 `Deletion` 标记在 `reconcile` 过程中标记，`Update` 在 `beginWork` 结束后 `commitUnitOfWork` -> `completeWork` 中标记，可以看[这里](url)，`Update` 标记的场景包括属性变更、文本变更等。

## reconcileSingleTextNode

处理单个文本节点与旧 Fiber 进行 diff 的逻辑。

```ts
function reconcileSingleTextNode(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  textContent: string,
  lanes: Lanes
): Fiber {
  if (currentFirstChild !== null && currentFirstChild.tag === HostText) {
    deleteRemainingChildren(returnFiber, currentFirstChild.sibling);
    const existing = useFiber(currentFirstChild, textContent);
    existing.return = returnFiber;
    return existing;
  }
  deleteRemainingChildren(returnFiber, currentFirstChild);
  const created = createFiberFromText(textContent, returnFiber.mode, lanes);
  created.return = returnFiber;
  return created;
}
```

**参数**：

- `returnFiber`：当前正在构建的 Fiber 节点。
- `currentFirstChild`：当前正在构建的 Fiber 节点对应的旧节点的第一个子节点（链表头部），首次挂载为 `null`，非首次通过 `current.child` 获取。
- `textContent`：当前 Fiber 节点的纯文本子节点，`string` 或 `number` 类型。
- `lanes`：本次渲染优先级。

**实现**：

1. 如果旧 Fiber 节点的**第一个**子节点也是纯文本节点，则复用。

- `deleteRemainingChildren` 将旧节点**第一个子 Fiber 之后的**兄弟节点都打上删除标记。

- `useFiber` 复用第一个文本子节点。

- 通过 `existing.return = returnFiber` 建立复用后节点的父子关系。

- 直接返回这个复用的旧节点 `existing` 作为新的 `workInProgress.child`。

2. 如果旧 Fiber 节点不存在或它的第一个子节点不是纯文本节点，则无法复用。

- `deleteRemainingChildren` 将旧节点的**所有子节点**打上删除标记。

- 调用 `createFiberFromText` 创建新的 `HostText` 文本节点。

- 得到新 Fiber 后建立 `return` 父子关系。

- 返回新创建的 Fiber 作为 `workInProgress.child`。

> 为什么文本节点的 diff 中没有 ReactElement 节点那样的 key 检查？
>
> 因为 React 不支持给文本节点直接加 key，所以 diff 时可以跳过 key 比较。

## reconcileChildrenArray

处理子节点数组（ReactElement[]）与旧 FiberList 进行 diff 的逻辑。

```ts
function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<*>,
  lanes: Lanes
): Fiber | null {
  let resultingFirstChild: Fiber | null = null;
  let previousNewFiber: Fiber | null = null;

  let oldFiber = currentFirstChild;
  let lastPlacedIndex = 0;
  let newIdx = 0;
  let nextOldFiber = null;
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (oldFiber.index > newIdx) {
      nextOldFiber = oldFiber;
      oldFiber = null;
    } else {
      nextOldFiber = oldFiber.sibling;
    }
    const newFiber = updateSlot(
      returnFiber,
      oldFiber,
      newChildren[newIdx],
      lanes
    );
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
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }

  if (newIdx === newChildren.length) {
    deleteRemainingChildren(returnFiber, oldFiber);
    if (getIsHydrating()) {
      const numberOfForks = newIdx;
      pushTreeFork(returnFiber, numberOfForks);
    }
    return resultingFirstChild;
  }

  if (oldFiber === null) {
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
      if (newFiber === null) {
        continue;
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
    return resultingFirstChild;
  }

  const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = updateFromMap(
      existingChildren,
      returnFiber,
      newIdx,
      newChildren[newIdx],
      lanes
    );
    if (newFiber !== null) {
      if (shouldTrackSideEffects) {
        if (newFiber.alternate !== null) {
          existingChildren.delete(
            newFiber.key === null ? newIdx : newFiber.key
          );
        }
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
  }

  if (shouldTrackSideEffects) {
    existingChildren.forEach((child) => deleteChild(returnFiber, child));
  }

  return resultingFirstChild;
}
```

**参数**：

- `returnFiber`：当前正在构建的 Fiber 节点。
- `currentFirstChild`：当前正在构建的 Fiber 节点对应的旧节点的第一个子节点（链表头部），首次挂载为 `null`，非首次通过 `current.child` 获取。
- `newChildren`：当前 Fiber 节点的子节点数组。
- `lanes`：本次渲染优先级。

**实现**：

分为 4 个阶段。

### 前向逐个对比

1. 循环取相同 `index` 的新旧节点：

- 新的 `newChildren` 是数组，通过 `index` 下标获取新节点。

- 旧的为链表结构，通过 `sibling` 向后获取旧节点。

2. 尝试复用：通过 `updateSlot` 进行 `key` + `type` 的匹配判断（实现类似 `reconcileSingleElement` 中），都匹配上则通过 `useFiber` 复用并返回复用后的新节点，否则返回 `null`。

3. 相同 `index` 的新旧节点一旦出现不能复用的情况，立刻 `break` 跳出前向循环。

4. 相同 `index` 的新旧节点如果 `key` + `type` 相同尝试复用了，检查是否成功复用。如果 `alternate === null` 表示复用失败，直接 `deleteChild` 删除旧节点。

5. 调用 `placeChild` 计算是否需要 `Placement`：

- 复用失败的新节点直接打上 `Placement` 标记，表示需要在 commit 阶段插入 DOM。

- 复用成功的新节点根据复用的旧节点的 `index` 位置决定是否要打上 `Placement` 标记，仅涉及**相对位置的变化**才打标，表示需要在 commit 阶段移动 DOM。

- 返回 `lastPlacedIndex` ，表示已被成功复用并放置的旧节点中最靠后的位置索引，**用于判断相对位置是否移动**。

> `placeChild` 的具体实现可以跳至「工具函数」一节查看。

### 仅剩余旧节点

`newIdx === newChildren.length` 表示新节点已复用/新建完毕：

1. 调用 `deleteRemainingChildren` 删除剩余旧节点。

2. 返回新的子 Fiber 链表头。

### 仅剩余新节点

`oldFiber === null` 表示旧节点已用完：

1. 遍历剩余新节点，调用 `createChild` 从 ReactElement 依次创建 Fiber。

2. 调用 `placeChild` 为新创建的 Fiber 节点打上 `Placement` 标记，表示需要在 commit 阶段插入 DOM。

3. 返回新的子 Fiber 链表头。

### 新旧节点均有剩余

1. 对剩余旧节点调用 `mapRemainingChildren` 构建 `key -> oldFiber` 的映射表。

> `mapRemainingChildren` 的具体实现可以跳至「工具函数」一节查看。

2. 遍历剩余新节点，调用 `updateFromMap` 从上一步构建的映射表中查找：

- 找不到就创建新 Fiber 并 `placeChild` 标记 `Placement`，表示需要在 commit 阶段插入 DOM。

- 找到就复用，并删除映射表中被复用的旧节点，然后 `placeChild` 根据复用的旧节点的 `index` 位置决定是否要打上 `Placement` 标记，在 commit 阶段移动 DOM。

3. 新节点遍历完成后，循环调用 `deleteChild` 删除映射表中剩余的旧节点。

4. 返回新的子 Fiber 链表头。

> **React 中子节点数组的 diff ，为了提升性能，使用了一些性能优化策略**：
>
> 1. 快路径 + 慢路径的结合
>
> 先乐观估计相同 index 的节点匹配，按照 index 对比，只有当位置不匹配的时候才 fallback 到使用映射表寻找。
>
> 2. 尽量复用旧节点，减少新创建
>
> 3. 复用旧节点的情况下，尽量减少 DOM 的移动
>
> `placeChild` 中通过 `lastPlacedIndex` 判断仅相对位置变化才标记 `Placement`。

# 工具函数

## deleteChild

为单个 Fiber 节点打上删除标记，在 commit 阶段统一处理。

```ts
function deleteChild(returnFiber: Fiber, childToDelete: Fiber): void {
  if (!shouldTrackSideEffects) {
    return;
  }
  const deletions = returnFiber.deletions;
  if (deletions === null) {
    returnFiber.deletions = [childToDelete];
    returnFiber.flags |= ChildDeletion;
  } else {
    deletions.push(childToDelete);
  }
}
```

1. `shouldTrackSideEffects` 为 `false` 表示首次挂载，不需要打删除标记，直接返回。

2. 将要删除的节点放到父节点 `deletions` 数组中，然后给父节点打上 `ChildDeletion` 副作用标记，后续 commit 阶段处理。

## deleteRemainingChildren

循环为某个节点以及它之后的兄弟节点都打上删除标记，在 commit 阶段统一处理。

```ts
function deleteRemainingChildren(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null
): null {
  if (!shouldTrackSideEffects) {
    return null;
  }

  let childToDelete = currentFirstChild;
  while (childToDelete !== null) {
    deleteChild(returnFiber, childToDelete);
    childToDelete = childToDelete.sibling;
  }
  return null;
}
```

1. `shouldTrackSideEffects` 为 `false` 表示首次挂载，不需要打删除标记，直接返回。

2. 从传入的 `currentFirstChild` 节点开始，对于它和它之后的兄弟节点，循环调用 `deleteChild`，放入父节点的 `deletions` 中，且给父节点打上 `ChildDeletion` 副作用标记，后续 commit 阶段处理。

## useFiber

调用 [`createWorkInProgress`](url) 复用原有节点，仅清空副作用标记。

```ts
function useFiber(fiber: Fiber, pendingProps: mixed): Fiber {
  const clone = createWorkInProgress(fiber, pendingProps);
  clone.index = 0;
  clone.sibling = null;
  return clone;
}
```

## createFiberFromElement

根据一个 ReactElement 创建对应的 Fiber 节点。

```ts
export function createFiberFromElement(
  element: ReactElement,
  mode: TypeOfMode,
  lanes: Lanes
): Fiber {
  let owner = null;
  const type = element.type;
  const key = element.key;
  const pendingProps = element.props;
  const fiber = createFiberFromTypeAndProps(
    type,
    key,
    pendingProps,
    owner,
    mode,
    lanes
  );
  return fiber;
}
```

**1. 获取 ReactElement 基础信息**

```ts
{
  $$typeof: Symbol(react.element),
  type: 'div' | FunctionComponent | Fragment | Lazy | Suspense 等,
  key: 'xxx' | null,
  ref: ...,
  props: {...}
}
```

**2. 调用 `createFiberFromTypeAndProps`**

```ts
function createFiberFromTypeAndProps(type, key, props, owner, mode, lanes) {
  let fiberTag = IndeterminateComponent;

  if (typeof type === "function") {
    // 函数组件或类组件
    fiberTag = shouldConstruct(type) ? ClassComponent : FunctionComponent;
  } else if (typeof type === "string") {
    // DOM 元素
    fiberTag = HostComponent;
  } else if (type.$$typeof === REACT_FORWARD_REF_TYPE) {
    fiberTag = ForwardRef;
  } else if (type.$$typeof === REACT_MEMO_TYPE) {
    fiberTag = MemoComponent;
  } else if (type === REACT_FRAGMENT_TYPE) {
    fiberTag = Fragment;
  } else {
    // fallback
  }

  const fiber = createFiber(fiberTag, props, key, mode);
  fiber.elementType = type;
  fiber.type = type;
  fiber.lanes = lanes;
  return fiber;
}
```

根据 ReactElement 的 `type` 决定创建什么类型的 Fiber：

- `type` 为 `string` 类型，比如 `"div"`：创建 `HostComponent`。

- `type` 为 `function` 类型：根据是否有构造函数，创建 `FunctionComponent` 或 `ClassComponent`。

- `type` 为 `REACT_FRAGMENT_TYPE`：创建 `Fragment`。

...

然后调用 `createFiber` 真正创建 Fiber 实例：

```ts
const createFiber = function (
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode
): Fiber {
  return new FiberNode(tag, pendingProps, key, mode);
};
```

## placeChild

用于在数组 diff 过程中判断新的子节点是否需要打上 `Placement` 标记，即需要在 commit 阶段插入或移动 DOM。

```ts
function placeChild(
  newFiber: Fiber,
  lastPlacedIndex: number,
  newIndex: number
): number {
  newFiber.index = newIndex;
  const current = newFiber.alternate;
  if (current !== null) {
    const oldIndex = current.index;
    if (oldIndex < lastPlacedIndex) {
      newFiber.flags |= Placement;
      return lastPlacedIndex;
    } else {
      return oldIndex;
    }
  } else {
    newFiber.flags |= Placement;
    return lastPlacedIndex;
  }
}
```

**参数**：

- `newFiber`：当前正在处理的新节点。

- `lastPlacedIndex`：表示已被成功复用并放置的旧节点中最靠后的位置索引。

- `newIndex`：当前新节点的下标。

**实现**：

1. 将新节点的下标放到新的 `newFiber.index` 上。

2. `current !== null` 复用了旧节点，根据旧节点的 `index` 判断：

- 复用的旧节点的 `index` 小于 `lastPlacedIndex`，表示相对于已处理的节点，顺序乱了，要移动到前面，则标记 `Placement`，并原样返回 `lastPlacedIndex`。

- 复用的旧节点的 `index` 大于等于 `lastPlacedIndex`，表示相对于已处理的节点排在后面，无需移动，则不标记 `Placement`，并返回 `index` 即目前最靠后的索引。

3. 未复用旧节点，直接标记 `Placement`，表示需插入。

## mapRemainingChildren

构建 `key -> oldFiber` 的映射表。

```ts
function mapRemainingChildren(
  returnFiber: Fiber,
  currentFirstChild: Fiber
): Map<string | number, Fiber> {
  const existingChildren: Map<string | number, Fiber> = new Map();

  let existingChild = currentFirstChild;
  while (existingChild !== null) {
    if (existingChild.key !== null) {
      existingChildren.set(existingChild.key, existingChild);
    } else {
      existingChildren.set(existingChild.index, existingChild);
    }
    existingChild = existingChild.sibling;
  }
  return existingChildren;
}
```

1. 从旧节点链表头开始，通过取 `sibling` 遍历旧节点链表。

2. 如果 Fiber 上有 key 就取 key，没有就取 index 作为 Map key，生成 Map 映射表。大概结构如下：

```ts
Map {
  'a' => <li key="a">,
  'b' => <li key="b">,
   2  => <li />, // 无 key 的用 index 代替
}
```

## updateFromMap

从 map 中查找并更新

根据 key 查 Map，看是否复用；

找不到就 createFiberFromElement;

找到但 alternate 是 null → 插入；

找到且 alternate 存在 → 复用；

遍历完成后，还没被用到的旧节点视为被删除，统一调用 deleteChild。

# 总结

本文介绍了 React 源码 render 阶段中的 `reconcile` 部分。展开讲解了不同类型节点如何通过 `reconcile` 比较当前 Fiber 与新生成的 ReactElement 之间的差异，生成新的标记了副作用的 Fiber 树。其实也就是 React 中的虚拟 DOM diff 算法。

下一章我们将进入 commit 阶段，了解如何使用 render 阶段标记的这些副作用（比如 `Placement`、`Deletion`、`Update`），去操作真实 DOM。
