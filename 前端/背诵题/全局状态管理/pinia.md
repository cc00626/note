### 如何实现数据的变更的？

**Store (定义)**：通过 `defineStore` 创建，每个 Store 有唯一的 ID。

**State (数据)**：必须是一个**返回对象的函数**（避免服务端渲染污染）。

**Getters (计算属性)**：类似于 Vue 的 `computed`，具有缓存功能。

**Actions (方法)**：**同步、异步逻辑都写在这里**，直接通过 `this` 访问 state。

**Plugins (插件)**：扩展功能，比如实现“持久化存储”。



如何使用

```
// stores/counter.js
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  // 1. 状态：必须是箭头函数
  state: () => ({
    count: 0,
    name: 'Gemini'
  }),
  // 2. 计算属性：自动缓存
  getters: {
    doubleCount: (state) => state.count * 2,
  },
  // 3. 业务逻辑：同步异步通吃
  actions: {
    increment() {
      this.count++ // 直接用 this 修改，爽翻了
    },
    async fetchData() {
      const data = await api.get('/info')
      this.name = data.name
    }
  }
})
```

