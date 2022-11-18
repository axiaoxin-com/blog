---
title: "使用 Github Actions 自动部署 Golang 应用到 VPS 服务器"
author: "axiaoxin"
date: 2021-06-19T16:12:08+08:00
subtitle: ""
image: ""
tags: ["golang", "工具", "github"]
slug: ""
---

## 环境说明

使用 GitHub Actions 可以在你提交代码后自动将最新代码编译发布部署到你的 VPS 服务器，这里记录一下配置步骤。

服务没有使用 Docker 部署，直接通过 supervisord 启动二进制文件，期望每次提交代码后自动完成编译部署发布上线，通过 Github Actions 实现自动化部署。

**自动部署流程**：

0. 提交代码到主分支
1. Github拉取最新代码跑单元测试后编译二进制文件
2. scp 二进制文件到VPS服务器
3. ssh 在VPS上远程执行服务重启命令

## 配置步骤

**1、创建SSH KEY**

由于是使用 ssh 进行部署，需要让 github 能登录服务器，因此先在 VPS 服务器上给 github actions 创建一个 ssh key：

```sh
cd ~/.ssh
ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): /root/.ssh/github_action
...
```

将生成的公钥保存到 authorized_keys 文件中：

```sh
cat github_action.pub >> authorized_keys
```

**2、配置 Actions Secrets**

由于部署过程会涉及一些隐私的变量，比如 scp、ssh 等需要的密钥信息等，可以将这些信息在代码仓库的 Settings/Secrets 中进行配置。如图：

![配置 Actions Secrets](https://user-images.githubusercontent.com/2876405/122633113-7606cc80-d109-11eb-986a-383b1f145b74.png)

在你的 Golang 应用的代码库页面进入Settings选择Secrets后，点击右上方的【New repository secret】创建 Secret 变量，Name 填变量名，Value 填变量值，
配置后可以在Actions的配置yml文件中使用`${{ secrets.XXX }}`来使用。

**3、创建 Github Actions workflows**

你可以直接在代码目录新建配置文件 `.github/workflows/deploy.yml`，也可以直接在github项目页面点击 Actions 页面创建并直接编辑提交，如图：

![创建 Github Actions workflows](https://user-images.githubusercontent.com/2876405/122633541-bbc49480-d10b-11eb-9f8d-d3faace030a9.png)

yml文件内容设置如下：

```yaml
name: deploy go

on:
    push:
    branches: [ main ] # main分支上push触发部署
    pull_request:
    branches: [ main ] # merge到main分支时触发部署

jobs:

    build:
    # 在ubuntu上进行构建操作
    runs-on: ubuntu-latest

    steps:
    # 拉取代码
    - uses: actions/checkout@v2

    # 设置你项目的golang版本
    - name: Set up Go
        uses: actions/setup-go@v2
        with:
        go-version: 1.16

    # 执行单元测试
    - name: Test
        run: go test -v ./...

    # 编译二进制
    # 注意没有使用cgo的务必加上CGO_ENABLED=0，不然可能编译通过但VPS上没有GLIBC或GLIBC版本不一致会导致无法启动
    - name: Build
        run: CGO_ENABLED=0 go build -o app -tags=jsoniter

    # 将编译出的二进制文件 scp 到你的VPS服务器
    - name: SCP Files
        uses: appleboy/scp-action@master
        with:
        host: ${{ secrets.REMOTE_HOST }}  # Secrets中的配置：vps IP地址
        username: ${{ secrets.REMOTE_USER }}  # Secrets中的配置：vps 登录用户名
        port: ${{ secrets.REMOTE_PORT }} # Secrets中的配置：vps 端口
        key: ${{ secrets.SERVER_SSH_KEY }} # Secrets中的配置：vps 上创建的ssh key的私钥内容
        source: 'app'  # 编译出的二进制文件名
        target: ${{ secrets.REMOTE_TARGET }} # Secrets中的配置：scp 到 vps 上的的目录

    # 通过ssh远程执行命令重启vps上的服务
    - name: SSH Remote Commands
        uses: appleboy/ssh-action@master
        with:
        host: ${{ secrets.REMOTE_HOST }} # Secrets中的配置：vps IP地址
        username: ${{ secrets.REMOTE_USER }} # Secrets中的配置：vps 登录用户名
        port: ${{ secrets.REMOTE_PORT }} # Secrets中的配置：vps 端口
        key: ${{ secrets.SERVER_SSH_KEY }} # Secrets中的配置：vps 上创建的ssh key的私钥内容
        script: ${{ secrets.AFTER_COMMAND }} # Secrets中的配置：scp二进制文件到vps服务器上后需要执行的相关shell命令重启服务
```

创建完成后，每次push到main分支就会自动执行steps中设置的步骤完成自动化部署。

你也可以根据你的实际需求，参考 [官方文档](https://docs.github.com/cn/actions/learn-github-actions) 修改 yml 配置。
