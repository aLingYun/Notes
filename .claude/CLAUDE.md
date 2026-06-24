# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 仓库概览

Obsidian 个人知识库/笔记仓库，内容以中文技术文章为主。

### 目录结构

- `Rust/` — Rust 技术文章（错误处理、tracing 日志、crate 使用技巧等）
- `Linux/` — Linux 系统配置、工具链搭建（Arch Linux、Neovim、i3 等）
- `AI/` — AI 相关笔记
- `Algorithm/` — 算法学习笔记（Rust 实现）
- `RISC-V/` — RISC-V 指令集学习笔记
- `Python/` — Python 实用工具笔记
- `区块链/` — 区块链相关笔记（Substrate、Rust 实现区块链）
- `Others/` — 其他（ffmpeg、Obsidian 同步、SDL2 等）
- `.obsidian/` — Obsidian 配置（启用插件：obsidian-git）

## 写作风格

所有文章采用中文技术博客风格，有统一的文章模式：
1. 痛点/场景切入（ relatable 开头）
2. 核心概念解释
3. 实战代码示例（带注释）
4. 对比表格/总结
5. 可复制的完整范例（彩蛋）
6. 友好的结尾

图片附件统一放在各目录下的 `attachments/` 子目录中。

## 常用操作

### 代码验证
Rust 文章中的代码示例，在 `/home/wlb/Documents/codes/rust/exam/` 下的 Cargo 项目中验证编译和运行。项目有多个 binary，根据验证的文章选择对应的 binary：
```bash
cd /home/wlb/Documents/codes/rust/exam

# 验证错误处理基础 + anyhow-vs-thiserror 两篇文章
cargo run --bin exam

# 验证 tracing 文章
cargo run --bin tracing_demo
```
依赖已在 `Cargo.toml` 中配置（anyhow、thiserror、serde、serde_json、toml、tracing、tracing-subscriber 等）。

### Git 操作
仓库由 Obsidian Git 插件自动备份，commit message 格式为 `vault backup: YYYY-MM-DD HH:MM:SS`。手动提交时也沿用此格式。

### 新建文章
在对应主题目录下创建 `.md` 文件，文件名使用中文描述性标题。Rust 文章中的代码示例需同步在 `exam/` 项目中验证。

## 关键依赖

Rust 文章常用 crate：`thiserror`、`anyhow`、`serde`、`serde_json`、`toml`、`tracing`、`tracing-subscriber`，均配置在 `exam/Cargo.toml` 中。
