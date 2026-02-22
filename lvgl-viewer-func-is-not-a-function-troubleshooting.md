---
title: 把一次 LVGL Viewer 的 TypeError 查到根上
date: 2026-02-22 12:00:00
description: "这次排查从“Could not load runtime”一路走到“TypeError: func is not a function”。中间试了项目结构、目录命名、最小复现，最后定位到 preview runtime 导出的 init 符号和 project.xml 项目名不一致。"
categories:
 - 折腾记录
tags:
 - LVGL
 - 调试
 - WebAssembly
 - UI工程
---

这两天在做一个 toaster 的 LVGL XML 项目，目标很简单：把项目丢到在线 Viewer 里直接预览。结果第一眼就翻车了，先是 `Could not load runtime`，修完以后又卡在更诡异的报错：`Error processing message: TypeError: func is not a function`。

最折磨人的不是报错本身，而是它看起来像“哪都可能错”。XML 可能错，资源可能错，目录结构可能错，runtime 也可能错。于是这次我干脆按“证据链”来，一步步排除，不猜。

## 第一轮：先把明显问题修掉

最开始 `Could not load runtime` 其实不难，给项目补上 `preview-bin` 里的 runtime 文件就能过。对应改动是把 `lved-runtime.js`、`lved-runtime.wasm`、`.lvgl-version` 放到项目目录里，这一步之后，Viewer 至少能启动运行时。

但问题没结束。页面不再报 runtime 缺失，却继续报 `TypeError: func is not a function`。也就是说，运行时加载了，但调用链里某个函数是 `undefined`。

## 第二轮：怀疑项目结构和目录命名

我当时优先怀疑的是“项目形状”不符合 Viewer 预期。因为官方文档写得很明确：目标目录至少要有 `project.xml` 和 `globals.xml`。于是我做了几件事。

先补齐 `components/fonts/images/screens/widgets` 这些目录下的 `README.md`，让结构更像官方例子。
再建一个最小 smoke 项目，只放一个 screen 和一行 label，尽量把 UI 复杂度降到 0。
再做 A/B：`toaster-smoke` 和 `toastersmoke` 两个目录都试，排除连字符命名的影响。

结果是：最小项目照样报同一个 `TypeError`。这一步很关键，它基本把“UI XML 写复杂了”这条线砍掉了。

## 第三轮：比对 runtime，找调用入口

既然最小 UI 也报错，那就该看 runtime 本体了。
我把工作目录和 `examples` 的 runtime 做对比，发现 wasm 和版本号看起来都没问题，但导出符号里有个细节很刺眼：项目 runtime 只暴露了 `examples_init`。

而我们的 `project.xml` 里项目名是 `toaster_smoke`（另一个项目是 `toaster_ui`）。按常理，Viewer 会按项目名去找 `<project_name>_init`，也就是 `toaster_smoke_init` / `toaster_ui_init`。如果 runtime 只有 `examples_init`，那调用时拿到的就是 `undefined`，最后就会变成那句熟悉的：`func is not a function`。

这就解释了为什么“能加载 runtime，但一切照样挂”。

## 真正根因

根因不是 XML 布局，不是图片，不是目录名。
根因是 **`project.xml` 的项目名和 runtime 导出的 init 符号不一致**。

简单说：Viewer 在找 A，runtime 只提供了 B。

## 修复方式

这次先用兼容补丁止血：在 `lved-runtime.js` 里补上别名，把 `toaster_smoke_init`、`toaster_ui_init`、`toastersmoke_init` 映射到已有的 `examples_init`。对应提交是：

`03d6316 Fix viewer init symbol mismatch in toaster runtime bundles`

补完后，下面这个链接可以正常打开并运行：

`https://viewer.lvgl.io/?repo=https://github.com/hayschan/lvgl_editor/tree/master/toastersmoke`

## 复盘

这次排查对我最大的提醒是：Web 端报错如果很“泛”，就尽快做最小复现，把“业务层”变量全砍掉。
一旦最小项目还能稳定复现，问题大概率就在基础设施层，比如加载链、符号名、协议约定，而不是 UI 细节本身。

另外，临时别名是能救火的，但长期正确做法还是：**从目标项目重新导出匹配项目名的 runtime**，让 `project.xml` 和 runtime 导出天然一致。这样后面迁移、协作、升级版本时，坑会少很多。
