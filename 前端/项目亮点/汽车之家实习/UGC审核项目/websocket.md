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

```
(function () {

    // XMLHttpRequest level 2
    function ajax (args) {
        var xhr = new XMLHttpRequest();
        xhr.open(args.method, args.url);
        xhr.onerror = function (e) {
            if (args.error) {
                args.error();
            }
        }
        xhr.onload = function (e) {
            // this.status this.statusText
            if (this.status == 200) {
                if (args.success) {
                    args.success(this.responseText);
                }
            } else {
                if (args.error) {
                    args.error();
                }
            }
        }
        xhr.send(args.data);
    }
    function decodeParameters (content) {
        var obj = {};
        if (content) {
            var arr = content.split("&");
            for (var i = 0, len = arr.length; i < len; i++) {
                var arr1 = arr[i].split("=");
                if (arr1.length > 1) {
                    obj[arr1[0]] = decodeURIComponent(arr1[1]);
                }
            }
        }
        return obj;
    }

    // 本地存储
    var LS = {
        getLastMsgId: function (userId) {
            var msgId = localStorage.getItem("athm_im_lastMsgId_" + userId);
            if (msgId == null) {
                return 0;
            }
            return msgId;
        },
        setLastMsgId: function (userId, lastMsgId) {
            localStorage.setItem("athm_im_lastMsgId_" + userId, lastMsgId);
        },
        getConversationId: function(targetType, targetId) {
            return targetType + "_" + targetId;
        },
        getConversation: function (userId, conversationId) {
            var str = localStorage.getItem("athm_im_conversation_" + userId + "_" + conversationId);
            if (str) {
                var conversation = JSON.parse(str);
                return conversation;
            }
            return null;
        },
        setConversation: function (userId, conversationId, conversation) {
            var jsonString = JSON.stringify({ unreadCount: conversation.unreadCount, lastMessage: conversation.lastMessage });
            localStorage.setItem("athm_im_conversation_" + userId + "_" + conversationId, jsonString);
        }
    };

    // lib
    var ImLib = {};
    ImLib.SDK_VERSION = "JSV5.0";
    ImLib.API_PATH = "//msg.api.autohome.com.cn/api";
    ImLib.SERVER_PATH = (window.location.protocol == "https:" ? "wss:" : "ws:") + "//socket.api.autohome.com.cn/websocket";
    ImLib.cmdType = {
        CONNECT_RESPONSE: "connectResponse",
        OFFLINE_MESSAGE: "offlineMessage",
        USER_UPDATE: "userUpdate",
        USER_MESSAGE: "userMessage",
        GROUP_JOIN: "groupJoin",
        GROUP_QUIT: "groupQuit",
        GROUP_UPDATE: "groupUpdate",
        GROUP_MESSAGE: "groupMessage",
        PERIODICAL_JOIN: "periodicalJoin",
        PERIODICAL_QUIT: "periodicalQuit",
        PERIODICAL_UPDATE: "periodicalUpdate",
        PERIODICAL_MESSAGE: "periodicalMessage"
    }
    ImLib.targetType = {
        USER: "user",
        GROUP: "group",
        PERIODICAL: "periodical"
    }

    // ImClient
    var ImClient = function() {
        var self = this;
        var isReconnection = false;
        window.addEventListener('offline', function () {
            if (self.isConnect()) {
                self.close();
                isReconnection = true;
            }
        });
        window.addEventListener('online', function () {
            if (isReconnection) {
                self.connect(self.token);
                isReconnection = false;
            }
        });
    }

    // 属性
    var ic = ImClient.prototype;
    ic.state = 0;  // 0未连接 1连接中 2已连接
    ic.token = "";
    ic.userId = 0;
    ic.channelId = ""; // 通道编号
    ic.source = "pc";
    ic.reconnectTime = 0; // 0不自动重连 >0自动重连
    ic.offlineMessageComplete = false;
    ic.conversationMap = null; // 会话
    ic.currentConversationId = null;
    ic.socket = null;

    // 是否连接
    ic.isConnect = function () {
        return this.state == 2;
    }

    // 连接
    ic.connect = function (token, userId) {
        var self = this;
        if (self.state !== 0) {
            return;
        }
        self.state = 1;
        self.token = token;
        self.userId = userId;
        self.offlineMessageComplete = false;
        self.conversationMap = {};
        if (self.reconnectTime < 1) {
            self.reconnectTime = 1;
        }
        // 连接中
        self.onConnect();
        // 创建连接
        self.socket = new WebSocket(ImLib.SERVER_PATH + "?token=" + encodeURIComponent(self.token) + "&source=" + self.source + "&sdkVersion=" + ImLib.SDK_VERSION + "&lastMsgId=" + LS.getLastMsgId(userId));
        // 连接打开
        self.socket.onopen = function (e) {
            this.send("type=opening");
            self.state = 2;
        }
        // 收到消息
        self.socket.onmessage = function (e) {
            // 解码
            var data = decodeParameters(e.data);
            // 连接响应
            if (data.type == ImLib.cmdType.CONNECT_RESPONSE) {
                if (data.code == 0) {
                    self.userId = data.userId;
                    self.channelId = data.channelId;
                    self.reconnectTime = 1;
                    self._heartbeat();
                    self.onConnectSuccess();
                } else {
                    self.reconnectTime = 0;
                    self.onConnectFail(data.code, data.message);
                }
            }
            // 离线消息
            if (data.type == ImLib.cmdType.OFFLINE_MESSAGE) {
                // 更新会话信息
                var readLastMsgId = 0;
                var conversationMap = self.conversationMap;
                var arr = JSON.parse(data.messages);
                for (var i = 0, len = arr.length; i < len; i++) {
                    var message = arr[i];
                    if (message.targetType == 2) {
                        var conversationId = LS.getConversationId(ImLib.targetType.GROUP, message.targetId);
                        var conversation = self.getConversation(ImLib.targetType.GROUP, targetId);
                        if (!conversation) {
                            conversation = { unreadCount: 0 };
                            conversationMap[conversationId] = conversation;
                        }
                        conversation.lastMessage = message;
                        if (message.fromUserId != self.userId) {
                            conversation.unreadCount++;
                        }
                    }
                    readLastMsgId = message.msgId;
                }
                // 离线消息完成
                if (data.flag == 0) {
                    // 设置最后消息id
                    if (readLastMsgId > 0) {
                        LS.setLastMsgId(self.userId, readLastMsgId);
                    }
                    // 设置会话
                    for (var key in conversationMap) {
                        LS.setConversation(self.userId, key, conversationMap[key]);
                    }
                    self.offlineMessageComplete = true;
                    self.onOfflineMessageComplete();
                }
            }
            // 用户更新
            if (data.type == ImLib.cmdType.USER_UPDATE) {
                //
            }
            // 用户消息
            if (data.type == ImLib.cmdType.USER_MESSAGE) {
                var targetId = data.fromUserId == self.userId ? data.userId : data.fromUserId;
                var conversationId = LS.getConversationId(ImLib.targetType.USER, targetId);
                if (self.currentConversationId == conversationId && data.fromUserId != self.userId) {
                    self.clearUnreadCount(ImLib.targetType.USER, targetId);
                }
                LS.setLastMsgId(self.userId, data.msgId);
                if (data.fromUserId != self.userId || data.channelId != self.channelId) {
                    self.onMessage(data);
                }
            }
            // 群组加入
            if (data.type == ImLib.cmdType.GROUP_JOIN) {
                //
            }
            // 群组退出
            if (data.type == ImLib.cmdType.GROUP_QUIT) {
                //
            }
            // 群组更新
            if (data.type == ImLib.cmdType.GROUP_UPDATE) {
                //
            }
            // 群组消息
            if (data.type == ImLib.cmdType.GROUP_MESSAGE) {
                var conversationId = LS.getConversationId(ImLib.targetType.GROUP, data.groupId);
                var conversation = self.getConversation(ImLib.targetType.GROUP, data.groupId);
                if (conversation) {
                    conversation.lastMessage = data;
                } else {
                    conversation = {
                        unreadCount: 0,
                        lastMessage: data
                    }
                    self.conversationMap[conversationId] = conversation;
                }
                if (self.currentConversationId != conversationId && data.fromUserId != self.userId) {
                    conversation.unreadCount++;
                }
                LS.setConversation(self.userId, conversationId, conversation);
                LS.setLastMsgId(self.userId, data.msgId);
                if (data.fromUserId != self.userId || data.channelId != self.channelId) {
                    self.onMessage(data);
                }
            }
            // 订阅加入
            if (data.type == ImLib.cmdType.PERIODICAL_JOIN) {
                //
            }
            // 订阅退出
            if (data.type == ImLib.cmdType.PERIODICAL_QUIT) {
                //
            }
            // 订阅更新
            if (data.type == ImLib.cmdType.PERIODICAL_UPDATE) {
                //
            }
            // 订阅消息
            if (data.type == ImLib.cmdType.PERIODICAL_MESSAGE) {
                //
            }
        }
        // 连接关闭
        self.socket.onclose = function (e) {
            if (self.state == 1) {
                self.state = 0;
                self.socket = null;
                self.onConnectFail(5, "服务器未响应");
            } else {
                self.state = 0;
                self.socket = null;
                self.onConnectInactive();
            }
            self._reconnect();
        }
        // 发生错误
        self.socket.onerror = function (e) {
            self.onError(e);
        }
    }

    // 重连
    ic._reconnect = function () {
        var self = this;
        if (!self.isConnect()) {
            if (self.reconnectTime > 0) {
                self.reconnectTime *= 2;
                if (self.reconnectTime > 64) {
                    self.reconnectTime = 0; // 放弃重连
                } else {
                    setTimeout(function () {
                        self.connect(self.token, self.userId);
                    }, self.reconnectTime * 1000);
                }
            }
        }
    }

    // 心跳
    ic._heartbeat = function () {
        var self = this;
        setTimeout(function () {
            if (self.isConnect()) {
                self.socket.send("type=h");
                self._heartbeat();
            }
        }, 50000);
    }

    // 关闭连接
    ic.close = function () {
        var self = this;
        if (self.socket) {
            self.reconnectTime = 0;
            self.socket.close();
        }
    }

    // 获取历史消息
    ic.getHistoryMessage = function (targetType, targetId, firstMsgId, size, callback) {
        var self = this;
        if (self.isConnect()) {
            // 获取记录
            if (firstMsgId == null) {
                var conversation = self.getConversation(targetType, targetId);
                if (conversation && conversation.firstMsgId) {
                    firstMsgId = conversation.firstMsgId;
                } else {
                    firstMsgId = 0;
                }
            }
            ajax({
                url: ImLib.API_PATH + "/" + targetType + "/tokenGetHistoryMessage?_appid=app.web&token=" + encodeURIComponent(self.token) + "&" + targetType + "Id=" + targetId + "&firstMsgId=" + firstMsgId + "&size=" + size,
                method: 'get',
                success: function (responseText) {
                    var re = JSON.parse(responseText);
                    if (re.returncode == 0) {
                        var list = re.result.list;
                        var length = list.length;
                        if (length >= size) {
                            re.result.hasMessage = true;
                        } else {
                            re.result.hasMessage = false;
                        }
                        // 会话更新
                        if (length > 0) {
                            var conversationId = LS.getConversationId(targetType, targetId);
                            var conversation = self.getConversation(targetType, targetId);
                            var firstMessage = list[0];
                            var lastMessage = list[length - 1];
                            if (conversation) {
                                conversation.firstMsgId = firstMessage.msgId;
                                if (lastMessage.msgId > conversation.lastMessage.msgId) {
                                    conversation.lastMessage = lastMessage;
                                    if (targetType == ImLib.targetType.GROUP) {
                                        LS.setConversation(self.userId, conversationId, conversation);
                                    }
                                }
                            } else {
                                conversation = {
                                    firstMsgId: firstMessage.msgId,
                                    unreadCount: 0,
                                    lastMessage: lastMessage
                                }
                                self.conversationMap[conversationId] = conversation;
                                if (targetType == ImLib.targetType.GROUP) {
                                    LS.setConversation(self.userId, conversationId, conversation);
                                }
                            }
                        }
                    }
                    callback(re);
                },
                error: function () {
                    callback({"returncode": 500, "message": "服务器错误"});
                }
            });
        } else {
            callback({"returncode": 501, "message": "未连接服务器"});
        }
    }

    // 发送消息
    ic.sendMessage = function (targetType, targetId, msgType, msgContent, marker, callback) {
        var self = this;
        if (self.isConnect()) {
            var data = new FormData();
            data.append('_appid', 'app.web');
            data.append('token', self.token);
            data.append(targetType + 'Id', targetId);
            data.append('msgType', msgType);
            data.append('msgContent', msgContent);
            data.append('channelId', self.channelId);
            data.append('source', self.source);
            data.append('marker', marker);
            ajax({
                url: ImLib.API_PATH + '/' + targetType + '/tokenSendMessage',
                data: data,
                method: 'post',
                success: function (responseText) {
                    var re = JSON.parse(responseText);
                    callback(re);
                },
                error: function () {
                    callback({"returncode": 500, "message": "服务器错误"});
                }
            });
        } else {
            callback({"returncode": 501, "message": "未连接服务器"});
        }
    }

    // 撤回消息
    ic.recallMessage = function (targetType, msgId, callback) {
        if (this.isConnect()) {
            var data = new FormData();
            data.append('_appid', 'app.web');
            data.append('token', this.token);
            data.append('msgId', msgId);
            data.append('source', this.source);
            data.append('marker', '');
            ajax({
                url: ImLib.API_PATH + '/' + targetType + '/tokenRecallMessage',
                data: data,
                method: 'post',
                success: function (responseText) {
                    var re = JSON.parse(responseText);
                    callback(re);
                },
                error: function () {
                    callback({"returncode": 500, "message": "服务器错误"});
                }
            });
        } else {
            callback({"returncode": 501, "message": "未连接服务器"});
        }
    }

    // 生成缩略图
    ic.makeThumbnail = function (file1, callback) {
        if (!file1) {
            throw "file1 is empty";
        }
        var image = new Image();
        image.onload = function () {
            var width = image.width;
            var height = image.height;
            var imageSize = width * height;
            if (imageSize > 20000) {
                var scale = Math.sqrt(imageSize / 20000);
                scale = Math.ceil(scale * 200) / 200;
                width = width / scale;
                height = height / scale;
            }
            var canvas = document.createElement("canvas"), context = canvas.getContext('2d');
            canvas.width = width;
            canvas.height = height;
            context.drawImage(image, 0, 0, width, height);
            try {
                var content = canvas.toDataURL("image/jpeg", 0.7);
                callback(content.substr(23));
            } catch (e) {}
        }
        image.src = URL.createObjectURL(file1);
    }

    // 上传图片
    ic.uploadImage = function (file1, callback) {
        if (!file1) {
            throw "file1 is empty";
        }
        if (this.isConnect()) {
            var data = new FormData();
            data.append('_appid', 'app.web');
            data.append('token', this.token);
            data.append('file1', file1);
            ajax({
                url: ImLib.API_PATH + '/sys/tokenUploadImage',
                data: data,
                method: 'post',
                success: function (responseText) {
                    var re = JSON.parse(responseText);
                    callback(re);
                },
                error: function () {
                    callback({"returncode": 500, "message": "服务器错误"});
                }
            });
        } else {
            callback({"returncode": 501, "message": "未连接服务器"});
        }
    }

    // 获取会话信息
    ic.getConversation = function (targetType, targetId) {
        var conversationId = LS.getConversationId(targetType, targetId);
        var conversation = this.conversationMap[conversationId];
        if (!conversation && targetType == ImLib.targetType.GROUP) {
            conversation = LS.getConversation(this.userId, conversationId);
            if (conversation) {
                this.conversationMap[conversationId] = conversation;
            }
        }
        return conversation;
    }

    // 清除会话气泡
    ic.clearUnreadCount = function (targetType, targetId) {
        var self = this;
        if (self.isConnect()) {
            var data = new FormData();
            data.append('_appid', 'app.web');
            data.append('token', self.token);
            data.append('targetType', targetType);
            data.append('targetId', targetId);
            ajax({
                url: ImLib.API_PATH + '/sys/tokenClearUnreadCount',
                data: data,
                method: 'post',
                success: function (responseText) {}
            });
        }
    }

    // 车友圈业务辅助
    ic._syncConversationList = function (callback) {
        var self = this;
        if (self.state != 0) {
            ajax({
                url: ImLib.API_PATH + '/sys/tokenGetConversationList?_appid=app.web&token=' + encodeURIComponent(this.token),
                method: 'get',
                success: function (responseText) {
                    var re = JSON.parse(responseText);
                    if (re.returncode == 0) {
                        var conversationMap = self.conversationMap;
                        var list = re.result.list;
                        for (var i = 0, len = list.length; i < len; i++) {
                            var item = list[i];
                            var conversationId = LS.getConversationId(item.targetType, item.targetId);
                            var conversation = conversationMap[conversationId];
                            if (conversation) {
                                conversation.unreadCount = item.unreadCount;
                                conversation.lastMessage = item.lastMessage;
                            } else {
                                conversationMap[conversationId] = {
                                    unreadCount: item.unreadCount,
                                    lastMessage: item.lastMessage
                                };
                            }
                        }
                    }
                    callback();
                },
                error: function () {
                    callback();
                }
            });
        }
    }
    ic._bindConversationList = function(list, callback) {
        var self = this;
        self._syncConversationList(function () {
            var totalUnreadCount = 0;
            for (var i = 0, len = list.length; i < len; i++) {
                var item = list[i];
                var conversation = self.getConversation(item.imTargetType, item.imTargetId);
                if (conversation) {
                    totalUnreadCount += conversation.unreadCount;
                    item.unreadCount = conversation.unreadCount;
                    item.lastMessage = conversation.lastMessage;
                    if (item.lastMessage) {
                        item.lastMsgTimestamp = item.lastMessage.msgTimestamp;
                    }
                }
            }
            callback(list, totalUnreadCount);
        });
    }
    ic.machConversationList = function(list, callback) {
        var self = this;
        if (list == null) {
            throw "list is null";
        } else {
            if (self.offlineMessageComplete) {
                self._bindConversationList(list, callback);
            } else {
                self.onOfflineMessageComplete = function () {
                    self._bindConversationList(list, callback);
                }
            }
        }
    }
    ic.machCyqConversationList = function(list, callback) {
        this.machConversationList(list, callback);
    }

    // 激活会话
    ic.activateConversation = function (targetType, targetId) {
        var self = this;
        var conversationId = LS.getConversationId(targetType, targetId);
        if (self.currentConversationId != conversationId) {
            if (targetType == ImLib.targetType.USER) {
                self.clearUnreadCount(targetType, targetId);
            } else {
                var conversation = self.getConversation(targetType, targetId);
                if (conversation && conversation.unreadCount > 0) {
                    conversation.unreadCount = 0;
                    LS.setConversation(self.userId, conversationId, conversation);
                }
            }
        }
        self.currentConversationId = conversationId;
    }

    // 事件
    ic.onConnect = function () {} // 连接中
    ic.onConnectSuccess = function () {} // 连接成功
    ic.onConnectFail = function (code, message) {} // 连接失败
    ic.onConnectInactive = function () {} // 连接断开
    ic.onError = function (e) {} // 发生错误
    ic.onMessage = function (message) {} // 收到消息
    ic.onOfflineMessageComplete = function () {} // 离线消息推送完毕

    // 暴露
    window.ImClient = ImClient;
    window.ImLib = ImLib;

})();超级详细解释代码
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

### 如何判断用户在线，下线

**上线判断**

客户端发起请求，创建websocket示例，在成功连接的时候触发onopen事件，此时客户端会发送一个 消息，消息的格式为{type=opening，token，lastMsgID}  ，服务端对token进行校验，并收到客户端发送的开场消息，会发送一条消息表示校验身份成功， 客户端收到消息，开启心跳包表示已经上线。





**下线判断**

用户点击退出的时候，触发oncloese方法，断开websocket连接。



网络原因 触发了offline事件，先进行一定次数的断线重连，如果重连失败，触发onclose方法。



心跳监测，服务端或者客户端长时间收不到心跳包，并且重连失败，表示下线。



### 如何实现群聊和私聊

消息推送是通过onmessage事件实现的，当服务端通过websocket发送消息的时候，客户端会根据data.type进入不同的分支。

**私聊**

如果接受消息的类型是user ，表示是私聊

私聊会在服务端维护一个Map对象，每次发送消息的时候检查用户是否在线，在线则进行推送，如果不在线存入离校消息库中。



**群聊**

写扩散： 客户端在群聊中发送一条消息，服务端接受消息，将在群聊中的所有用户都分发一次，需要插入多次消息，适合小群聊



读扩散：

不需要为每个用户写数据表，只需要一张表记录所有的聊天记录，服务端通过websocket向所有在线的用户进行发放。

如果是离线的用户，前端在下一次上线的时候会带上一个id向后端发起请求，表示收到的最后一条消息，后端对消息进行补发。

### 下线之后消息存在哪里

### websocket的数据类型

| **属性名**         | **类型**      | **说明**                                                     |
| ------------------ | ------------- | ------------------------------------------------------------ |
| **`type`**         | String        | 消息命令类型（见 `ImLib.cmdType`，如 `userMessage`, `groupMessage`） |
| **`msgId`**        | Number/String | 消息唯一 ID（用于去重和排序）                                |
| **`fromUserId`**   | Number        | 发送者 ID                                                    |
| **`userId`**       | Number        | 接收者 ID (私聊时)                                           |
| **`targetId`**     | Number        | 目标 ID（可能是用户 ID，也可能是群组 ID）                    |
| **`targetType`**   | String        | 目标类型 (`user`, `group`, `periodical`)                     |
| **`msgType`**      | Number        | 消息内容类型（如 1:文本, 2:图片, 3:语音等）                  |
| **`msgContent`**   | String        | 消息正文（如果是图片，通常是 URL 或 JSON 字符串）            |
| **`msgTimestamp`** | Number        | 消息发送的时间戳                                             |
| **`channelId`**    | String        | 通道编号（多端登录时用于区分设备）                           |
