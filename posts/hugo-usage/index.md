# Hugo 使用方法

> 折腾过很多种blog,都半途而废了,借助于AI,搭建hugo

使用 GitHub 配合 Hugo 部署个人博客是一个高效且免费的方案，主要利用 GitHub Pages 托管静态文件。以下是详细步骤：
## 一、准备工作
1. 安装工具
  ○ 安装 Hugo（建议安装扩展版 hugo_extended，支持更多主题功能）
  ○ 安装 Git
  ○ 注册 GitHub 账号
## 二、创建 Hugo 博客项目
1. 初始化博客
打开终端，执行以下命令创建 Hugo 项目（将 my-blog 替换为你的博客名称）：
```bash
hugo new site my-blog
cd my-blog
```

2. 添加主题
选择一个 Hugo 主题（推荐从 Hugo Themes 挑选），以经典主题 LoveIt 为例：
```bash
git init  # 初始化 Git 仓库
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt #（添加主题为子模块，方便后续更新）
```

3. 配置主题
```bash
cp themes/LoveIt/exampleSite/config.toml .
```
编辑 config.toml，修改博客名称、作者、语言等基础信息。

4. 创建第一篇文章bash
```bash
hugo new posts/hello-hugo.md
```
打开 content/posts/hello-hugo.md，用 Markdown 编写内容（注意删除头部 draft: true 才会发布）。
5. 本地预览
启动 Hugo 本地服务器，实时查看效果：
```bash
hugo server -D  # -D 表示包含草稿文章
```
访问 http://localhost:1313 即可预览博客。
## 三、通过 GitHub Pages 部署
GitHub Pages 支持两种部署方式，推荐使用 第二种（更灵活）：
方式 1：直接部署 public 目录（简单但功能有限）
1. 构建静态文件
生成可部署的静态文件到 public 目录：
```bash
hugo  # 不带 -D，只生成非草稿内容
```

2. 创建 GitHub 仓库
在 GitHub 新建仓库，命名为 <你的用户名>.github.io（必须严格匹配，例如 octocat.github.io），这是 GitHub Pages 的特殊仓库名，用于托管个人主页。
3. 推送 public 目录到仓库
```bash
cd public
git init
git add .
git commit -m "First deploy"
git branch -M main
git remote add origin https://github.com/<你的用户名>/<你的用户名>.github.io.git
git push -u origin main
```

几分钟后，访问 https://<你的用户名>.github.io 即可看到博客。
方式 2：使用 GitHub Actions 自动部署（推荐）
通过 GitHub Actions，可实现 “推送代码后自动构建并部署”，更适合持续更新。


1. 创建两个仓库
  ○ 仓库 A：命名为 my-blog-source（存储 Hugo 源码，私有 / 公开均可）
  ○ 仓库 B：命名为 <你的用户名>.github.io（存储最终静态文件，必须公开）
2. 推送 Hugo 源码到仓库 Abash
## 四、在博客根目录（my-blog）执行
```bash
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/<你的用户名>/my-blog-source.git
git push -u origin main
```
3. 配置 GitHub Actions
在仓库 A 中创建工作流文件：
```bash
mkdir -p .github/workflows
touch .github/workflows/deploy.yml
```


编辑 deploy.yml，粘贴以下内容（替换 <你的用户名> 为实际 GitHub 用户名）：
```yaml
name: Deploy Hugo Site to GitHub Pages
on:
  push:
    branches: [ main ]  # 源码仓库的主分支

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true  # 拉取主题子模块
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest' #可以指定版本 如 0.128.0
          extended: true  # 使用扩展版

      - name: Build
        run: hugo --minify  # 构建静态文件

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}  # 部署密钥
          external_repository: <你的用户名>/<你的用户名>.github.io  # 仓库 B
          publish_branch: main  # 仓库 B 的部署分支
          publish_dir: ./public  # 构建结果目录
```

4. 配置部署密钥
  ○ 本地生成 SSH 密钥（一路回车，不要设置密码）
  ○ 复制公钥（hugo-deploy-key.pub 内容）到仓库 B 的 Settings → Deploy keys，勾选 Allow write access。
  ○ 复制私钥（hugo-deploy-key 内容）到仓库 A 的 Settings → Secrets → Actions，新建密钥 ACTIONS_DEPLOY_KEY，粘贴私钥。
5. 完成部署
推送代码到仓库 A 的 main 分支，GitHub Actions 会自动运行构建并部署到仓库 B。几分钟后，访问 https://<你的用户名>.github.io 即可看到博客。
## 五、后续维护
- 发布新文章：用 hugo new posts/新文章.md 创建并编辑，推送源码到仓库 A 后自动部署。
- 更新主题：执行 git submodule update --remote --merge 更新主题，再推送代码。
- 自定义域名：在仓库 B 的 Settings → Pages → Custom domain 中配置自己的域名（需提前解析 DNS）。

通过这种方式，你可以专注于内容创作，GitHub 会自动处理构建和部署，非常适合个人博客的长期维护。


换电脑后，只需重新配置开发环境并同步 Hugo 博客的源代码，就能继续维护博客。以下是详细步骤：
## 六、在新电脑上安装必要工具
1. 安装 Git
下载并安装 Git，确保终端中能运行 git --version 验证安装成功。
2. 安装 Hugo
根据新电脑的系统（Windows/macOS/Linux），从 Hugo 官网 下载对应版本（推荐 hugo_extended 扩展版），安装后通过 hugo version 验证。
## 七、同步博客源代码
如果之前已将 Hugo 源码（非 public 目录）推送到 GitHub 仓库（如前文的 my-blog-source），直接克隆仓库即可：
1. 克隆代码到新电脑
打开终端，进入你想存放博客的目录，执行：
```bash
git clone https://github.com/<你的用户名>/my-blog-source.git
cd my-blog-source  # 进入博客目录
```

2. 拉取主题子模块
由于主题是作为 Git 子模块存在的，克隆后需要单独拉取主题文件：
```bash
git submodule update --init --recursive
```
（这一步会下载主题到 themes 目录，确保本地能正常预览）
七、验证本地环境
1. 预览博客
运行本地服务器，确认博客能正常显示：
```bash
hugo server -D
```
访问 http://localhost:1313，检查文章、主题样式是否正常。
2. 测试发布流程
可以尝试创建一篇测试文章，推送代码到 GitHub，验证 GitHub Actions 是否能自动部署（如果用了自动部署方案）：
```bash
hugo new posts/test-on-new-pc.md  # 创建文章
# 编辑文章后，删除 draft: true
git add .
git commit -m "Test on new PC"
git push origin main
```
## 八、补充说明
-  如果之前没备份源码：
若只推送了 public 目录到 GitHub（方式 1），则需要从旧电脑拷贝 Hugo 源码（整个博客目录，除 public 外）到新电脑，再按上述步骤初始化 Git 并推送（建议后续改用双仓库 + Actions 的方式管理）。
-  密钥 / 权限问题：
如果推送代码时出现权限错误，可能是新电脑未配置 GitHub 账号：
  a. 终端中运行 git config --global user.name "你的用户名" 和 git config --global user.email "你的邮箱"
  b. 若用 SSH 地址克隆，需在新电脑生成 SSH 密钥并添加到 GitHub（ Settings → SSH and GPG keys ）。


通过以上步骤，新电脑就能完全复用之前的博客环境，后续写作、更新主题等操作和原来完全一致。核心是确保源码通过 Git 备份，换设备时只需克隆代码 + 安装依赖即可。

