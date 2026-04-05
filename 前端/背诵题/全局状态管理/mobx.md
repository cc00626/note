### 响应式数据

通过`makeObservable` 或者 `makeAutoObservable`创建响应式变量。

```
import { makeAutoObservable } from "mobx";

class WeatherStore {
  temp = 25;       // 自动推断为 observable
  city = "广州";    // 自动推断为 observable

  constructor() {
    // 关键一行：自动把这个类里的属性和方法变响应式
    makeAutoObservable(this);
  }

  // Getter 自动推断为 computed（带缓存）
  get description() {
    return `${this.city}当前温度：${this.temp}度`;
  }

  // 普通方法自动推断为 action (或 autoAction)
  updateTemp(newTemp) {
    this.temp = newTemp;
  }

  // Generator 函数自动推断为 flow (处理异步)
  *fetchWeather() {
    const data = yield fetch("/api/gz-weather").then(r => r.json());
    this.temp = data.temp;
  }
}

export const weatherStore = new WeatherStore();
```

### actions

### 1. 为什么一定要用 Action？（事务性 Transaction）

文档里提到了一个很关键的概念：**Transaction（事务）**。

**普通修改的问题：** 假设你要更新气象站的经度和纬度。如果你不用 action，每改一个值，依赖它的地图组件就会重绘一次。改两次，重绘两次。

**Action 的优势：**

JavaScript

```
increment() {
    this.value++ // 此时 UI 不更新
    this.value++ // 此时 UI 不更新
} // Action 执行完毕，UI 只更新一次
```

MobX 会开启一个“批处理”模式。在 `increment` 函数运行完之前，外界看不到中间状态。这不仅**提升了性能**，还防止了 UI 显示出“只有经度改了，纬度还没改”的错位状态。

------

### 2. `action.bound`：解决 `this` 指向黑洞

这是 JS 开发中经常踩的坑。

JavaScript

```
// ❌ 错误做法：直接传递方法，this 会丢失
setInterval(store.increment, 1000) 

// ✅ 正确做法 A：使用箭头函数（makeAutoObservable 默认支持）
// ✅ 正确做法 B：使用 action.bound
makeObservable(this, {
    increment: action.bound
})
```

使用了 `.bound` 后，无论这个函数被传到哪里（比如传给 `onClick` 或者 `setInterval`），它内部的 `this` 永远指向 `store` 实例，你再也不用写 `.bind(this)` 了。

------

### 3. `runInAction`：临时突击队

当你不想专门写一个类方法，只想在某个地方临时改一下数据时，用它。

**常见场景：** 在异步请求的回调里。因为异步代码（如 `fetch` 的 `.then`）执行时，原来的 action 已经结束了，所以你需要用 `runInAction` 把数据赋值语句包起来。

JavaScript

```
// 在组件或普通函数里
runInAction(() => {
    state.value = 100; // 这里的修改是受 MobX 保护和优化的
})
```

------

### 4. 异步 Action 的“终极方案”：`flow` {🚀}

这是文档中最推荐的部分，也是最难理解的部分。它解决了 `async/await` 必须不停写 `runInAction` 的麻烦。

**对比一下两种写法：**

#### **写法 A：`async/await`（麻烦版）**

JavaScript

```
async fetchProjects() {
    this.state = "pending"
    const data = await api.get()
    // 报错！await 之后不在 action 内部了，必须手动包一层
    runInAction(() => {
        this.projects = data
        this.state = "done"
    })
}
```

#### **写法 B：`flow`（优雅版）**

JavaScript

```
*fetchProjects() { // 注意星号 *
    this.state = "pending"
    try {
        const data = yield api.get() // yield 代替 await
        // 自动被 action 包裹，直接改，不报错！
        this.projects = data
        this.state = "done"
    } catch (e) {
        this.state = "error"
    }
}
```