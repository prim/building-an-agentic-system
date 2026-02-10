## Claude Code vs. anon-kode vs. Amp

了解这个生态系统有助于理解本指南中的架构模式：

### Claude Code
Anthropic 的本地 CLI 工具，将 AI 能力直接带入终端：
- **架构**：Node.js 后端搭配 React/Ink 实现终端 UI
- **定位**：面向单用户的本地开发，提供强大的文件和代码操作能力
- **核心创新**：支持流式响应的响应式终端界面
- **发布形式**：研究预览版，免费使用（[文档在此](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview)）

### anon-kode
Daniel Nakov 的开源分支，扩展了 Claude Code 的能力：
- **主要新增**：
  - 多供应商支持（OpenAI、本地模型等）
  - `/model` 命令用于切换供应商
  - UI 定制和调整后的默认设置
- **架构**：保持 Claude Code 的核心设计
- **价值**：展示了基础架构如何支持不同的 AI 后端

### Amp
Anthropic 的协作平台，将这些概念扩展到团队使用：
- **演进**：将 Claude Code 的模式带入多用户环境
- **核心特性**：
  - 实时协作和共享
  - 企业认证（SSO、SAML）
  - 团队工作流和权限管理
  - 用量分析和成本管理
- **架构**：支持状态同步的分布式系统
- **目标用户**：需要协作式 AI 开发的团队和企业

### 为什么这很重要

本指南分析了这三个系统的模式：
- **第一册**聚焦于 Claude Code 和 anon-kode 共有的本地模式
- **第二册**探索 Amp 如何将这些模式扩展到协作场景

Claude Code 的架构决策奠定了一个基础，anon-kode 和 Amp 都能在此基础上构建——理解这些模式将帮助你在任何规模上构建自己的 AI 编程助手。

