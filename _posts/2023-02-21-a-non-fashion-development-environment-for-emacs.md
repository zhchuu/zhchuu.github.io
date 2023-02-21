---
layout: post
title: "A Non-fashion Development Environment for Emacs"
date: 2023-02-21
categories: blog
permalink: /:categories/:year/:month/:title.html
---


## 0. Preliminary

本文简单介绍一种 Emacs 开发环境配置，它的特点是简单高效，依赖轻量，所有开发语言通用，大规模项目下也流畅。

## 1. General Requirements

通用的开发环境无非是满足以下要求：

- 语意补全（Semantic completion）

- 代码跳转（Jump to definition）

- 代码浏览（Code navigation）

- 文档提示（Document）

- 实时语法检测（Real-time syntax checking）

语意补全的要求很高，对我个人而言能力弱一些的补全也能接受。实时语法检测几乎无法完成，也不是很必要，尤其是大规模、分布式的项目。但是代码跳转和代码浏览是刚需。

### 1.1 Package list

|                       | 语意补全 | 代码跳转 | 代码浏览 | 文档提示 | 语法检测 |
|:----------------------|:--------:|:--------:|:--------:|:--------:|:--------:|
| ``lsp-mode``          | Y        | Y        | Y        | Y        | Y        |
| ``ctags`` + ``citre`` | N        | Y        | Y        | N        | N        |
| ``cscope``            | N        | Y        | Y        | N        | N        |

要实现以上功能最著名的包莫过于 ``lsp-mode``，LSP 是 Language Server Protocol 的简称，是微软在开发 VS Code 时提出来的，现在已经成为了众多 IDE 配置开发环境的标准。``lsp-mode`` 是 Emacs 使用 LSP 的模式。但这篇文章并不讲它，因为官方文档已经阐述得足够详细了，简单讲讲另外几个。

## 2. ctags + citre

[[Universal Ctags](https://github.com/universal-ctags/ctags)]：``ctags`` 最早是为 C 语言生成 tags（索引）的工具，经过发展目前支持 140 多种语言。它能够为源代码的变量、函数、类、结构体等构建索引，排序后写入到 tags 文件中。Universal Ctags 是目前仍在维护的分支。

[[citre](https://github.com/universal-ctags/citre)]：``citre`` 是 Emacs 的 tags 前端。它主要借助 ``readtags`` （From Universal Ctags）工具读取 tags 的内容展示到 Emacs 中。由于 tags 有序，``readtags`` 用二分查找读取，速度是有保证的。

### 2.1 Installation of ctags

根据官网的文档编译安装即可，没什么特殊的要求，安装完成后可以得到：

```bash
$ ctags --version | head 1
Universal Ctags 5.9.0(2fd9b0c7), Copyright (C) 2015-2022 Universal Ctags Team
```

```bash
$ readtags --version
5.9.0
```

### 2.2 Installation of citre

这个包开箱即用，用官网推荐的、没几行的配置就能用：

```lisp
(use-package citre
  :defer t
  :init
  (require 'citre)
  (require 'citre-config)
  (global-set-key (kbd "M-.") 'citre-jump)
  (global-set-key (kbd "M-,") 'citre-jump-back)
  (global-set-key (kbd "M-P") 'citre-peek)
  (setq
   ;; Set these if readtags/ctags is not in your PATH.
   citre-readtags-program "/opt/homebrew/bin/readtags"
   citre-ctags-program "/opt/homebrew/bin/ctags"
   citre-auto-enable-citre-mode-modes '(prog-mode))
  )
```

二进制文件的路径根据自己环境配置即可，最重要的三个操作：跳转定义、跳出定义和小窗查看，我分别绑定了``M-.````M-,``和``M-P``。

### 2.3 Generate tags

以 Linux 内核源码为例，它足够庞大，用来验证速度合适，在源码目录下执行：

```bash
$ ctags --exclude=build --languages=c,c++ -R
```

tags 文件会生成在本目录下：

```bash
$ du -h ./tags 
1.0G	./tags
```

### 2.4 Now, it should look like...

``citre-peek``：打开小窗查看函数的定义。

<p align="left">
   <img src="/assets/a-non-fashion-development-environment-for-emacs/citre_peek.jpg" width=440/>
</p>

``complete-at-point``：配合 ``company-mode`` 实现补全。

<p align="left">
   <img src="/assets/a-non-fashion-development-environment-for-emacs/complete_at_point.jpg" width=440/>
</p>

速度很快，可以用 in a blink 来形容。做不到语意补全，但也够用。


## 3. cscope

[[cscope](https://cscope.sourceforge.net/)]：``cscope`` 是一个和 ``ctags`` 类似的工具，原理也差不多，最早诞生于贝尔实验室。

[[xcscope](https://github.com/dkogan/xcscope.el)]：``xcscope`` 是 Emacs 与 ``cscope`` 对接的包。

### 3.1 Generate cscope.files

在 Linux 源码目录下执行，命令的含义是找到所有 .c / .h / .cpp 文件（可以自定义加上别的）并对它们构建索引：

```bash
$ find ./ -name "*.c" -o -name "*.h" -o -name "*.cpp" | tee cscope.files
$ cscope -Rbkq -i cscope.files
```

### 3.2 Installation of xcscope

同样开箱即用的包：

```lisp
(use-package xcscope
  :init
  (setq cscope-index-recursively 1)
  :config
  (cscope-setup))
```

### 3.3 Now, it should look like...

``cscope-find-functions-calling-this-function``：找到所有调用该函数的地方（并不是简单的 Grep 或 Ag 搜索）。

<p align="left">
   <img src="/assets/a-non-fashion-development-environment-for-emacs/cscope_find_functions_calling_this_function.jpg">
</p>

可以看到搜索大约花费 1.7 秒，当然不同函数调用次数不同，时间会有一定的差异，但对于这么大的项目来说算得上合格（大部分时候我会配合 ``projectile-ag`` 来使用）。

## Summary

- 两个包（citre 和 xcscope）的配置加在一起不超过 20 行，外加 3 行的执行命令，就能够丝滑地对 Linux 这样大的项目进行代码跳转和浏览，我觉得是相当高效率了。投入小，回报大，很符合降本增效的理念（:p）。

- 两个包还有很多其它功能，我所介绍的不足十分之一，用得好的话阅读代码会非常方便。当然它也有缺点，毕竟构建的索引是静态的，如果代码更新频繁还需要同时更新索引文件。

- 本文提供一种能够快速浏览陌生项目的思路。比如拿到一个新语言的项目，但一时间没那么快配置好对应的 LSP Client，那么本文提供的方法或许能帮上忙。

## Reference

- [CScopeAndEmacs](https://www.emacswiki.org/emacs/CScopeAndEmacs)
- [lsp-mode](https://emacs-lsp.github.io/lsp-mode/)
