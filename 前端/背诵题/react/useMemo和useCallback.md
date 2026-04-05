### 为什么使用

在react中每次组件重新进行渲染，内部的普通变量和函数都会重新创建，分配新的内存地址。

```
// 父组件
const handleClick = () => { console.log('clicked') }; 
return <Child onClick={handleClick} />;
```



在子组件使用了react.mome的时候，每次handleChilck的引用都会变，这样子组件就是强制重新渲染，这时候就需要使用useCallBack锁定函数引用。



useMemo为了保持引用不变和复杂昂贵的计算。 配合 `useEffect` 避免死循环。如果你的 `useEffect` 依赖于一个对象或数组，必须确保这个对象在没有逻辑变化时引用是稳定的，否则 `useEffect` 会在每次渲染后重复触发。



### 区别：useCallback就是useMemo的语法糖

在 React 源码中，两者的逻辑高度统一。你可以把 `useCallback` 看作是 `useMemo` 的一个**特例**：

- `useMemo(() => value, deps)` -> 缓存值。
- `useCallback(fn, deps)` -> 缓存函数。
- **本质等价：** `useCallback(fn, deps)` 内部其实就是在做 `useMemo(() => fn, deps)`。

### React.memo

默认情况下，当父组件重新渲染时，React 会**递归地渲染其所有的子组件**，无论子组件的 Props 是否发生了变化。

`React.memo` 通过包裹子组件，让 React 在渲染前先进行一次 **Props 比对**：

- **如果 Props 没变**：直接跳过该组件的渲染，复用上一次的渲染结果（Memoized Result）。
- **如果 Props 变了**：执行正常的渲染流程。

```
const MyComponent = React.memo(function MyComponent(props) {
  /* 使用 props 渲染 */
});
```



`React.memo` 默认执行的是**浅比较**。这意味着它只检查 Props 对象的第一层属性：

- **基本类型**（String, Number, Boolean）：比较**值**是否相等。
- **引用类型**（Object, Array, Function）：比较**内存地址（引用）**是否相同。

> **注意**：这正是为什么你之前问 `useCallback` 的原因。如果你给 `React.memo` 包裹的子组件传了一个普通函数，父组件一变，函数地址就变，`React.memo` 的浅比较就会判定“Props 已改变”，导致缓存失效。



**自定义比较逻辑**

`React.memo` 接受第二个可选参数，允许你手动控制比较逻辑。这在处理复杂的深层对象时非常有用。

```
const areEqual = (prevProps, nextProps) => {
  // 返回 true 则【跳过】重新渲染
  // 返回 false 则【触发】重新渲染
  return prevProps.id === nextProps.id && prevProps.data.version === nextProps.data.version;
};

export default React.memo(MyChildComponent, areEqual);
```



### useCallback的原理

**1. 核心矛盾：函数组件的“自杀式”更新**

在 JavaScript 中，每次执行函数都会创建一个新的内存地址。

- **现象：** 每次父组件重新渲染（Re-render），内部定义的函数都会被“重新创建”。
- **后果：** 即使函数逻辑没变，它的**引用地址**变了。如果这个函数作为 Props 传给子组件，会导致被 `React.memo` 包裹的子组件误以为数据变了，从而引发无意义的重复渲染。

------

**2. 内部原理：闭包与 Fiber 存储**

`useCallback` 的本质是利用 **Memoization（记忆化）** 技术。

**挂载阶段 (Mount)**

当组件首次渲染时，React 执行 `useCallback(fn, deps)`：

1. 它将你传入的函数 `fn` 和依赖项数组 `deps` 直接存储在当前 Fiber 节点的 `memoizedState` 中。
2. 返回你传入的这个 `fn`。

**更新阶段 (Update)**

当组件重新渲染时，React 会对比新旧依赖项：

1. **对比依赖项：** 使用 `Object.is` 算法逐一比对 `deps` 数组。
2. **命中缓存：** 如果依赖项**完全一致**，React 会直接从 Fiber 节点取出**上次存好的旧函数**返回，丢弃本次新创建的函数。
3. **失效更新：** 如果依赖项**发生了变化**，React 才会存储并返回本次新创建的函数。



------

### 4. 关键误区：并不是所有函数都要加 useCallback

这是新手最容易犯的错误。**滥用 `useCallback` 反而会变慢。**

- **成本：** 每次渲染，React 依然要创建新函数，还要额外创建一个数组（deps），并进行一次循环遍历对比。
- **结论：** 只有在以下两种场景下，`useCallback` 才有意义：
  1. **配合 `React.memo`：** 当函数作为 Props 传递给被 `memo` 的子组件时。
  2. **作为其他 Hook 的依赖：** 比如该函数被用在了 `useEffect` 的依赖项中，为了防止 Effect 频繁触发

### useMemo原理

**1. 核心原理：记忆化存储**

`useMemo` 的原理可以概括为：**“空间换时间”**。React 在内部开辟了一块空间，用来记录上一次计算的结果和当时的依赖项。

**挂载阶段 (Mount)**

当组件首次渲染时，React 执行 `useMemo(calculateValue, deps)`：

1. **执行函数：** 调用 `calculateValue()` 得到初始计算结果。
2. **存入 Fiber：** 将该**结果**和**依赖项数组**一起存入当前 Fiber 节点的 `memoizedState` 中。
3. **返回结果：** 将计算出的值返回给组件。

**更新阶段 (Update)**

当组件重新渲染时，React 会执行以下逻辑：

1. **取出旧数据：** 从 Fiber 节点取出上次存的 `{ value, deps }`。
2. **对比依赖项：** 使用 `Object.is` 逐一对比新的 `deps` 和旧的 `deps`。
3. **判断逻辑：**
   - **如果依赖项没变：** 直接返回上次存好的 `value`，**完全不执行**计算函数。
   - **如果依赖项变了：** 重新执行 `calculateValue()`，存入新的结果和依赖项，并返回新值。

