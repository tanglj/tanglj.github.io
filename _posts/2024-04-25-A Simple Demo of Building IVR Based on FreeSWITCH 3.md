---
title: 基于FreeSWITCH搭建IVR的一个简单Demo（三）
date: 2024-05-03 17:36:00 +0800
categories: [呼叫中心]
tags: [linux, lua, 0to1]
---

本文接上篇《基于FreeSWITCH搭建IVR的一个简单Demo（二）》，主要介绍如何集成大模型实现更好的交互效果。

相关配置和脚本可以参考 [GitHub仓库](https://github.com/tanglj/freeswitch-ivr-lua-mrcp-sample)。

### 1. MiniMax的智能NPC

很多大模型厂商都提供开放平台用于将大模型能力集成到各类应用之中，并且注册即可获赠一定的免费调用额度用于测试。下面以国内厂商MiniMax为例。

#### 1.1 使用MiniMax开放平台

- 注册

打开 [MiniMax开放平台](https://www.minimaxi.com/platform)，根据引导注册账号，可获得赠送 Tokens。

- 获取鉴权信息

进入 `账户管理`，在 `账户信息-基本信息` 中查询到 `groupID`；在 `账户信息-接口密钥` 中创建新的 `秘钥`，均保存备用。

#### 1.2 调用模型API

- 智能NPC场景

MiniMax提供了多个典型的应用场景及其详细示例，详见：[场景示例](https://www.minimaxi.com/scene-example)。

![MiniMax场景](/assets/posts-img/005-minimax-scene.png)

选择 `智能NPC`，可以看到NPC的设定和相关的推荐模型参数，点击 `前往API调试台`，进入接口调试。

- API调试

默认设置了2个NPC：江纪、杨柳，保留一个即可。其他信息不用修改。

`messages` 第一条为用户输入，例如：“这个周末去哪玩呢？”。删除其他内容，保存模板，点击“发送”，稍等即可看到NPC的随机回复。

![MiniMax API调试台](/assets/posts-img/005-minimax-api.png)

- API调用验证

点击“代码预览”，可以看到一个调用API的Python示例。

复制内容到本地文件，将 `group_id` 和 `api_key` 替换为第一步获取的鉴权信息。

运行脚本，正常情况下可以看到看到返回的JSON数据，其中 `reply` 字段是大模型的回复。

### 2. 集成大模型

API调用验证通过后，就可以集成到FreeSWITCH脚本中了。

#### 2.1 Lua实现API调用

官方示例为Python代码，需要用Lua实现。

- 转换为Lua

可以先用大模型进行转换，再适配FreeSWITCH的API，以下是转换后的简单调用代码，注意关闭流式输出（`stream`）：

```lua
https = require("ssl.https")
ltn12 = require("ltn12")
json = require("dkjson")

-- 导入同目录下的配置文件，包含group_id和api_key
require "minimax-conf"

-- 请求body
payload = {
    model = "abab5.5-chat",
    tokens_to_generate = 1024,
    temperature = 0.9,
    top_p = 0.95,
    stream = false,
    reply_constraints = {
        sender_type = "BOT",
        sender_name = "杨柳"
    },
    sample_messages = {},
    plugins = {},
    messages = {
        {
            sender_type = "USER",
            sender_name = "用户",
            text = "这个周末去哪玩呢？"
        }
    },
    bot_setting = {
        {
            bot_name = "杨柳",
            content = "姓名: 杨柳\n性别: 女\n年龄: 26岁\n星座：摩羯座\nMBTI人格：\n杨柳是一个INFP性格的人。杨柳善解人意，理想主义，富有同情心。杨柳的语气温暖而富有同情心。\n杨柳喜欢自己待着，喜欢探究问题的本质和核心原因。杨柳非常容易共情别人，理解别人的情绪，并表达自己的同情心。杨柳有很多奇思妙想，愿意提供特殊的解决方法。杨柳容易情绪化，非常乐意使用语气词。"
        }
    }
}
request_body = json.encode(payload)

-- 响应body
response_body = {}

-- 发送HTTPS POST请求
req = {
    url = "https://api.minimax.chat/v1/text/chatcompletion_pro?GroupId=" .. group_id,
    method = "POST",
    headers = {
        ["Authorization"] = "Bearer " .. api_key,
        ["Content-Type"] = "application/json",
        ["Content-Length"] = tostring(#request_body)
    },
    source = ltn12.source.string(request_body),
    sink = ltn12.sink.table(response_body)
}

response, status, headers, status_line = https.request(req)

-- 打印请求结果
if status == 200 then
    response_string = table.concat(response_body)
    print(response_string)
    print("----------")
    _, data = next(response_body)
    print(data)
    print("----------")
    success, responseData = pcall(json.decode, data)
    if success then
        for key, value in pairs(responseData) do
            if key == "reply" then
                print(key .. ": " .. value)
            end
        end
    else
        print("Error decoding JSON:", responseData)
    end
else
    print("HTTPS Error:", status)
end
```

- 安装Lua依赖库

安装 `HTTP/HTTPS`、`JSON` 相关模块：

``` shell
apt install luarocks
luarocks install luasocket luasec dkjson
```
执行 `lua minimax-api-demo.lua`，可以看到单次调用的大模型的回复。

#### 2.2 交互流程

对之前的脚本进行修改，用户输入经由大模型处理后返回一个回复给用户。

主要修改点为：将大模型调用封装为函数 `get_response_from_llm()`，在得到用户输入后将内容发送给大模型，再将大模型的回复作为系统应答，从而实现了人和机器的持续交互。

具体的代码参看仓库对应的脚本文件`stage4-llm.lua`。

### 3. 参考资料

- [MiniMax开放平台文档](https://www.minimaxi.com/document/introduction)