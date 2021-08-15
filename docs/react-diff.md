React Diff 策略
====

React 的核心思想：内存中维护一颗虚拟 DOM 树，数据变化时（`setState`），自动更新虚拟 DOM 得到一颗新树，然后 Diff 新老虚拟 DOM 树，找到有变化的部分，得到每个补丁（`Patch`），将这些 Patch 加入队列，最终批量更新这些 Patch 到 DOM 中。

组件更新归根到底是 DOM 的更新，这部分主要分两个部分来完成：

- 调和阶段，虚拟 DOM 的 diff 算法。
- 提交阶段，也就是将上一个阶段中 diff 出来的内容体现到 DOM。

整个更新过程（不包括渲染）就是在反复寻找工作单元并运行它们：

``` js
// while 循环只有当找不到工作单元或者应该打断的时候才会终止。找不到工作单元的情况只有当循环完所有工作单元才会触发，打断的情况是调度器触发的
while (nextUnitOfWork !== null && !shouldYield()) {
  // performUnitOfWork：寻找下一个工作单元
  nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
}
```

### 如何寻找工作单元？

每个 `fiberNode` 都有 `return`、`child`、`sibling` 这三个属性，`return` 指向父节点，`child` 指向它的第一个子节点，即便该节点有好多个子节点。`sibling` 属性指向着下一个兄弟节点。通过它们构成一个 fiber 树的数据结构，fiber 树其实是一个单链表树结构。每个 `fiberNode` 就是一个工作单元，循环寻找工作单元是自顶向下再向上的一个单链表循环。其循环规则如下：

1. `root` 永远是第一个工作单元，不管之前有没有被打断过任务。
2. 首先判断当前节点是否存在第一个子节点，存在的话它就是下一个工作单元，并让下一个工作节点继续执行该条规则，不存在的话就跳到规则 3。
3. 判断当前节点是否存在兄弟节点。如果存在兄弟节点，就回到规则 2，否则跳到规则 4；
4. 回到父节点并判断父节点是否存在。如果存在则执行规则 3，否则跳到规则 5；
5. 当前工作单元为 `null`，即为完成整个循环。

