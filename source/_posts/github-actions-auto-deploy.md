---
title: 使用 Github Actions 作为 hexo 自动部署 CI 的尝试
date: 2022-05-30 20:26:42
tags:
  - github actions
  - auto deploy
  - 自动部署
categories:
	- 记录
	- 日志
---
> 在理解了 git 工作流和 Hexo 部署工作流的特性和差异后，笔者打算尝试将目前的工作流切换到 git 工作流上

<!-- more -->

## Hexo 部署工作流
首先我们来看一看 Hexo 的部署工作流是什么。

如果你刚刚开始学习使用用 Hexo 搭建博客，那么你一定对这个工作流不陌生。这个工作流来源于 npm 包`hexo-deployer-git`。在使用这个工作流之前，你需要先把`_config.yml`中的 Deployment 填写完成，否则可能无法正常工作。
```shel
$ npm install hexo-deployer-git --save
$ hexo deploy
```
这个过程中发生了什么事情呢，笔者可以在这里给出一个大致的回答：
1. 将静态文件生成到`.deploy_git/`文件夹下
2. 切换工作目录到该文件夹下，将其初始化为新的 git 仓库
3. 将该 git 仓库推送到远端的指定分支（在`_config.yml`中指定）
4. 默认的 github pages 的 github actions 触发，将静态文件部署到对应的 `*.github.io` github pages 上

如果想要知道更多细节/配置可以参考[对应的 github 页面](https://github.com/hexojs/hexo-deployer-git)


## 更换工作流的诱因
在不考虑这个项目/博客基于 git 的情况下，上述的工作流完全足够我们日常使用。github 此时对于我们的作用只在于储存我们在本地更新生成的静态文件，以及将静态文件部署到服务器上。

如果不选择 github 本身进行备份，我们可以选用 Dropbox, OneDrive, Google Drive 等一系列的网盘服务，将整个项目作为一个文件夹保存。毕竟这个项目的核心在于用于储存文章的 markdown 文件和一些配置文件。

但是上述的使用网盘类服务做即时备份的时候有一个非常严重的问题：因为项目中的`node_modules/`文件夹的存在，在每一次项目中的 npm package 发生变化的时候，会有许多的文件变化，而这些海量的文件变化数次卡住了笔者的 OneDrive 同步，以至于其他的文件都无法及时得到同步。因此笔者才会痛定思痛，避免使用这些服务对本项目进行备份。

## github 工作流
其实如果仅仅是使用 github 作为一个备份服务的话，我们仍旧可以使用先前的 Hexo 部署工作流，只不过是 github 除了保存静态文件 (`gh-pages`) 分支以外，多保留了一个博客源码 (`blog-src`) 分支。但是在笔者看来，这么做的缺点一方面是在操作上有许多冗余。在写完文章保存 md 文件后，除了`hexo deploy`以外还需要使用`git add/git commit/git push/git fetch`等一系列的指令。另一方面，如果已经对 git 指令非常熟悉，那么之前工作流中的`hexo deploy`也会成为比较多余的步骤。

在对 git 有一定的了解，能够很好地在 github 上管理这两个分支的情况下，将 github 整合 CI 可以有效地简化工作流，自动完成后续的部署步骤。

而笔者这次尝试主要是盯上了 github actions。原因如下：
1. 免费的资源不用白不用
2. github actions 不需要额外配置，甚至不需要额外生成 token

笔者最开始是尝试将 hexo deploy 这一个 command 本身放到 github actions 的 config 之中。但是因为无法将 github token 环境变量成功传入，所以只得寻找其他的解决方案。

在 Google 的帮助下笔者找到了一个现成的[解决方案](https://github.com/marketplace/actions/github-pages-action#%EF%B8%8F-set-another-github-pages-branch-publish_branch)，是别人已经做好了的 Github Action。根据这个 GitHub Action 的文档，它可以被运用在非常多的静态博客上。除了该项目正在使用的 Hexo 以外，还有 Hugo，MkDocs，mdBook 等等静态网站生成器可以使用该 action 进行自动化部署。

笔者使用的配置如下

```json
name: GitHub Pages

on:
  push:
    branches:
      - <source branch>
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    permissions:
      contents: write
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
    steps:
      - uses: actions/checkout@v3

      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '14'

      - name: Cache dependencies
        uses: actions/cache@v2
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - run: npm ci
      - run: npm run build

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: ${{ github.ref == 'refs/heads/main' }}
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          # publish_branch: <target branch>
          publish_dir: ./public
```

需要注意的是，该 action 需要响应的 branch 应当替换为博客源码 branch（blog-src）。而在最后的 Deploy 环节，需要将 publish_branch 替换成存放静态文件的 branch（gh-pages）。在正确更改配置后，将文件放到`./.github/workflows/main.yml`即可创建该配置。从此以后每次需要发布新文章至博客就无须手动部署了，每次博客源码分支的更新都会触发 github action 的自动构建和部署。

至此笔者的创作工作流便可以切换到 git 工作流了，可以完完全全将博客视作一个 GitHub 仓库进行维护。