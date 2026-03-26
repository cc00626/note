# Sse对话

## 1.背景

 在用户输入知识库中的内容后需要，在页面中我们需要进行实时的通讯。需要前端和后端双方不断发送消息。

## 2.技术调研

**协议方面**

**长轮询，短轮询**

**SSE**

1. 客户端只能向服务端发送一次请求，后续需要服务端向客户端发送请求。
2. sse基于http协议，不需要重复进行握手
3. 适合发送文本消息。每个消息以\n\n进行结尾
4. 不需要手动处理重连机制，自动支持重连，实现起来简单。

**websocket**

1. 双向通信，服务端和客户端都能互相发送请求。
2. 基于tcp协议，需要进行握手的过程。
3. 可以传递文本消息或者二进制消息
4. 需要手动处理心跳重连，断线重连机制。



由于原生的sse有以下几个缺点

1. 只支持get请求，不能携带请求头和请求体
2. 如果服务端没有断开，不能够进行重连

如果自己封装需要处理数据格式的问题，使用axios，fetch封装返回的是数据流，需要decode，实现复杂。

基于以上考量使用了[fetch-event-source](https://github.com/Azure/fetch-event-source)这样一个库开进行实现



## 3.实现

后端会通过ContentType：text/event-stream 响应头设置返回的格式，然后返回数据。

后端会将ai返回的结果编程sse标准的格式，将内容data.chiose.content放入  data：\n\n中 ，然后根据finish_reason,判断是否返回完成，返回完成则断开连接。

 前端通过监听onMessage的回调能够拿到数据。数据会包装成一个对象，对象中的data属性存放数据的内容。event表示数据的标识，将data中的数据打印出来，当数据添加完成之后会触发onclose事件关闭sse。

sse会返回markdown格式的数据，然后通过markdown-it 库转化为html。 

通过v-html将数据进行渲染。





### sse如何实现的

“在做知识库 AI 对话功能时，我们需要实现类似 ChatGPT 的‘打字机’实时输出效果。我调研了 WebSocket 和 SSE。 虽然 WebSocket 支持双向通信，但 AI 对话本质上是 **‘一问多答’**，即用户发一个 Prompt，服务端持续推送。SSE 基于 HTTP 协议，比 WebSocket 更轻量，且天然支持断线重连。 不过，原生 `EventSource` 有个致命缺点：**只支持 GET 请求且不能自定义 Header**。因为对话接口通常需要 POST 大段 JSON 并在 Header 传 Token，所以我最终选用了 `@microsoft/fetch-event-source` 这个库，它通过封装 Fetch API 完美解决了 POST 请求和鉴权问题。



“在底层实现上，SSE 的核心是建立一个**长连接的 HTTP 流**。 服务端需要在响应头中设置 `Content-Type: text/event-stream`，并禁用缓存 `Cache-Control: no-cache`。

SSE 的返回格式是**纯文本流**。每一条消息由多个字段组成，并以**双换行符 `\n\n`** 作为消息的分隔符。常见的字段有：

- **`data`**：存放具体内容，我们通常会将 AI 生成的增量文本序列化为 JSON 放在这里。
- **`event`**：自定义事件名，方便前端区分‘正在输入’或‘回答结束’。
- **`id`**：用于断线重连时告知服务端上次接收的位置。
- **`retry`**：指定重连间隔。

比如在我的项目中，服务端会不断推送 `data: {"content": "xxx"}\n\n`，通过监听onMessage回调拿到数据，最后当 AI 生成结束，后端会根据 `finish_reason` 发送一个结束标识。”



### SSE是如何实现的，返回的数据格式是什么

SSE 返回的是**纯文本流**，每条消息由**两个换行符（`\n\n`）**分隔。每条消息内部包含多个字段：

- **`data`**: 消息内容（最重要的部分）。如果是 JSON 对象，通常写成 `data: {"text": "Hello"}`。
- **`event`**: 事件类型（可选），前端可以用 `addEventListener` 监听特定事件。
- **`id`**: 消息 ID（可选），用于重连时服务器从特定位置恢复。
- **`retry`**: 服务器建议的重连时间。



### 如何处理断线重连

如果是前端断开，会触发onError回调，返回一个毫秒数，前端会根据这个毫秒数自动进行重连

```
let retryCount = 0;
const MAX_RETRY = 3;

onerror(err) {
  // 1. 如果是网络中断（没有 response 对象），尝试自动重连
  if (!err.status) {
    if (retryCount < MAX_RETRY) {
      retryCount++;
      const delay = Math.pow(2, retryCount) * 1000; // 指数退避：2s, 4s, 8s
      console.warn(`网络异常，${delay}ms 后进行第 ${retryCount} 次重连...`);
      return delay; // 返回毫秒数，库会自动重新发起请求
    } else {
      showToast('网络连接已断开，请检查网络后重试');
      throw err; // 达到最大次数，停止重连
    }
  }
}
```



如果是后端正常断开连接，会触发onclose回调。如果是异常，需要重新调用请求函数

```
onclose() {
  if (!isFinished) { // isFinished 是在 onmessage 里根据后端结束标识设置的变量
    console.error('检测到后端异常关闭，且未收到结束标识');
    // 手动触发重连逻辑
    setTimeout(() => {
      startSSE(); // 重新调用封装好的 SSE 发起函数
    }, 2000);
  } else {
    console.log('会话正常结束');
  }
}
```



### 如何防止重发的数据不重复

SSE 协议标准中内置了对 `id` 字段的支持。`fetch-event-source` 库遵循这一标准。

- **记录 ID**：在 `onmessage` 回调中，库会自动解析消息中的 `id` 字段。
- **自动携带**：当连接异常触发 `onerror` 并返回重连等待时间后，库在下一次发起 `fetch` 请求时，会自动在 HTTP Header 中加入： `Last-Event-ID: <你上次收到的最后一条 ID>`。

**代码逻辑示例：**

JavaScript

```
onmessage(msg) {
    if (msg.id) {
        // 库内部会记录这个 id，下一次重连会自动带上
        // 你也可以自己在外部存一份用于业务逻辑
        this.lastId = msg.id; 
    }
    // 渲染逻辑...
}
```



------

2. **后端支持：偏移量处理（关键）**

这是最难的一步，需要后端配合。后端在接收到请求后，应检查 Header 中是否有 `Last-Event-ID`。

- **存储响应快照**：后端需要将 AI 生成的内容分段存储在缓存（如 Redis）中，并给每一段分配一个自增 ID。
- **根据 ID 切片**：
  - 如果 `Last-Event-ID` 为空，从头开始推送。
  - 如果 `Last-Event-ID` 有值（比如为 `5`），后端去缓存中查找该会话的消息序列，从第 `6` 段开始继续推送。

### 为什么不用fetch 和axios

| **特性**          | **原生 EventSource**      | **直接用 Fetch / Axios** | **@microsoft/fetch-event-source** |
| ----------------- | ------------------------- | ------------------------ | --------------------------------- |
| **请求类型**      | 仅支持 **GET**            | 支持所有 (GET/POST)      | 支持所有 (GET/POST)               |
| **自定义 Header** | **不支持** (无法传 Token) | 支持                     | 支持                              |
| **自动重连**      | 内置 (简单但不可控)       | **需手动实现**           | 内置 (高度可定制)                 |
| **数据解析**      | 自动解析为消息对象        | **需手动解析字节流**     | 自动解析为消息对象                |
| **重连 ID 记录**  | 自动 (Last-Event-ID)      | 需手动维护               | 自动维护                          |
