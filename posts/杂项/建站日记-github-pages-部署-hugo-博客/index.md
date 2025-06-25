# 建站日记-Github Pages 部署 Hugo 博客

## Hugo 博客配置与内容创建

本章节将详细介绍如何从零开始配置一个 Hugo 博客，包括环境准备、站点初始化、主题安装与配置，以及如何创建基本内容。

### 准备 Hugo 环境

你可以选择在本地直接安装 Hugo，或者使用 Docker 容器化环境。对于追求环境隔离和一致性的开发者，我们推荐使用 Docker。

> [!TIP]
> 使用 Docker 的好处在于可以避免在本地系统中安装各种依赖，保持系统整洁。同时，`hugomods/hugo` 镜像已经内置了 `Sass/SCSS` 和 `PostCSS` 等扩展，开箱即用，非常方便。

首先，拉取推荐的 Hugo 镜像。镜像的标签含义可以参考[官方文档](https://docker.hugomods.com/docs/tags/)。

```bash
docker pull hugomods/hugo:debian-exts-0.147.7
```

### 初始化博客项目

环境准备好后，我们来创建 Hugo 站点。如果你使用 Docker，可以先启动一个临时容器来执行初始化命令。

```bash
# 进入你的项目根目录，例如 /mnt/nanopc/Repo
# docker run -it --rm --name hugo -v $(pwd):/src hugomods/hugo:debian-exts-0.147.7 bash
# 为了方便 Windows 用户，这里使用绝对路径
docker run -it --rm --name hugo -v /mnt/nanopc/Repo:/src hugomods/hugo:debian-exts-0.147.7 bash

# 在容器内执行以下命令
# --force 可以在已存在的非空目录中初始化
# --format yaml 指定使用 YAML 格式的配置文件，这是目前推荐的格式
hugo new site blogs --force --format yaml
```

> [!TIP]
> 仅初始化需要在仓库的上级路径执行指令，所以在后续调整 Hugo 配置及内容时，可以使用以下指令直接启动开发服务器 
> `docker run -itd --network host  --rm --name hugo -v /mnt/nanopc/Repo/HugoBlogs:/src hugomods/hugo:debian-exts-0.147.7 hugo serve --bind 0.0.0.0  --baseURL http://192.168.2.20`

### 安装并选择主题

一个美观的主题能让你的博客更具吸引力。Hugo 拥有丰富的主题生态，这里以 DoIt 主题为例进行说明。我们推荐使用 Git Submodule 的方式来安装主题，这样可以方便地对主题进行版本控制和更新。

```bash
cd blogs
# 使用 PaperMod 主题
# git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
# 使用 DoIt 主题
git submodule add https://github.com/HEIGE-PCloud/DoIt.git themes/DoIt
# 如果你是第一次克隆包含子模块的仓库，或需要更新子模块，请执行以下命令
git submodule update --init --recursive
```

> [!TIP]
> 另一个广受欢迎的主题是 [PaperMod](https://github.com/adityatelange/hugo-PaperMod.git)，你也可以尝试使用它：
> `git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod`
> 安装主题后，别忘了在 `hugo.yaml` 中启用它。

### 配置 `hugo.yaml`

`hugo.yaml` 是 Hugo 站点的核心配置文件。你需要在这里设置站点的元数据、菜单、主题等。

用下面的内容覆盖你的 `hugo.yaml` 文件，并根据你的实际情况进行修改。

>[!WARNING]
> **`baseURL` 必须以 `/` 结尾！**
> 这是一个非常常见的错误。如果 `baseURL` (例如 `https://your-domain.com`) 后面没有斜杠，可能会导致 CSS、JS 等静态资源加载失败（404错误），因为 Hugo 在拼接 URL 时会出错，产生类似 `https://your-domain.comyour-domain.com/css/style.css` 这样的无效地址。

```yaml
# Basic site configuration
baseURL: https://www.65811786.xyz/  # Website URL
title: 小饭的技术站  # Site title
theme: DoIt  # Theme name
languageCode: zh-CN  # Primary language code

# Chinese language and other global settings
hasCJKLanguage: true  # Enable Chinese/Japanese/Korean language support
enableEmoji: true  # Enable emoji shortcodes
enableRobotsTXT: true  # Generate robots.txt file
# DoIt theme version
version: "0.4.X"  # DoIt theme version
# Public git repo url (only effective when enableGitInfo is true)
gitRepo: ""  # Public git repository URL
# Which hash function for SRI
fingerprint: ""  # Hash function for SRI ("sha256", "sha384", "sha512", "md5")
# Website images for Open Graph and Twitter Cards
images: []  # Website images for social sharing
# Enable PWA support
enablePWA: false  # Enable Progressive Web App support
# License information
license: '<a rel="license external nofollow noopener noreferrer" href="https://creativecommons.org/licenses/by-nc/4.0/" target="_blank">CC BY-NC 4.0</a>'  # License information
# Experimental bundle JS
bundle: false  # Bundle JavaScript files

# Content organization
taxonomies:
  category: categories  # Category taxonomy
  tag: tags  # Tag taxonomy
  series: series  # Series taxonomy

# Output formats
outputs:
  home:
    - HTML  # HTML output
    - RSS  # RSS feed
    - JSON  # JSON output for search

# Language settings
languages:
  zh:
    languageName: "Chinese"  # Language display name
    weight: 1  # Priority weight
    menu:
      main:
        - identifier: posts
          name: 文章  # Posts menu item
          url: /posts
          weight: 2
        - identifier: categories
          name: 分类  # Categories menu item
          url: categories
          weight: 4
        - identifier: tags
          name: 标签  # Tags menu item
          url: /tags
          weight: 5

# Site parameters
params:
  author:
    name: "小饭"  # Author name
    email: ""  # Author email
    link: "https://www.65811786.xyz/"  # Author link
    avatar: ""  # Author avatar
    gravatarEmail: ""  # Gravatar email
    
  # Site basic settings
  defaultTheme: auto  # Theme mode (auto, light, dark, black)
  DateFormat: "2006-01-02"  # Date format
  DisplayToc: true  # Display table of contents
  
  # Image processing settings
  image:
    cacheRemote: true  # Cache remote images
    optimise: true  # Optimize images
  
  label:
    text: "小饭技术站"  # Site label text
    icon: "img/logo.jpg"  # Site label icon
    iconHeight: 35  # Icon height
    
  assets:
    favicon: "img/logo.jpg"  # Site favicon
    
  # Header configuration
  header:
    themeChangeMode: "select"  # Theme change mode ("switch", "select")
    title:
      logo: ""  # Logo URL
      name: "小饭技术站"  # Title name
      pre: ""  # Content before title
      post: ""  # Content after title
      typeit: false  # Whether to use typeit animation
      
  # Footer configuration
  footer:
    enable: true  # Enable footer
    custom: ""  # Custom content
    hugo: true  # Show Hugo info
    hostedOn: ""  # Hosting info
    copyright: true  # Show copyright
    author: true  # Show author
    since: 2025  # Site creation year
    icp: ""  # ICP filing info (China)
    license: '<a rel="license external nofollow noopener noreferrer" href="https://creativecommons.org/licenses/by-nc/4.0/" target="_blank">CC BY-NC 4.0</a>'  # License info
    
  # Section page config (all posts)
  section:
    paginate: 20  # Posts per page
    dateFormat: "01-02"  # Date format (month and day)
    rss: 10  # RSS posts count
    recentlyUpdated:
      enable: true  # Enable recently updated posts
      rss: true  # Include in RSS
      days: 30  # Days count as recent
      maxCount: 10  # Max posts to show
      
  # List page config (category/tag)
  list:
    paginate: 20  # Posts per page
    dateFormat: "01-02"  # Date format (month and day)
    rss: 10  # RSS posts count
    
  # Page configuration
  page:
    hiddenFromHomePage: false  # Hide from home page
    hiddenFromSearch: false  # Hide from search
    twemoji: false  # Use twemoji
    lightgallery: false  # Enable lightgallery
    ruby: true  # Enable ruby syntax
    fraction: true  # Enable fraction syntax
    linkToMarkdown: true  # Link to raw markdown
    linkToSource: ""  # Link to source
    linkToEdit: ""  # Link to edit page
    linkToReport: ""  # Link to report issues
    rssFullText: false  # Full text in RSS
    seriesNavigation: true  # Series navigation
    enableLastMod: true  # Show last modified time
    enableWordCount: true  # Show word count
    enableReadingTime: true  # Show reading time
    
    # Outdated article reminder
    outdatedArticleReminder:
      enable: false  # Enable reminder
      reminder: 90  # Days to show reminder
      warning: 180  # Days to show warning
      
    # InstantPage preloading
    instantpage:
      enable: true  # Enable InstantPage
      
    # Table of contents
    toc:
      enable: true  # Enable ToC
      keepStatic: false  # Keep static ToC
      auto: false  # Auto collapse
      
    # Code display
    code:
      maxShownLines: 10  # Max lines to show
      lineNos: true  # Show line numbers
      wrap: false  # Enable line wrapping
      header: true  # Show header
      
    # Table settings
    table:
      sort: true  # Enable sorting
      
    # Header settings
    header:
      number:
        enable: true  # Enable header numbering
        format:
          h2: "{h2} {title}"
          h3: "{h2}.{h3} {title}"
          h4: "{h2}.{h3}.{h4} {title}"
          h5: "{h2}.{h3}.{h4}.{h5} {title}"
          h6: "{h2}.{h3}.{h4}.{h5}.{h6} {title}"
          
    # Math expressions
    math:
      enable: false  # Enable math
      blockLeftDelimiter: ""  # Block left delimiter
      blockRightDelimiter: ""  # Block right delimiter
      inlineLeftDelimiter: ""  # Inline left delimiter
      inlineRightDelimiter: ""  # Inline right delimiter
      copyTex: true  # Copy TeX
      mhchem: true  # mhchem extension
      mathjax: false  # Use MathJax
      
    # MapBox GL JS config
    mapbox:
      accessToken: ""  # Access token
      lightStyle: "mapbox://styles/mapbox/light-v10?optimize=true"  # Light theme style
      darkStyle: "mapbox://styles/mapbox/dark-v10?optimize=true"  # Dark theme style
      navigation: true  # Show navigation control
      geolocate: true  # Show geolocation control
      scale: true  # Show scale control
      fullscreen: true  # Show fullscreen control
      
    # Social share links
    share:
      enable: true  # Enable sharing
      Twitter: true  # Twitter sharing
      Facebook: true  # Facebook sharing
      Linkedin: true  # LinkedIn sharing
      Weibo: true  # Weibo sharing
      Telegram: true  # Telegram sharing
      
    # Comment systems
    comment:
      enable: false  # Enable comments
      
    # PlantUML diagram
    plantuml:
      server: "https://www.plantuml.com/plantuml"  # PlantUML server
      
  # TypeIt config
  typeit:
    speed: 100  # Typing speed
    cursorSpeed: 1000  # Cursor blinking speed
    cursorChar: "|"  # Cursor character
    duration: -1  # Duration after typing
    
  # Site verification
  verification:
    google: ""  # Google verification
    bing: ""  # Bing verification
    baidu: ""  # Baidu verification
    
  # Analytics configuration
  analytics:
    enable: false  # Enable analytics
    
  # Cookie consent
  cookieconsent:
    enable: false  # Enable cookie consent

# Markup related configuration in Hugo
markup:
  # Syntax Highlighting (https://gohugo.io/content-management/syntax-highlighting)
  highlight:
    codeFences: true  # Enable code fences
    guessSyntax: true  # Auto detect language
    lineNos: true  # Show line numbers
    lineNumbersInTable: true  # Show line numbers in table
    noClasses: false  # False is necessary (https://github.com/dillonzq/LoveIt/issues/158)
    style: darcula  # Highlight style theme
  
  # Goldmark is Hugo's default Markdown processor since v0.60
  goldmark:
    extensions:
      definitionList: true  # Enable definition lists
      footnote: true  # Enable footnotes
      linkify: true  # Auto-convert plain text links
      strikethrough: true  # Enable strikethrough
      table: true  # Enable tables
      taskList: true  # Enable task lists
      typographer: true  # Enable typographic replacements
    renderer:
      unsafe: true  # Allow HTML tags in markdown
    extensions.passthrough:
      enable: true
    extensions.passthrough.delimiters:
      block: 
        - ['\[', '\]']
        - ['$$', '$$']
      inline: 
        - ['\(', '\)']
  
  # Table Of Contents settings
  tableOfContents:
    startLevel: 2  # Start TOC at heading level 2
    endLevel: 6  # End TOC at heading level 6

```

> [!NOTE]
> 上述配置只是一个基础示例。不同的主题（如 DoIt 或 PaperMod）通常有自己专属的参数，用于实现更丰富的功能。请务必查阅你所使用主题的官方文档，以了解所有可用的配置项。

### 创建基本内容结构

配置完成后，我们需要为博客创建一些基本的页面结构，例如文章列表页、分类页和标签页。这些页面通常通过在 `content` 目录下创建 `_index.md` 文件来定义。

- **主页 (`content/_index.md`)**: 定义了网站首页的布局和内容。
- **文章列表页 (`content/posts/_index.md`)**: 作为所有文章的入口。
- **分类和标签页 (`content/categories/_index.md`, `content/tags/_index.md`)**: 用于展示所有的分类和标签。

以下是这些文件的基本内容模板：

<details>
<summary>点击展开查看页面模板</summary>

- 主页说明 `content/_index.md`
```markdown
---
title: "主页"
layout: "home"
# home layout for PaperMod theme
---
```

- 分类页 `content/categories/_index.md`
```markdown
---
title: "所有分类"
layout: "categories"
---
```

- 标签页 `content/tags/_index.md`
```markdown
---
title: "所有标签"
layout: "tags"
---
```

- 文章列表页 `content/posts/_index.md`
```markdown
---
title: "所有文章"
layout: "posts"
summary: "posts"
---

在这里，你可以找到我发布过的所有文章。
```

</details>

### 创建你的第一篇文章

万事俱备，现在来创建你的第一篇文章吧！你可以手动在 `content/posts/` 目录下创建 Markdown 文件，但更推荐使用 Hugo 的命令来生成，这样可以自动带上预设的 Front Matter。

```bash
hugo new posts/my-first-post.md
```

这会在 `content/posts/` 目录下创建一个 `my-first-post.md` 文件，内容如下：

```markdown
---
title: "My First Post"
date: 2024-07-29T15:00:00+08:00
draft: true
---
```

> [!TIP]
> - `draft: true` 表示这是一篇草稿，默认情况下不会被构建到最终的网站中。当你完成写作后，可以将其改为 `false` 或直接删除该行。
> - 你可以通过修改 `archetypes/default.md` 文件来定制新建文章的默认 Front Matter 模板。

### 启动本地开发服务器

在完成了以上所有配置后，就可以启动 Hugo 内置的开发服务器来实时预览你的网站了。

```bash
# --bind 0.0.0.0 允许其他设备通过 IP 访问
# --baseURL 用于在本地预览时正确解析链接
# -D 或 --buildDrafts 参数会连同草稿一起构建
hugo serve --bind 0.0.0.0 --baseURL http://192.168.2.20 -D
```

服务启动后，在浏览器中访问 `http://localhost:1313` 或 `http://192.168.2.20:1313` 即可看到你的博客。并且，当你修改任何文件并保存后，页面都会自动刷新，非常方便。

> [!TIP]
> 如果你使用 Docker，可以运行以下命令来启动开发服务器，它会将容器的1313端口映射到宿主机：
> `docker run -itd --network host  --rm --name hugo -v $(pwd):/src hugomods/hugo:debian-exts-0.147.7 hugo serve --bind 0.0.0.0  --baseURL http://192.168.2.20`
> 注意：此处的 `$(pwd)` 需要指向你的博客项目根目录。

## GitHub 配置：部署你的博客网站

在本地完成 Hugo 的配置和内容创作后，下一步就是将它部署到互联网上，让所有人都能访问。本章将指导你如何使用 GitHub Pages 完成这一过程。

我们将采用一种常见且高效的策略：
- **一个私有仓库**：用于存储你的博客源文件（Markdown 文章、Hugo 配置文件、主题等）。
- **一个公开仓库**：用于托管由 Hugo 生成的静态网站文件（HTML, CSS, JS），并作为 GitHub Pages 的发布源。

这种方式能很好地将你的创作内容和最终发布的网站分离开来。

### 步骤一：创建 GitHub 仓库

首先，你需要在 GitHub 上创建两个仓库。

#### 1. 创建博客源码仓库 (私有)

这个仓库用来存放你的整个 Hugo 项目。
- **Repository name**: 可以任意命名，例如 `my-hugo-blog`。
- **Visibility**: 强烈建议设置为 **Private**，因为这里可能包含一些草稿或未完成的内容。

#### 2. 创建 GitHub Pages 仓库 (公开)

这个仓库用于托管最终生成的网站，并对外提供访问。

> [!WARNING]
> 仓库的命名和设置至关重要，请严格按照以下规则操作：
> - **Repository name**: 必须是 `你的GitHub用户名.github.io` 的格式。例如，如果你的用户名是 `octocat`，那么仓库名就是 `octocat.github.io`。
> - **Username case**: `你的GitHub用户名` 必须是全小写。
> - **Visibility**: 必须设置为 **Public**。

创建时，可以选择 "Add a README file" 来初始化仓库。

### 步骤二：配置自动部署密钥

为了实现从私有源码仓库自动部署到公开 Pages 仓库，我们需要设置一对 SSH 密钥，让两个仓库之间可以安全地"通信"。

#### 1. 生成 SSH 密钥对

在你的本地电脑上，打开终端，执行以下命令来生成一对新的 SSH 密钥。

```bash
# -f 参数指定了密钥文件的名称，建议使用一个有辨识度的名字
# -N "" 表示不设置密码，方便自动化脚本使用
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f hugo-deploy-key -N ""
```

执行后，当前目录下会生成两个文件：
- `hugo-deploy-key` (私钥)
- `hugo-deploy-key.pub` (公钥)

#### 2. 在源码仓库中配置私钥

私钥需要作为秘密（Secret）添加到你的 **私有源码仓库** 中，以便在自动化流程（如 GitHub Actions）中使用它来推送文件。

1.  访问你的私有源码仓库，进入 `Settings > Secrets and variables > Actions`。
2.  点击 `New repository secret`。
3.  **Name**: 必须填写为 `ACTIONS_DEPLOY_KEY` (或者你自定义的名称，但需要和后续 CI/CD 脚本中的名称保持一致)。
4.  **Secret**: 将 **私钥** 文件 (`hugo-deploy-key`) 的 **全部内容** 复制并粘贴进去。
5.  点击 `Add secret` 保存。

> [!TIP]
>  你可以直接访问 `https://github.com/<你的用户名>/<源码仓库名>/settings/secrets/actions` 来快速进入配置页面。

#### 3. 在 Pages 仓库中配置公钥

公钥需要作为部署密钥（Deploy Key）添加到你的 **公开 Pages 仓库** 中，以授予源码仓库向其推送更改的权限。

1.  访问你的公开 Pages 仓库 (`<用户名>.github.io`)，进入 `Settings > Deploy keys`。
2.  点击 `Add deploy key`。
3.  **Title**: 可以任意填写，例如 `Hugo Deploy Key`。
4.  **Key**: 将 **公钥** 文件 (`hugo-deploy-key.pub`) 的 **全部内容** 复制并粘贴进去。
5.  **关键一步**: 勾选 `Allow write access` 复选框，授予其写入权限。
6.  点击 `Add key` 保存。

> [!TIP]
> 你可以直接访问 `https://github.com/<你的用户名>/<你的用户名>.github.io/settings/keys` 来快速进入配置页面。

### 步骤三：配置自定义域名 (可选)

虽然你可以直接通过 `xxx.github.io` 访问你的博客，但使用自定义域名会更专业。

#### 1. DNS 解析配置

前往你的域名提供商（如 Cloudflare, GoDaddy）的 DNS 管理后台，添加以下解析记录。

```
# 将 www 子域名指向你的 GitHub Pages 地址
CNAME   www.your-domain.com    your-username.github.io.
# 将根域名指向 GitHub Pages 的服务器 IP 地址
A       your-domain.com        185.199.108.153
A       your-domain.com        185.199.109.153
A       your-domain.com        185.199.110.153
A       your-domain.com        185.199.111.153
```
> [!NOTE]
> GitHub Pages 的 IP 地址可能会变更，你可以查阅 [GitHub 官方文档](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site) 获取最新的 IP 地址列表。

#### 2. 在 GitHub Pages 中配置域名

1.  访问你的公开 Pages 仓库设置页面 `Settings > Pages`。
2.  在 `Custom domain` 部分，输入你的自定义域名 (例如 `www.your-domain.com`) 并保存。
3.  等待 DNS 检查通过后，勾选 `Enforce HTTPS` 选项，为你的网站启用 SSL 加密。

#### 3. 在 Hugo 项目中创建 CNAME 文件

这是防止自定义域名设置在每次自动部署后被重置的关键一步。

在 Hugo 项目的 `static` 目录下创建一个名为 `CNAME` 的文件，文件内容只有一行，即你的自定义域名。

`static/CNAME`:
```text
www.your-domain.com
```

> [!WARNING]
> 如果没有这个 `CNAME` 文件，每当 GitHub Actions 部署你的网站时，它会覆盖 Pages 仓库的所有文件，导致你在 GitHub 网页上设置的自定义域名信息丢失，域名会自动回退到 `xxx.github.io`。

## 附录：高级环境配置

### 在 Incus/LXD 容器中准备 Docker 环境

本附录为希望在隔离的 Linux 容器环境中进行开发的进阶用户提供指导。我们将介绍如何在 [Incus](https://linuxcontainers.org/incus/) (LXD 的一个分支) 容器中安装并配置 Docker，从而为 Hugo 创建一个完全封装的开发环境。

> [!NOTE]
> 这是一种高级部署策略。对于大多数用户而言，直接在你的操作系统（如 Windows、macOS 或 Linux）上安装 Docker Desktop 或 Docker Engine 会更简单直接。如果你不熟悉 Incus/LXD，可以跳过本节。

#### 步骤一：启动并进入 Incus 容器

首先，我们需要一个支持嵌套虚拟化和特权模式的 Debian 容器。

```bash
# 列出可用的 Debian 镜像
incus image list images:debian

# 启动一个 Debian 12 容器
# -c security.privileged=true: 以特权模式运行容器
# -c security.nesting=true: 允许在容器内再运行容器（比如 Docker）
incus launch images:debian/12 debian-hugo-env -c security.privileged=true -c security.nesting=true

# 进入容器的 shell 环境
incus exec debian-hugo-env bash
```
> [!TIP]
> `debian-hugo-env` 是我们为容器起的名字，你可以替换为任何你喜欢的名称。

#### 步骤二：安装基础依赖

进入容器后，首先更新软件包列表并安装网络管理和下载工具。

```bash
apt update
apt install -y netplan.io curl
```

#### 步骤三：配置静态网络 (可选)

如果你需要为容器设置一个固定的局域网 IP 地址以便于访问，可以配置 `netplan`。

> [!WARNING]
> 以下网络配置仅为示例，你需要根据你自己的局域网设置（如网段、网关地址）进行修改。如果不需要固定 IP，可以跳过此步骤，容器默认会通过 DHCP 获取 IP。

```bash
# 创建 netplan 配置文件
cat > /etc/netplan/01-netcfg.yaml <<-EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.2.20/24
      routes:
        - to: default
          via: 192.168.2.10
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
EOF
chmod 600 /etc/netplan/01-netcfg.yaml
netplan try
```

#### 步骤四：安装 Docker Engine

最后，我们使用 Docker 官方的便捷脚本来安装 Docker Engine。

```bash
# 下载安装脚本
curl -fsSL https://get.docker.com -o get-docker.sh

# 使用国内镜像源运行脚本以加速下载
sh get-docker.sh --mirror Aliyun
```

安装完成后，你就在 Incus 容器中有了一个可以正常运行的 Docker 环境。现在，你可以按照正文中的指导，使用 `docker pull` 和 `docker run` 命令来部署和管理你的 Hugo 容器了。

## 参考资料
- [Windows下使用hugo和Github Pages配置博客](https://www.haoyep.com/posts/windows-hugo-blog-github)
- [PaperMod主题配置](https://www.shaohanyun.top/posts/env/blog_build2/)
- [Github Pages 配置和 Deploy Keys 使用](https://github.com/peaceiris/actions-gh-pages?tab=readme-ov-file#%EF%B8%8F-create-ssh-deploy-key)
- [自定义域名解析配置方法](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site)
- [DoIt 文档](https://hugodoit.pages.dev/zh-cn/categories/documentation/)
