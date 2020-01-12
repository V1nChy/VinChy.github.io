---
layout: post
title:  "热更新方案记录"
date:   2019-12-21
excerpt: "mmorpg手游项目中热更新流程、管理方案记录。"
tag:
- mmorpg
- 手游
comments: false
---
## 概述

在实际项目的热更新方案选择中，使用Tolua热更新方案，需要热更的内容会涉及到资源、配置和代码，并且项目中的战斗、功能模块相关逻辑全部都由lua实现，也就是说除了热更新管理、资源管理、网络管理等基础核心模块会以C#实现，其他的整个游戏都是要能进行热更的，这就要求游戏客户端框架中，需要一套完善的版本资源热更系统，进行资源下载更新。

## 出包

在游戏出包时，包体中除需附带必须的初始资源，还需要包含关键配置数据文件，其中需要有：
1. 包体配置。当前包相关的资源后台、平台、版本、语言配置等。
2. 资源版本信息库。当前包版本的所有AssetBundle信息db库、二进制文件，以及记录库md5变化信息的资源版本文件。
3. 当出非空包时（比如需要包含新手阶段所有资源的包），需要将用到的AssetBundle打包成zip文件，将这些zip文件和记录文件移动到StreamingAssets，在游戏首次启动时进行解包。

## 资源打包

当lua代码、资源改动到，有热更需求的时候，则进行打包，并生成必要的版本信息文件：
1. 生成的Bundle涉及到3中。lua代码Bundle；必要资源AssetBundle（UI界面资源、公用场景资源，及所有依赖关系的资源）；非必要资源AssetBundle（单独的大图，音频文件等无需依赖关系的资源）。
2. 根据最新Bundle，生成版本信息二进制文件，分为必要资源版本文件与非必要资源版本文件，lua代码Bundle属于必要资源文件。
3. 生成版本二进制文件md5值记录文件，热更时首先获取这个文件，根据md5值判断需要检查更新的bundle。

## 更新流程

游戏客户端启动时，每次都执行以下流程，检查更新本地文件：  
1. 加载本地包体配置。资源更新起点，从配置中获得资源后台地址,以及apk版本，平台配置，语言配置等关键性配置。
2. 获得后台包体配置。从资源后台取得当前包体的最新配置，根据不同的发包需求，动态修改配置。
3. 检测apk版本。根据最新的配置，检测当前apk版本是否为最新的。
4. 资源解压。检查是否为初次启动，如果非空包则进行资源解压。
5. 下载最新版本的资源。必要资源bundle需要在游戏启动后就更新到最新，才允许进入游戏；而非必要资源bundle在玩家用到时再进行下载，或者进入到游戏中，在空闲时间自动启动下载更新。游戏启动后进行的资源更新步骤主要有：  
   1. 从资源后台获取最新的版本二进制文件md5值记录文件，根据文件中记录的必要资源bundle与非必要资源bundle的md5值,与本地md5值进行比对。一旦发生改变，则进行bundle检查更新流程。
   2. 初始化本地bundle的db库，并从后台下载最新版本信息二进制文件，进行版本、md5比对，确定需要更新的必要资源文件列表。
   3. 进行必要资源bundle下载。下载完成后进入游戏。
  
## 总结
这套资源热更流程基本满足现有项目需求，但有些做法并不是很高效，后续还能跟进优化。
* 项目中的做法，在检查更新时需要遍历所有的资源文件，且需要加载本地文件生成md5，这就造成了巨大的io压力，且会产生大量GC Alloc。
* 版本号管理不是很明确，无法直观的获知当前更新到哪个版本；是否可以建立一套基于版本号热更资源的机制，以版本号去确定需要热更的资源，而不用去遍历检查所有资源，还有待商榷。
* 只能一个资源一个资源的下载，不支持多任务下载更新，也不能够断点续传，不过这也是因为项目热更的资源普遍较小吧。