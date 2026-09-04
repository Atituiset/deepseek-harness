# Agent Note: 经文档站投影发布教程书

Status: implemented

[English](2026-09-04-learn-book-projection.md) | 中文

## Problem

`learn/` 下的 15 章 Agent 开发教程此前作为独立的 mdBook 站点发布。部署它意味着 Pages workflow 要安装一个钉版本的 mdBook 二进制，把 `learn/book/` 构建为 `website/.dist` 之外的第二棵产物树，再挂载到 `/learn/`——没有共享导航、搜索和语言切换。教程游离在发布清单之外，章节无法用路由链接到已发布文档，站点的链接、大纲与 fragment 校验也从不覆盖它。

## Decision

mdBook 层已移除。`learn/src/` 仍是权威 Markdown 层（教程拥有自己的层级，正如 `docs/user/` 拥有指南），发布走既有投影：[website/docs.ts](../../../../website/docs.ts) 以 `mirroredPages` 声明全部 16 页——两个 locale 路由有意投影同一份可用的中文源，挂在新的 `learn/` 顶级模块下，拥有独立的 sidebar collection（`zh-learn`/`en-learn`）与导航入口（教程 / Learn）。

projector 把章节间的互引重写为站内路由：指向 `docs/architecture.md` 的行为保持代码文本（对照源码阅读是本书的用法），章与章的链接在站内导航。`pnpm run doc-sync` 从此用与其它已发布页面相同的门禁覆盖教程，包括对已构建站点的 fragment 校验。

配对 spec 接受了一个泛化：名字不是 `.zh.md` 的 `zh-CN` 页面，只要清单把它镜像进两个 locale 路由，就可以发布——这正是 Cordis `inherited.md` 条目早已在英文方向使用的 `mirroredPages` 契约。配对命名的源仍保持更严格的兄弟断言。

## Alternatives considered

**保留 mdBook 并嵌入其产物。** 拒绝：两套构建系统、两棵产物树，且站点门禁永不校验教程。

**把 `learn/src/**` 改名为 `.zh.md` 并写英文占位。** 拒绝：英文对应方应该只在真的有人写时才存在，空占位会搅乱配对语料。

**把章节移进 `docs/`。** 拒绝：教程是独立的学习路径而非子系统参考；独立层级保住了它的章节编号与专属大纲深度（`[2, 3]`）。

## Consequences

一次构建产出整站；workflow 删掉了 mdbook 安装步骤，`learn/book/`、`book.toml`、`src/SUMMARY.md` 一并删除。教程免费获得站点导航、本地搜索与语言切换。`/en/learn/` 的跨语言读者今天读到中文源——这是文档化的 `mirroredPages` 回退——将来落地英文翻译时，把条目换成 `pairedPages` 只是清单改动。
