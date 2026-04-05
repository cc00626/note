### 基本使用

```
//index.js

const worker = new Worker("./script/worker.js");

worker.onmessage = ({ data }) => {
  console.log("main recv:", data);
};

console.log("main: worker created");

worker.postMessage({ a: 1 });


//worker.js
self.onmessage = (e) => {
  console.log("worker recv:", e.data);
  self.postMessage({ reply: e.data.a + 1 });
};

// 如果想让worker一开始就发一次，也可以
self.postMessage({ ready: true });

```



### 注意

Worker 加载的脚本文件必须与主页面**同源**（协议、域名、端口必须完全一致）。

**跨域方案**：如果脚本在 CDN 上，通常需要通过 `Blob` 代理来绕过限制：

```
const workerCode = `importScripts('https://cdn.com/worker-script.js')`;
const blob = new Blob([workerCode], { type: 'text/javascript' });
const worker = new Worker(URL.createObjectURL(blob));
```





不能使用DOM和window对象，不能传递函数





**数据传输的“拷贝”开销**

默认情况下，主线程与 Worker 之间传输数据是**结构化克隆（Structured Clone）**。

- **原理**：数据在发送前被序列化，接收后被反序列化。

- **影响**：如果传输一个 500MB 的文件，内存会瞬间占用 1GB（原件 + 副本），且克隆过程会阻塞主线程。

- **终极优化：Transferable Objects** 通过转移所有权（而不是复制）实现零拷贝传输：

  JavaScript

  ```
  // 仅支持 ArrayBuffer, MessagePort 等特定类型
  const buffer = new ArrayBuffer(1024 * 1024);
  worker.postMessage(buffer, [buffer]); // 第二个参数指定转移对象
  // 此时主线程中的 buffer 已失效，不再占用主线程内存
  ```





**处理错误**

Worker 内部的报错不会自动弹到主线程的控制台，有时会表现为“无声的失败”。

- **双重保障**：

  1. 在 Worker 内部使用 `try...catch` 并通过 `postMessage` 传出错误信息。
  2. 在主线程通过 `worker.onerror` 监听未捕获的异常：

  JavaScript

  ```
  worker.onerror = (event) => {
    console.log('Worker 报错文件：', 		event.filename);
    console.log('报错行号：', event.lineno);
    console.log('错误信息：', event.message);
  };
  ```