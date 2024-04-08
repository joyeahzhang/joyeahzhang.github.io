+++
title = 'Vscode中的C++环境配置'
discription = ""
tags = ["clangd"]
categories = ["vscode"]
date = 2024-04-08T16:05:34+08:00
draft = false
slug = ""
+++

# Vscode中的C++环境配置
本篇介绍windows下基于cmake + llvm-mingw + clangd + vscode-clangd插件完成代码生成、提示和跳转的环境配置。

## cmake准备

从cmake官网(https://cmake.org/download/)下载适合自己平台的camke，一路按自己的喜好安装，最后确保cmake加入到了系统的环境变量中。

测试是否安装成功：
```
cmake --version
```
安装并配置环境变量成功会显示cmake的版本信息。

## 编译器准备

从github下载最新的llvm-mingw(https://github.com/mstorsjo/llvm-mingw/releases)，下载后将bin目录添加到系统的环境变量中。

测试是否安装成功：
```
mingw32-make --version
```
安装并配置环境变量成功会显示mingw32-make的版本信息。

## 安装clangd作为language server

clangd是llvm项目推出的C++语言服务器，通过LSP (Language Server Protocal)协议向编辑器如vscode/vim/emacs提供语法补全、错误检测、跳转、格式化等等功能。

万幸，在上一步中下载的llvm-mingw就已经包含了clangd.exe(在bin目录中)，我们无需另行下载

## 安装插件

最基本的vscode插件就是clangd，我们安装好clangd插件后，进入用户配置json，添加如下的文本：
```json
"clangd.detectExtensionConflicts": true,
    "clangd.arguments": [
        // 在后台自动分析文件（基于complie_commands)
        "--background-index",
        // 标记compelie_commands.json文件的目录位置
        "--compile-commands-dir=${workspaceFolder}/build",
        // 同时开启的任务数量
        "-j=12",
        // clangd使用的编译器
        // "--query-driver=/usr/bin/clang++",
        // clang-tidy功能
        "--clang-tidy",
        "--clang-tidy-checks=performance-*,bugprone-*",
        // 全局补全（会自动补充头文件）
        "--all-scopes-completion",
        // 更详细的补全内容
        "--completion-style=detailed",
        // 补充头文件的形式
        "--header-insertion=iwyu",
        // pch优化的位置
        "--pch-storage=disk",
      ],
    "clangd.path": "your-path-to-clangd.exe",
```

## 编写CMakeLists.txt并制定cmake -G "MinGW Makefiles"

这一步我不知道为什么必须-G，但是如果不这样做，那么cmake会自动使用nm而非mingw32-make导致失败。

另外CMakeLists.txt中一定要指定compile_commands.json文件的生成，因为clangd就是基于它来获取代码信息的。

```
# 使CMake生成compile_commands.json文件
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

