---
title: 基于FreeSWITCH搭建IVR的一个简单Demo（二）
date: 2024-04-30 17:24:00 +0800
categories: [呼叫中心]
tags: [linux, lua]
---

本文接上篇《基于FreeSWITCH搭建IVR的一个简单Demo（一）》，主要介绍如何集成`ASR`、`TTS`实现语音交互。

相关配置和脚本可以参考 [GitHub仓库](https://github.com/tanglj/freeswitch-ivr-lua-mrcp-sample)。

### 1. 安装MRCP模块

新版FreeSWITCH源码已经不再包含MRCP模块，需另行安装。

源码仓库地址：<https://github.com/freeswitch/mod_unimrcp>。

仓库README中提供了详尽的安装方法。

- 安装UniMRCP依赖

```bash
# 下载依赖源码
wget https://www.unimrcp.org/project/component-view/unimrcp-deps-1-6-0-tar-gz/download -O unimrcp-deps-1.6.0.tar.gz
tar xvzf unimrcp-deps-1.6.0.tar.gz
cd unimrcp-deps-1.6.0

# 安装依赖：apr
cd libs/apr
./configure --prefix=/usr/local/apr
make
sudo make install 
cd ..

# 安装依赖：apr-util
cd apr-util
./configure --prefix=/usr/local/apr
make
sudo make install
cd ..
```

- 安装UniMRCP

```bash
# 安装unimrcp
git clone https://github.com/unispeech/unimrcp.git
cd unimrcp
./bootstrap
./configure
make
make install
cd ..
```

- 安装mod_unimrcp

```bash
git clone https://github.com/freeswitch/mod_unimrcp.git
cd mod_unimrcp
export PKG_CONFIG_PATH=/usr/local/freeswitch/lib/pkgconfig:/usr/local/unimrcp/lib/pkgconfig
./bootstrap.sh
./configure
make
make install
```

### 2. 部署百度云MRCP语音服务

百度智能云提供了MRCP接口的语音识别和语音合成服务，与FreeSWITCH对接可以实现实时语音识别及合成，基于此可以实现复杂的智能语音交互流程。

#### 1.1 获取认证信息

首先开通百度云的语音服务，接入方法可以按照 [官网指引](https://ai.baidu.com/ai-doc/SPEECH/qknh9i8ed) 完成。

- 领取免费资源

在百度云控制台中，进入 `语音技术`，在 `概览` 中的 `操作指引` 可领取免费资源。

![领取资源](assets/posts-img/004-bce-speech-free.png)

领取时 `服务类型` 选择 `呼叫中心语音`。

- 创建应用

根据文档指引创建引用，`接口选择` 需要包含 `呼叫中心语音`。

![选择接口](assets/posts-img/004-bce-speech-interface.png)

完成后取得 `AppID`、 `API Key` 待用。

#### 1.2 部署百度云MRCP Server

百度MRCP Server的安装可以参考 [官方文档](https://ai.baidu.com/ai-doc/SPEECH/7kaxz0h2z)。

- 下载

通过 [下载页面](https://ai.baidu.com/sdk) 取得安装包后，将其上传并解压到 `/usr/local/baidu-unimrcp`。

- 配置

编辑 `conf/mrcp-asr.conf` 和 `conf/mrcp-proxy.conf` 文件，将 `AUTH_APPID` 和 `AUTH_APPKEY` 均配置为上一步得到的 `AppID` 和 `API Key`。

编辑 `conf/logger.xml` 文件，将 `output` 设置为 `CONSOLE,FILE` 便于调试。

- 启动

执行 `./bootstrap.sh` 配置GCC环境。

进入 `mrcp-server` 目录，执行 `./bin/unimrcpserver -r . &` 启动服务。

执行 `netstat -nlptu | grep unimr`，如果 `5060`、`1544` 和 `1554` 端口被该进程占用，服务成功启动。

- 验证

将 `mrcp-server/lib` 目录加入系统环境变量中：`export LD_LIBRARY_PATH=/usr/local/baidu-unimrcp/mrcp-server/lib:$LD_LIBRARY_PATH`。

切换到 `mrcp-server/bin` 目录下，执行 `./asrclient`，输入 `run grammar.xml xeq.pcm`，等待一段时间可看到返回的识别结果，完成后输入 `quit` 回车退出。

![识别成功](assets/posts-img/004-bce-speech-asr.png)

执行 `./unimrcpclient`，输入 `run synth`，等待一段时间可看到 `SPEAK-COMPLETE` 合成结束，输入 `quit` 回车退出。

![合成成功](assets/posts-img/004-bce-speech-tts.png)

> 部署问题可参考官方文档或百度

#### 1.3 配置FreeSWITCH mrcp_profile

配置可参考 [官方文档](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Modules/mod_unimrcp_6586728/) 的 `MRCPv2 example`。

- 新建配置

在`/usr/local/freeswitch/conf/mrcp_profiles/`目录新建文件`baidu-cloud.xml`

仿照填写配置，其中 `server-ip` 是百度MRCP Server的IP，`server-port` 是服务端口（默认5060）。

```xml
<include>
 <profile name="baidu-cloud" version="2">
   <param name="server-ip" value="172.21.25.184"/>
   <param name="server-port" value="5060"/>
   <param name="sip-transport" value="udp"/>
   <param name="codecs" value="PCMU PCMA L16/96/8000"/>
   <recogparams>
       <param name="start-input-timers" value="false"/>
   </recogparams>
 </profile>
</include>
```

- 加载配置

添加后在FreeSWITCH控制台中，执行 `reload mod_unimrcp` 使之生效。

### 3. Lua中调用识别和合成资源

- dialplan配置

在`/usr/local/freeswitch/conf/dialplan/default.xml`添加新的extension，分机号为1102

```xml
<extension name="stage2-dialog">
    <condition field="destination_number" expression="^1102$">
        <action application="lua" data="stage2-dialog.lua"/>
    </condition>
</extension>
```

- Lua脚本

在`/usr/local/freeswitch/scripts/`目录下新建`stage2-dialog.lua`脚本，内容如下：

```lua
-- 接通
session:answer()

-- 控制台打印日志
session:consoleLog("INFO", "开始播报识别\n")

-- 设置TTS参数
session:setVariable("tts_engine", "unimrcp:baidu-cloud")
session:setVariable("tts_voice", "fduxiaowen")

-- 播放TTS提示语并开启识别
session:execute("play_and_detect_speech", "say:请输入一句话 detect:unimrcp:baidu-cloud {start-input-timers=false,No-Input-Timeout=3000,Speech-Complete-Timeout=1200}http://192.168.0.1/grammars/not-exist.gram")
local result = session:getVariable('detect_speech_result')
if result == nil then
    session:consoleLog("INFO", "引擎异常\n")
elseif result == "Completion-Cause: 001" then
    session:consoleLog("INFO", "未识别\n")
elseif result == "Completion-Cause: 002" then
    session:consoleLog("INFO", "未输入\n")
else
    session:consoleLog("INFO", "识别结果是：\n".. result)
end

session:consoleLog("INFO", "结束播报识别\n")

-- 挂机
session:hangup()
```

- 脚本说明

在这个脚本中，通过 `session:execute()` 方法调用了FreeSWITCH的 `play_and_detect_speech` 应用，其参数包含3部分：

1. `say:请输入一句话` 调用TTS播放提示语；

2. `detect:unimrcp:baidu-cloud {start-input-timers=false,No-Input-Timeout=3000,Speech-Complete-Timeout=1200}` 是识别的配置，其中 `unimrcp:baidu-cloud` 是选择的百度profile name，后面 `{}` 内配置的是识别参数：未输入超时为3秒，识别完成超时1.2秒，其他可选的参数及其意义可参考 [MRCPv2规范](https://tools.ietf.org/html/rfc6787)；

3. 最后的部分是语法，这在百度引擎中没有任何作用，配置一个合法的HTTP文件路径即可。

`play_and_detect_speech` 调用完成后，识别结果被存储在 `detect_speech_result` 会话变量中，如果没有任何结果（比如引擎无法调用）则为空，异常情况则存储MRCP头（001未识别/002未输入等），有识别结果则为XML字符串。

- 测试

使用软电话拨打1102，在提示音后输入任意一句话，稍等一下后自动挂机，如果在控制台看到对应XML文本的识别结果日志，则表明调用成功。

![识别结果日志示例](assets/posts-img/004-lua-asr-tts.png)

### 4. 配置简单的语音IVR流程

最后，通过一个简单的流程，展示搭建好的FreeSWITCH IVR环境所具有的功能。

#### 4.1 基本的流程设计

以下是基本的流程设计：

1. 电话接通后，首先提示用户“你好，请问有什么可以帮您的？”，等待输入；

2. 识别用户输入后，如果用户说的是“退出”，则提示“识别结果是退出，再见。”，挂机；如果是其他，则播报识别结果，并提示继续输入，不断循环；

3. 如果播报识别过程中调用TTS或ASR错误，则打印/播报错误信息并退出。

![Demo流程图](assets/posts-img/004-flow.png)

#### 4.2 交互流程

在具体的流程实现上，首先配置dialplan并使之生效、新建对应的Lua脚本文件，配置Lua可以使用上一步的。

在Lua脚本中，引入了一个第三方模块 `xmlSimple` ([GitHub Repo](https://github.com/Cluain/Lua-Simple-XML-Parser))，用以解析ASR识别结果XML，取出用户输入文本。

在流程执行中，使用了一个`while session:ready() do`的循环，除非用户说“退出”或出现异常，否则一直进行识别播报，展示ASR和TTS引擎调用的功能。

具体的代码参看仓库对应的脚本文件`stage3-loop.lua`。

### 5. 参考资料

- 百度呼叫中心语音解决方案：<https://ai.baidu.com/ai-doc/SPEECH/Gkarlx6k1>

- FreeSWITCH中调用合成和识别的核心方法：<https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Modules/mod-dptools/6586714/>