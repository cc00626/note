## 背景

项目问卷调研显示，70%创作者希望增加粉丝私信功能，方便创作者给粉丝回消息，接受商业广告内容，不用打开app端直接在网页端即可。



## 实现

**长轮询**

发起一次请求后，服务端会等待是否有新数据，有就返回。然后客户端发起下一次请求，等待新消息。

相比短轮询，请求次数减少。

```
async function pollMessage() {
  try {
    const res = await fetch("/api/message/poll");
    const data = await res.json();

    if (data.messages.length) {
      renderMessage(data.messages);
    }

    pollMessage(); // 再次轮询
  } catch (e) {
    setTimeout(pollMessage, 3000);
  }
}

pollMessage();
```



**短轮询**

每隔一段时间不断发起请求

优点：实现简单

缺点：请求次数过多，浪费服务器资源。

```
function poll() {
  fetch("/api/message")
    .then(res => res.json())
    .then(data => {
      if (data.length > 0) {
        console.log("收到消息", data);
      }
    });
}

setInterval(poll, 3000);
```



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



性能上sse和websocket更好，sse是单向通讯，本场景是双向，最后选择了websocket



## websocket连接建立的过程

1. 客户端和服务端先建立http连接

2. 客户端发起http请求，带上特殊头部，表示要升级websocket协议。

   ```javascript
   GET /chat HTTP/1.1
   Host: example.com
   Upgrade: websocket            // 告诉服务器我想升级到 WebSocket
   Connection: Upgrade           // 必须的，表示“升级连接”
   Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==   // 随机生成的 Base64 字符串
   Sec-WebSocket-Version: 13     // 指定 WebSocket 协议版本
   ```

3. 服务器检查请求，查看字段是否正确，如果正确，返回101状态码

   ```javascript
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: HSmrc0sMlYUkAGmm5OPpG2HaGWk=
   ```

4. 客户端收到101状态码，升级为websocket协议

## websocket心跳重连机制

websocket是长连接，在客户端长时间没有发送消息的话，websocket就认为客户端断开。就会关闭断开连接。

**心跳检测机制**：客户端每 50 秒发送一次 ping 消息，服务器返回 pong 作为响应。如果超时未收到 pong，会判定连接异常并触发重连；



如果断开连接会触发onerror，或者onclose回调，开始进行重连，使用指数退避算法，防止短时间内重复连接。

```
function tryReconnect() {
    if (reconnectAttempts < maxReconnect) {
        let delay = Math.pow(2, reconnectAttempts) * 1000; // 指数退避
        reconnectAttempts++;
        console.log(`尝试重连，延迟 ${delay / 1000}s`);
        setTimeout(() => connect(), delay);
    } else {
        console.log("重连失败，已达最大次数");
    }
}
```



## 处理消息同步

为了保证消息不丢失，代码引入了 **`lastMsgId`** 的概念：

1. **初始化：** 连接 WebSocket 时，将 LocalStorage 中存储的最后一个消息 ID 传给后端。
2. **补发：** 服务器根据这个 ID，将客户端离线期间的所有消息通过 `OFFLINE_MESSAGE` 类型一次性推送到前端。
3. **持久化：** 收到消息后立即更新本地存储的 `lastMsgId`，确保下次登录时能无缝衔接。



## 处理未读消息数

SDK 在内存中维护了一个 `conversationMap`，它的结构本质上是一个 KV 映射：

- **Key:** `conversationId` (由 `targetType` + `targetId` 组成，如 `user_123` 或 `group_456`)。
- **Value:** 包含 `unreadCount`（未读数）和 `lastMessage`（最后一条消息体）的对象。

同时，利用 `LocalStorage` 进行持久化，确保页面刷新后未读数不会归零。



当 `onmessage` 监听到新消息时，SDK 会执行以下逻辑判断：

1. **判定当前状态：**
   - 获取新消息的 `conversationId`。
   - 对比 `this.currentConversationId`（用户当前正在看的聊天窗口）。
2. **执行自增：**
   - **如果当前不在该窗口：** 从 `conversationMap` 找到对应对象，执行 `unreadCount++`。
   - **如果当前就在该窗口：** 保持 `unreadCount` 为 0，不进行累加。
3. **持久化：** 立即调用 `LS.setConversation()` 将更新后的数字写入本地存储。



在 IM 系统中，未读数管理的核心矛盾在于**“多端同步”**与**“本地展示”**的一致性。代码中通过 `conversationMap`（内存）、`LocalStorage`（持久化）以及后端接口（多端同步）三位一体地解决了这个问题。

以下是具体的实现思路拆解：

SDK 在内存中维护了一个 `conversationMap`，它的结构本质上是一个 KV 映射：

- **Key:** `conversationId` (由 `targetType` + `targetId` 组成，如 `user_123` 或 `group_456`)。
- **Value:** 包含 `unreadCount`（未读数）和 `lastMessage`（最后一条消息体）的对象。

同时，利用 `LocalStorage` 进行持久化，确保页面刷新后未读数不会归零。



## 如何实现

项目问卷调研显示，70%创作者希望增加粉丝私信功能，方便创作者给粉丝回消息，接受商业广告内容，不用打开app端直接在网页端即可。使用websocket实现前后端实时双向通信，包括消息发送，接受，未读消息状态同步。

并且还通过断线重连，心跳监测，等机制确保了通信的稳定性。



## 断线重连怎么实现的？为什么处理断线重连

原因：websocket是长连接，在客户端长时间没有发送消息的话，websocket就认为客户端断开。就会关闭断开连接。

我们会监听浏览器的网络状态变化，通过监听online和offline事件，当浏览器断网的时候关闭websocket连接，标记需要进行重连，在重新联网时候进行重连。

同时我们每50s发送一次心跳包，使连接保持活跃，如果服务端没有相应，也会触发重连。

在进行重连的视乎我们是哦那个指数退避，防止短时间内发起大量请求。



## 在重连的时候，app端有消息发送，怎么实现消息恢复？

重连成功之后，需要恢复消息状态。

连接时会带一个参数：

```
lastMsgId
```

服务端把 **msgId 大于 lastMsgId 的所有消息** 作为离线消息推送给客户端。



## 会话未读消息更新

我们将每个会话的聊天记录、未读状态等信息都存在 redux状态树中，并按用户 ID 维护消息 Map。

- 当收到 WebSocket 消息时，会触发对应会话的 `messageList[userId].push(message)`；
- 如果当前激活窗口是对应用户，会话内容直接更新，同时设为已读；
- 如果不是当前窗口，就将未读数加一，并在侧边栏显示红点；
- 由于状态是全局响应式的，组件会自动监听相关数据变更，更新 UI；
- 同时在切换会话窗口时我们还会调用接口同步消息状态，确保消息持久化同步。

这种方式保证了消息和 UI 的精准同步，也支持多个窗口独立导航和消息并发处理。



### websocket多个连接怎么处理

