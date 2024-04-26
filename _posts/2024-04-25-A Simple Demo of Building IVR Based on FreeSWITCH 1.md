---
title: 基于FreeSWITCH搭建IVR的一个简单Demo（一）
date: 2024-04-26 17:21:00 +0800
categories: [呼叫中心]
tags: [linux, lua]
---

本文介绍如何基于 `FreeSWITCH` 平台搭建一个简单的 `IVR`。

相关配置和脚本可以参考 [GitHub仓库](https://github.com/tanglj/freeswitch-ivr-lua-mrcp-sample)。

### 1. 术语介绍

**`IVR`**：Interactive Voice Response，即交互式语音应答。在呼叫中心系统中得到广泛应用，它利用语音识别、语音合成、语义理解等AI技术，并接入其他业务系统，自动回复和解决客户的问题。`IVR`可以全天候自动化提供服务，帮助实现呼叫中心的降本增效。

**`FreeSWITCH`**：一款开源的电话软交换平台。支持多种音频、视频、文本和其他媒体类型，通过插件系统可以方便地集成各种功能，从而构建一个功能丰富的通信平台，例如`IVR`。

**`Lua`**：一种轻量且小巧的脚本语言，可用于在`FreeSWITCH`中编写会话处理脚本。

**`MRCP`**：媒体资源控制协议（Media Resource Control Protocol），是一种用于`FreeSWITCH`调用语音识别和语音合成服务的通信协议。

**`ASR`**：自动语音识别（Automatic Speech Recognition），是一种将人类语音转换为文本的技术，可将电话中用户的声音转换为文字，用于后续处理。

**`TTS`**：文本转语音（Text To Speech），是一种将文本自动转换为人类语音的技术，可将系统的文本的应答转换成语音回复给用户。

**`NLP`**：自然语言处理（Natural Language Processing），是利用计算机技术来分析、理解和处理自然语言（通常指文本）的技术，可理解用户意图并回答问题。

**`LLM`**：大语言模型（Large Language Model），是基于 `Transformer` 架构的超大参数神经网络模型，与以往的NLP技术相比具备更好的理解和生成能力。

### 2. 安装FreeSWITCH

> 环境：Ubuntu 22.04

#### 2.1 安装与启动

- 获取源码

下载最新的发布版源码包：<https://files.freeswitch.org/freeswitch-releases/>。

解压并进入目录。

- 修改配置

修改 `module.conf` 文件，注释掉 `applications/mod_signalwire` 和 `applications/mod_spandsp` 模块。

> `mod_signalwire` 为不需要的商业化模块

> `mod_spandsp` 为传真模块，可能导致安装失败，所以去除

- 安装依赖

用 `apt` 安装依赖：

```bash
apt install -y libtool-bin pkg-config libpq-dev libtiff-dev libcurl4-dev libcurl4-openssl-dev libpcre3-dev libspeexdsp-dev libldns-dev libedit-dev libavformat-dev libswscale-dev
```

有部分依赖 `apt` 仓库中的版本不满足要求，需要下载最新源码进行编译安装：

| 依赖 | 源码下载地址 |
| --- | --- |
| spandsp | <https://github.com/freeswitch/spands> |
| sofia-sip | <https://github.com/freeswitch/sofia-sip> |
| sqlite3 | <https://www.sqlite.org/download.html> |
| libks | <https://github.com/signalwire/libks> |
| yasm | <https://github.com/yasm/yasm> |

- 编译安装

执行 `./configure` 命令。若提示缺少依赖，根据提示安装。

执行 `make` 命令。若提示缺少依赖，根据提示安装，安装后需先执行 `./configure` 再执行 `make`。

编译成功示例：

![FreeSWITCH编译成功](assets/posts-img/003-fs-make-success.png)

编译完成后，执行 `make install` 命令进行安装。

- 启动程序

建立命令行程序软链接，方便使用：

```bash
ln -sf /usr/local/freeswitch/bin/freeswitch /usr/local/bin/
ln -sf /usr/local/freeswitch/bin/fs_cli /usr/local/bin/
```

启动FreeSWITCH：`freeswitch -nonat -nc`。

稍等一会待启动完成，进入FreeSWITCH控制台：`fs_cli`。

![FreeSWITCH启动成功](assets/posts-img/003-fs-run-success.png)

#### 2.2 配置

- 修改FreeSWITCH基本配置

编辑 `/usr/local/freeswitch/conf/vars.xml`，`default_password` 改为默认值 `1234` 以外的值（例如：`1335779`），保证安全。

如果是公网环境，`internal_sip_port` 和 `internal_tls_port` 改成 `5060/5061` 以外的可用端口（例如：`15060/15061`），避免受到攻击。

如果是公网环境，`external_rtp_ip` 和 `external_sip_ip` 配置为服务器公网IP（例如：`10.10.5.1`）。

- 修改FreeSWITCH启动加载模块配置

编辑 `/usr/local/freeswitch/conf/autoload_configs/modules.conf.xml`，注释一些未安装的模块：

```xml
#<load module="mod_spandsp"/>
#<load module="mod_signalwire"/>
```

- 重启FreeSWITCH

执行 `freeswitch -stop` 或在FreeSWITCH控制台执行 `shutdown`，再启动。

### 3. 实现呼叫

#### 3.1 注册分机

- 安装软电话

选择安装 `Adore SIP Client`(iOS)、`CSipSimple/Linphone`(Android)、`MicroSIP`(Windows)、`Telephone`(Mac)。

> 最好使用手机，避免收音问题

- 注册分机

在软电话中配置账号：

| 配置项 | 值 |
| --- | --- |
| 服务器 | FreeSWITCH的IP:之前配置的端口，例如：`10.10.5.1:15060` |
| 账号 | 1001-1020中任意一个，例如：`1001` |
| 密码 | 填写之前配置的密码，例如：`1335779` |

- 查看注册状态

注册成功在软电话和FreeSWITCH控制台均会有提示。

在控制台中，执行`sofia status profile internal reg`，若能看到相关信息表明注册成功。

#### 3.2 使用echo测试分机

在FreeSWITCH控制台中，执行 `originate user/1001 &echo`，其中 `1001` 是注册的分机号。

执行成功注册的软电话会振铃，接通后可以听到自己说话的“回声”。

### 4. 配置简单的Lua流程

#### 4.1 拨号计划(dialplan)

- 新建dialplan

编辑`/usr/local/freeswitch/conf/dialplan/default.xml`，在 `content` 元素中添加一个新的 `extension`：

```xml
<extension name="stage1-prompt">
    <condition field="destination_number" expression="^1101$">
        <action application="lua" data="stage1-prompt.lua"/>
    </condition>
</extension>
```

新增的配置指定 `1101` 分机调用 `Lua` 模块，执行 `stage1-prompt.lua` 脚本。

- 加载dialplan

保存后在FreeSWITCH控制台中，执行 `reloadxml` 使之生效。

#### 4.2 Lua脚本

- 创建脚本

在 `/usr/local/freeswitch/scripts/` 目录下新建 `stage1-prompt.lua` 脚本，内容如下：

```lua
-- 接通
session:answer()

-- 控制台打印日志
session:consoleLog("INFO", "开始播放音频\n")

-- 播放音频 “欢迎你来到新世界”
session:streamFile("/usr/local/freeswitch/storage/stage1-test.wav")

session:consoleLog("INFO", "结束播放音频\n")

-- 挂机
session:hangup()
```

`stage1-test.wav` 是存在于本地的文件。

- 测试脚本

使用软电话直接拨打 `1101`，如果听到播报“欢迎你来到新世界”然后自动挂机，则表明流程配置成功，在控制台应该同时可以看到两行由脚本插入的INFO级别日志。

![Lua脚本日志](assets/posts-img/003-fs-lua-sample.png)

- 脚本API

FreeSWITCH Lua流程的更多写法可以参考：[官方Lua API](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Client-and-Developer-Interfaces/Lua-API-Reference/)

### 5. 参考资料

- FreeSWITCH官方文档：<https://freeswitch.org/confluence/>

- 中文图书《FreeSWITCH权威指南》：<https://book.douban.com/subject/25902109/>