---
title: 如何用Github Pages创建个人博客网站
date: 2024-04-03 22:41:00 +0800
categories: [Github_Pages]
tags: [web, tutorial]
---

本文介绍使用**Github Pages**并选择Jekyll主题**Chirpy**，快速搭建个人博客网站。

### 1. 引言

用 Github Pages 和 Chirpy 搭建个人博客网站，可以使用 Markdown 写博客，提交后网站即更新，不必关注其他技术细节。

Github Pages [官网](https://pages.github.com/) 介绍：

> GitHub Pages 是一项静态站点托管服务，它直接从 GitHub 上的仓库获取 HTML、CSS 和 JavaScript 文件，（可选）通过构建过程运行文件，然后发布网站。

Chirpy [示例网站](https://chirpy.cotes.page/) 介绍：

> 用于技术写作的最小化、响应式和功能丰富的Jekyll主题。（A minimal, responsive, and feature-rich Jekyll theme for technical writing.）

![Chirpy示例网站](assets/posts-img/001-chirpy-default.png)

### 2. 创建仓库

> 备注：需要有 GitHub 账号 [注册/登录地址](https://github.com)

- 根据模板创建

打开网页 `https://github.com/cotes2020/chirpy-starter`

点击 `Use this template`

选择 `Create a new repository`

输入仓库名，格式为 `username.github.io`，点击 `Create repository` 创建仓库

- 预览网站

进入仓库，顶部进入 `Settings`，左侧选中 `Pages`，点击 `Visit site` 打开页面预览

### 3. 本地调试

> 备注：需要本机安装 Git [下载安装地址](https://git-scm.com/)

- 克隆仓库到本地

```sh
git clone https://github.com/username/username.github.io.git
```

- 安装环境依赖

安装Ruby、Jekyll等依赖

```sh
brew install ruby ruby-gem
gem install jekyll bundler
```

> 备注：不同系统有不同的的安装方式 [安装说明](https://jekyllrb.com/docs/installation/)

> `brew/gem/bundler` 可设置速度更快的国内源，具体方法可使用搜索引擎

- 本地运行

进入项目目录，安装项目依赖然后运行

```sh
bundle install
bundle exec jekyll serve
```

打开 `http://127.0.0.1:4000/` 预览网站效果

### 4. 修改默认配置

修改 `_config.yml` 里的默认配置可改变网站显示，主要包括：

``` yml
lang: zh-CN    # 设置网站显示为中文
timezone: Asia/Shanghai    # 设置时区为东八区

title: User's Blog   # 设置侧边栏 LOGO 下显示的博客标题
url: https://username.github.io # 设置 Github Pages 地址，其他如 username/name/email，设置Github账号信息

avatar: 'assets/img/avatar.jpeg'  # 设置侧边栏的 LOGO 图标地址，可以是本地的也可以是外部的
```

> 修改配置需重启服务生效

### 5. 写一篇文章

- 添加文章

在 `_posts` 下创建文档，文件名为 `YYYY-MM-DD-标题.md`，即可新建文章

使用 Markdown 编写文章内容

- 设置文章信息

若要指定文章名、发布时间、为文章添加分类和标签等，则在文章的顶部加入以下内容：

``` markdown
---
title: 文章的标题
date: 2024-01-01 12:00:00 +0800
categories: [一级分类, 二级分类]
tags: [标签1, 标签2]
---
```

分类最多两级，标签数量无限制，设置后可通过侧边栏进入筛选

### 6. 部署生效

- 提交修改

使用 `Git` 仓库提交和推送
``` sh
git add .
git commit -m "add the first post"
git push
```

稍等一会即可看到网页更新

![网站预览](assets/posts-img/001-chirpy-one-blog.png)


### 7. 参考资料

- 模板方案建议：[https://zhuanlan.zhihu.com/p/641525444](https://zhuanlan.zhihu.com/p/641525444)

- 使用介绍：[https://chirpy.cotes.page/](https://chirpy.cotes.page/)

- GitHub Pages 中文文档：[https://docs.github.com/zh/pages](https://docs.github.com/zh/pages)
