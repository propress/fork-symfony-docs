# 第 00 章：入门——心智准备、安装、项目解剖

> **一句话说明**：在写第一行 Symfony 代码之前，先装好工具、建好项目、认识文件。
> **对应官方英文文档**：`setup.rst`、`page_creation.rst`
> **Symfony 版本基准**：8.0
> **前置阅读**：本书 [`README.md`](../README.md)、[`BOOKSTORE_PROJECT.md`](../BOOKSTORE_PROJECT.md)

---

## 🎯 30 秒速览

- **本章干什么**：把开发机调到"能开始学 Symfony"的状态，并理解"一个 Symfony 项目到底是什么"。
- **完成后你能**：
  - 用一条命令创建 Symfony 8.0 项目并在浏览器看到 Welcome 页
  - 看懂根目录每个文件 / 文件夹大致是干嘛的
  - 知道 `symfony` / `composer` / `php bin/console` 三个命令的分工
  - 给 bookstore 起一个能跑的 git 仓库

## 本章结构

| 小节 | 标题 | 目标读者 | 时长 |
| --- | --- | --- | --- |
| 01 | [心智准备、安装、项目解剖（主线）](./01_心智准备_安装_项目解剖.md) | 所有人必读 | 30 分钟 |
| 02 | 第一次请求-响应全链路 *（下次续跑）* | 想懂"这一下发生了什么"的人 | 20 分钟 |
| 03 | 项目骨架逐文件解剖 *（下次续跑）* | 想知道每个文件在干嘛的人 | 15 分钟 |
| 07 | 选择指南：Symfony CLI / Docker / 原生 PHP *（下次续跑）* | 拿不准选哪个的人 | 10 分钟 |
| 08 | 常见报错图鉴 *（下次续跑）* | 装不上 / 跑不起来的人 | 按需 |
| 09 | 陷阱与反模式 *（下次续跑）* | 所有人必读 | 10 分钟 |
| 10 | 调试手册：`about` / `debug:*` 初遇 *（下次续跑）* | 所有人必读 | 10 分钟 |
| 12 | bookstore 增量：仓库初始化 *（下次续跑）* | 跟做的人 | 10 分钟 |
| 13 | FAQ *（下次续跑）* | 所有人 | 按需 |
| 99 | 速查表 *（下次续跑）* | 所有人 | 查时用 |

> 本次发布只包含 **01 节**。其他小节将按 `PROGRESS.md` 中的「下次开工清单」推进。任务书明确要求"宁可一次只交付一节高质量内容，也绝不提交十节半成品"，本书严格遵守。

## 你将学会

- Symfony 在整个 PHP 世界中的位置（和 Laravel / Lumen / Slim 不一样在哪）
- PHP 8.4、Composer、Symfony CLI 三件套的安装
- `symfony new --webapp` 跟 `symfony new` 的区别（最容易踩的坑）
- Symfony 8.0 项目的目录结构心智模型
- 为什么 `vendor/` 不该进 git，`var/cache/` 不该进 git
- `symfony` / `composer` / `php bin/console` 三条命令谁管什么

## 前置知识

- **必需**：PHP 基础语法、命名空间、Composer 是什么（不需要精通）
- **建议**：用过任意一个 PHP 框架（Laravel / ThinkPHP）——没有也不影响
- **不需要**：任何 Symfony 经验

## 给急着干活的人

如果你现在就要一个能跑的 Symfony 项目，最短路径：

```bash
$ symfony check:requirements          # 看看你的机器还缺什么
$ symfony new bookstore --version="8.0.*" --webapp
$ cd bookstore
$ symfony server:start                # 用浏览器打开 https://127.0.0.1:8000
```

但这种"知其然不知其所以然"的跑法，**两周后你会在第一次报错时崩溃**。所以强烈建议读完 01 节再动手。
