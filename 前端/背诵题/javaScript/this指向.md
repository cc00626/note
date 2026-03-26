### 1. 默认绑定（最低优先级）

当一个函数作为独立函数调用时（不带任何修饰），`this` 指向全局对象。

- **非严格模式**：指向 `window` (浏览器) 或 `global` (Node.js)。
- **严格模式 (`"use strict"`)**：指向 `undefined`。

### 2. 隐式绑定

当函数作为某个对象的方法被调用时，`this` 指向该**直接对象**。

- **注意“隐式丢失”**：如果把对象的方法赋值给一个变量再调用，`this` 会丢失并指向全局。

```javascript
const obj = {
  name: 'Gemini',
  sayName() { console.log(this.name); }
};
obj.sayName(); // this 指向 obj

const alias = obj.sayName;
alias(); // this 指向 window (隐式丢失)
```

### 3. 显式绑定 (`call`, `apply`, `bind`)

通过这三个方法，我们可以硬性改变 `this` 的指向。

- `call/apply` 是立即调用。
- `bind` 是返回一个新函数。

------

### 4. `new` 绑定

使用 `new` 关键字调用构造函数时，JS 引擎会：

1. 创建一个空对象。
2. **将这个空对象的 `this` 绑定到构造函数上。**
3. 执行构造函数。
4. 返回这个新对象。

### 5. 箭头函数（特殊存在）

箭头函数**没有自己的 `this`**。

- 它会捕获**定义时**所处的外部（父级）词法环境的 `this`。
- **重点**：一旦确定，`call/apply/bind` 都无法改变箭头函数的 `this` 指向。

------

###  优先级总结（从高到低）

1. **`new` 绑定**（优先级最高）
2. **显式绑定** (`call/apply/bind`)
3. **隐式绑定** (对象调用)
4. **默认绑定** (直接调用，优先级最低)

------

### 面试常见“坑”点（加分项）

1. **DOM 事件绑定**：
   - 在 `addEventListener` 的回调中，`this` 指向**绑定事件的元素**（`event.currentTarget`）。
2. **定时器 `setTimeout`**：
   - 普通的 `setTimeout` 回调函数，其 `this` 默认指向 `window`（因为它是异步全局调用的）。
3. **类 (Class) 中的方法**：
   - 类的方法默认开启“严格模式”，如果直接解构出来调用，`this` 会是 `undefined`。

