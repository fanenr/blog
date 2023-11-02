---
title: Hexo Config
date: 2023-06-22 19:48:33
updated: 2023-06-23 19:00:00
tags:
  - Hexo
  - Blog
categories:
  - Hexo
---

&emsp;&emsp;该说不说的，Hexo 配置还真是多，之前用语雀就是无脑写。

<!-- more -->

## 站点配置

&emsp;&emsp;这一级的配置文件位于`_config.yml`，配置文件采用 Yaml 语法描述。我只截取我修改的部分来说明，大部分情况下，配置还有很多可选项，自己可以按需调整。

### 基本信息

&emsp;&emsp;这里主要是配置站点信息，站点名称，语言等等信息。Next 主题很好的一点是能自动读取这里的信息来生成网页，Fluid 还需要自己手动修改主题配置中的基本信息来使其生效。

``` yaml
# Site
title: Arthur's blog
author: Arthur
language: zh-CN
```

### 部署信息

&emsp;&emsp;很重要的一点是配置部署信息即同步到网络上，我选择部署到个人 GitHub Pages 中。所以只介绍配置到 GitHub 中的信息。

``` yaml
# Deployment
deploy:
  type: git
  repo: git@github.com:fanenr/fanenr.github.io.git
  branch: master
  token: ghp_******
```

&emsp;&emsp;这里需要修改的只有两部分：`repo`修改为个人仓库地址，我建议是 ssh 地址。`token`修改为个人的 token。如果不知道这两条信息，建议先行 Google 教程并创建仓库和 token。

### 其他配置

&emsp;&emsp;这里再记录下我的其他两个配置信息，第一是创建文档时自动创建资源文件夹，后续在文章中插入图片会方便很多。

``` yaml
post_asset_folder: true
```

&emsp;&emsp;还有一点就是 Next 主题的本地搜索也要在此配置，原本的配置文件中并未包含，所以直接将信息拷贝到此即可。

``` yaml
# Search
search:
  path: search.json
  field: post
  format: html
  limit: 10000
```

### Markdown-it

&emsp;&emsp;开始以为自带的 Markdown 渲染器肯定够用了，谁知道这么多问题，一是无法解析`&emsp`符号，二是扩展麻烦。

&emsp;&emsp;现在选择切换到`hexo-renderer-markdown-it`插件，这个插件不仅没有上面第一个问题，渲染速度也快了不少，并且扩展起来也方便很多。

#### 安装

1. 先卸载`hexo-renderer-marked`插件

```bash
# cd Hexo-Blog
npm uninstall hexo-renderer-marked --save
```

2. 安装`hexo-renderer-markdown-it`

```bash
npm install hexo-renderer-markdown-it --save
```

#### 配置

&emsp;&emsp;Markdown-it 的配置位于站点配置文件中，直接在空白处插入配置内容即可。

&emsp;&emsp;我这里只开了三个内置插件，用来使用 Markdown 扩展语法：2^x^，d~y~ 和脚注。

```yaml
# file: _config.yml

# Markdown-it
markdown:
  plugins:
  - markdown-it-sub
  - markdown-it-sup
  - markdown-it-footnote
```

## 主题配置

&emsp;&emsp;这一级的配置文件位于`_config.next.yml`，如果使用其他主题，那么对应的文件应该是`_config.theme.yml`。在新版本的 Hexo 中这个文件应该从主题包中拷贝并且改名而来。为了使本级配置生效，确保在站点配置中已经切换主题。

### 切换主题

&emsp;&emsp;这里的主题是指 Next 中自带的四个子主题，切换主题很简单，只需要启用目标主题，并注释其他选项即可。并且， Next 的 Dark Mode 也位于此。

``` yaml
# Schemes
#scheme: Muse
#scheme: Mist
scheme: Pisces
#scheme: Gemini

# Dark Mode
darkmode: false
```

### 菜单设置

&emsp;&emsp;这里目的是启用首页中显示`标签`和`分类`两个菜单选项，首先在配置文件中修改相应信息，启用这两个选项。

``` yaml
menu:
  tags: /tags/ || fa fa-tags
  categories: /categories/ || fa fa-th
```

&emsp;&emsp;只修改这里还是不行的，还需要创建两个对应的页面。

``` bash
hexo new page tags
hexo new page categories
```

&emsp;&emsp;然后进入`source`文件夹修改这两个页面的元信息，这里只展示`categories.md`，tags 页面只需要修改 type 为 tags。

```yaml
---
title: categories
type: categories
comments: false
date: 2023-06-22 19:38:47
---
```

### 阅读设置

&emsp;&emsp;这里配置三个选项：文章大纲，一键回顶，阅读更多。注释已经够详细了，个人按需开启即可。

```yam
toc:
  enable: true
  number: true
  wrap: false
  max_depth: 6

back2top:
  enable: true
  sidebar: false
  scrollpercent: true
  
read_more_btn: true
```

### 代码样式

&emsp;&emsp;代码块样式的配置比较重要（与我而言），[官方仓库](https://github.com/chriskempson/tomorrow-theme)有图片预览，这里是我的配置。

```yaml
codeblock:
  highlight_theme: night
  copy_button:
    enable: true
    show_result: true
    style: default
```

### 社交链接

&emsp;&emsp;这里我只开启了个人的 GitHub 链接地址，只需要修改 Link 即可，其他社交配置可以自行百度。

``` yaml
github_banner:
  enable: true
  permalink: https://github.com/fanenr
  title: Follow me on GitHub
```

### 阅读统计

&emsp;&emsp;阅读统计的配置不难，但是需要接入额外的第三方服务。我使用的是 LeanCloud 网络上有很多教程，我不再赘述，只做一点提醒：注册国际版，并且在应用中创建`Counter`数据表。如果你需要所谓的安全统计，请再查询其他教程。如果像我一样不关心所谓的安全，只想足够简单，请将`security`关掉。

```yaml
leancloud_visitors:
  enable: true
  app_id: your_id
  app_key: your_key
  security: false
```

### 评论系统

&emsp;&emsp;评论系统不太好整，但也不是太复杂。首先还是要注册 LeanCloud 并且在应用中创建`Comment`数据表。我使用 Valine 的评论系统，因为目前 Next 主题直接支持的评论系统不算多。而且 Valine 足够简单。首先要开启评论，并且启用 Valine。然后配置 Valine。

```yaml
# Multiple Comment System Support
comments:
  style: tabs
  active: valine
  
# Valine
valine:
  enable: true
  appid: your_id
  appkey: your_key
  placeholder: Just go go
  avatar: wavatar
  visitor: false
  serverURLs: https://4sp63ovo.api.lncldglobal.com
```

&emsp;&emsp;这里记录一下容易踩坑的几点：

1. 如果博客显示有评论框，但是提交无反应，需要设置`serverURLs`字段，这个地址在 LeanCloud 的设置中可以找到。
2. 目前 Valine 的默认头像修改不生效，具体我也不知道是咋回事。
3. 不要同时启用 Valine Visitor 统计和之前开启的 LeanCloudVisitor 统计。

### 版权说明

&emsp;&emsp;只需要将`creative_commons`中的`post`字段改为 true 即可，注意 clean 后重新生成页面查看。

```yaml
creative_commons:
  license: by-nc-sa
  sidebar: false
  post: true
  language:
```