![图例](https://bobi.ink/images/react-fiber/fiber-node.png)

### 每个工作单元是如何工作的？

一个工作单元可以是dom和组件，同时组件也分多种类型。不同的工作单元就存在不同的处理分支。以 class 组件为例，**调和过程**分成处理生命周期函数与调和子组件（diff 算法的过程）两个过程。

### 生命周期函数的处理？

1. 调用 `componentWillReceiveProps` ：注册了 `getDerivedStateFromProps` 或者 `getSnapshotBeforeUpdate`，或者 `props` 前后无差别就不会调用此函数。
2. 调用 `getDerivedStateFromProps` 设置最新的 `state`。
3. 判断组件是否更新，判断 `shouldComponentUpdate` 是否存在：
  - 存在则调用 `shouldComponentUpdate`；
  - 不存在则判断组件是否继承自 `PureComponent`，是则浅比较前后的 `props` 及 `state`。
4. 调用 `componentWillUpdate`。
5. 处理 `componentDidUpdate`。
6. 处理 `getSnapshotBeforeUpdate`。

调和阶段并不会调用以上两个函数，而是在 `effectTag` 打上 tag，以便将来使用位运算知晓是否需要使用它们：

``` js
if (typeof instance.componentDidUpdate === 'function') {
  workInProgress.effectTag |= Update;
}
if (typeof instance.getSnapshotBeforeUpdate === 'function') {
  workInProgress.effectTag |= Snapshot;
}
```

### 子组件 diff 算法的过程？

diff 的主要目的是为了复用。 当 diff 过程中出现需要在渲染阶段进行处理的节点时，会把这些节点放入父节点的 effect 链表中，比如需要被删除的节点就会把加入进链表。这个链表的作用是可以帮助我们在渲染阶段迅速找到需要更新的节点。创建 WorkInProgress Tree 的过程也是一个 diff 的过程，diff 完成之后会生成一个 Effect List，这个 Effect List 就是最终 Commit 阶段用来处理副作用的阶段。

组件 `render` 方法返回 newChild，用于与 oldChild 进行对比。

- `returnFiber`：父组件。
- `currentFirstChild`：父组件的第一个 child，每个 fiber 都有一个 `sibling` 属性指向下一个兄弟节点。
- `newChild`：`render` 返回的内容。

不同类型的 `newChild` 都会有不同的 diff 策略，`Fragment` 类型、`object`、`number` 或者 `string` 类型的处理都很简单。可迭代类型（`Array` 或者 `Iterator` 类型）相对复杂，它们核心逻辑是一致的，以 `Array` 类型为例：

1. 第一轮遍历：相同位置（`index`）进行比较，一旦出现不能复用的情况就跳出遍历。

节点复用的规则：
- 新旧节点都为文本节点，可以直接复用，因为文本节点不需要 `key`。
- 其他类型节点一律通过判断 `key` 是否相同来复用或创建节点（可能类型不同但 `key` 相同）。

``` js
// 简化代码
// newChildren 是一个数组结果
// oldFiber 是一个链表接口，通过 sibling 查下一个 nextOldFiber
for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
  // 找到下一个老的子节点，nextOldFiber 为 null 时 oldFiber 已经遍历完
  nextOldFiber = oldFiber.sibling;
  // 通过 oldFiber 和 newChildren[newIdx] 判断是否可以复用
  // 并给复用出来的节点的 return 属性赋值 returnFiber
  const newFiber = reuse(
    returnFiber,
    oldFiber,
    newChildren[newIdx]
  );
  // 不能复用，跳出
  if (newFiber === null) {
    break;
  }
}
```

2. 第二轮遍历。可能出现两种情况：

- 如果 `newChild` 已经遍历完，把剩下的老节点删除。

``` js
if (newIdx === newChildren.length) {
  // 新的 children 长度已经够了，所以把剩下的删除掉
  deleteRemainingChildren(returnFiber, oldFiber);
  return resultingFirstChild;
}

```

- 若老的节点已经遍历完，只需要把剩余新的节点全部创建新的 `Fiber`。

``` js
if (oldFiber === null) {
  // 如果老的节点已经被复用完了，对剩下的新节点进行操作
  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = createChild(
      returnFiber,
      newChildren[newIdx],
      expirationTime,
    );
  }
  return resultingFirstChild;
}

```

3. 第三轮遍历。判断节点是否发生过移动操作，是则复用节点，否则创建 `fiber`。

先所有老数组元素按 `key` 或者是 `index` 放 Map 里：

``` js
function mapRemainingChildren(
 returnFiber: Fiber,
 currentFirstChild: Fiber,
): Map<string | number, Fiber> {
  const existingChildren: Map<string | number, Fiber> = new Map();

  let existingChild = currentFirstChild; // currentFirstChild 是老数组链表的第一个元素
  while (existingChild !== null) {
  // 看到这里可能会疑惑怎么在 Map 里面的key 是 fiber 的key 还是 fiber 的 index 呢？
  // 我觉得是根据数据类型，fiber 的key 是字符串，而 index 是数字，这样就能区分了
  // 所以这里是用的 map，而不是对象，如果是对象的key 就不能区分 字符串类型和数字类型了。
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

遍历新节点，如果在 Map 里面存在复用的节点就复用，否则就创建 `newFiber`。

``` js
// 如果前面的算法有复用，那么 newIdx 就不从 0 开始
for (; newIdx < newChildren.length; newIdx++) {
  const newFiber = updateFromMap(
    existingChildren,
    returnFiber,
    newIdx,
    newChildren[newIdx],
    expirationTime,
  );
 // 省略删除 existingChildren 中的元素和添加 Placement 副作用的情况
}

```

完成以上步骤后，同层的 diff 过程完毕。

总结：

第一遍遍历：相同位置（`index`）的新、老节点进行对比，相同则通过 `updateSlot` 方法找到可以复用的节点，直到找到不可以复用的节点就退出循环。
第二遍遍历：新节点（`newChildren`）先遍历完则删除剩余的老节点，老节点链表（`oldFiber`）先遍历完则创建剩余的新节点的。
第三遍遍历：把所有老节点按 `key` 或 `index` 放 Map 里，遍历新数组（`newChildren`），存在可复用节点则复用，否则创建新的 `fiber`。

## 参考

- [组件更新流程二](https://yuchengkai.cn/react/2019-08-05.html#%E5%A4%84%E7%90%86-class-%E7%BB%84%E4%BB%B6%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E5%87%BD%E6%95%B0)
- [Deep In React 之详谈 React 16 Diff 策略](https://juejin.im/post/5d3e3231e51d4510926a7c39)
