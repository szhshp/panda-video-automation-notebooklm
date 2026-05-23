<div align="center">
  <img src="./docs/panda-notebooklm.png" width="200" alt="Panda Video NotebookLM Logo">
  
  # Panda Video — NotebookLM
  
  **NotebookLM 视频自动化引擎**

  *灵感源自 Panda Video Generator · 专为 NotebookLM 笔记转视频场景优化*

  [![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
  [![TypeScript](https://img.shields.io/badge/TypeScript-5.9-blue)](https://www.typescriptlang.org/)
</div>

---

<div align="center" style="margin: 12px 0 20px;">
  <a href="https://github.com/szhshp/panda-video-automation-notebooklm" title="Panda Video NotebookLM 仓库" style="display: inline-block; padding: 12px 28px; border: 2px solid #0969da; border-radius: 8px; font-weight: 700; font-size: 1.05em; text-decoration: none; color: #0969da;">GitHub 仓库</a>
</div>

## 上游项目

- V1: 一切的起源-熊猫视频自动化引擎
  - [Panda Video Generator](https://github.com/szhshp/panda-video-generator)
- V2: 已开源-熊猫视频自动化发布工具
  - [panda-video-automations-publisher](https://github.com/szhshp/panda-video-automations-publisher)

## ✨ 核心特性
<!-- TODO -->

<!-- GitHub.com strips most flex/gap inline styles; use a plain row table so 3 columns match on the website. -->
<table cellspacing="20" cellpadding="0" border="0">
  <tr valign="top">
    <td width="34%" valign="top">
      <h3>📝 <mark>一键</mark> NotebookLM 研究</h3>
      <p>一键创建笔记本、Web 研究、导入来源，自动完成 NotebookLM 研究流程。</p>
    </td>
    <td width="33%" valign="top">
      <h3>🎬 <mark>一键</mark>视频生成</h3>
      <p>一键从 NotebookLM 笔记本导出视频制品（Deep Dive 对话 / 音频概览），自动下载并准备上传。</p>
    </td>
    <td width="33%" valign="top">
      <h3>🚀 <mark>一键</mark>多平台发布</h3>
      <p>一键驱动浏览器自动化上传；B 站、抖音、微信视频号、小红书等共用相近流程。</p>
    </td>
  </tr>
</table>

## 接入方式

<table cellspacing="20" cellpadding="0" border="0">
  <tr valign="top">
    <td width="100%" valign="top">
      <h3>🤖 Agent Skills 方式</h3>
      <ul>
        <li>在 AI Agent 中直接运行技能，编排 NotebookLM 视频生成流程。</li>
        <li>支持研究、视频下载、多平台上传等技能。</li>
        <li>支持 Cursor、Claude Code、Copilot 等常用 AI Agent。</li>
      </ul>
      <p>无需手动安装 CLI，首次运行 <code>/setup-pva-notebooklm</code> 技能即可自动完成依赖检查和安装。</p>
    </td>
  </tr>
</table>

---

## 📖 简介

**Panda Video — NotebookLM** 灵感源自 [Panda Video Generator](https://github.com/szhshp/panda-video-generator)，发布上传部分基于 [panda-video-automations-publisher](https://github.com/szhshp/panda-video-automations-publisher)。专注于 NotebookLM 笔记转视频场景，提供从 NotebookLM 笔记本创建、研究、视频导出到多平台发布的一站式自动化工作流。

---

## 🚀 快速开始

### TL'DR

在 AI Agent 中直接运行技能 `/setup-pva-notebooklm`，自动检查所有依赖是否就绪。

### 先决条件

- **Python 3** — 安装 [notebooklm-py](https://pypi.org/project/notebooklm-py/)
- **Node.js 20+**

> **无需手动安装 PVA CLI**。首次运行技能 `/setup-pva-notebooklm` 会自动完成依赖检查和安装。

---

## 🧩 可用技能

| 技能 | 说明 |
|------|------|
| `setup-pva-notebooklm` | 检查所有前置依赖 (Python, Node.js, notebooklm-py, PVA) |
| `setup-pva-notebooklm` | 安装 @panda-video-automation/pva |
| `notebooklm-status` | 查看所有笔记本及制品状态 |
| `notebooklm-research` | 创建笔记本并执行深度研究 |
| `notebooklm-prep-upload` | 准备上传内容到 NotebookLM |
| `notebooklm-video` | 从笔记本生成视频 |

---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

---

## 📄 许可证

本项目采用 MIT 许可证。详见 [LICENSE](LICENSE) 文件。

---

## 👤 作者

**szhshp**

- Email: 24031shp@sina.com
- GitHub: [@szhshp](https://github.com/szhshp)

---

## ⚠️ 免责声明

本项目按「原样」提供，作者不对因使用本软件而产生的任何直接、间接或附带损失承担责任。你在使用爬虫、文本转语音、视频生成、浏览器自动化上传等功能时，须**自行确保**符合适用法律法规、各内容/社交平台的服务条款、robots 规则及版权与隐私要求；请勿将本工具用于未经授权的抓取、侵权转载或垃圾信息传播。本仓库与第三方平台**无任何隶属或合作关系**；相关商标与产品名称归各自权利人所有。以上说明不构成法律意见；如有合规疑虑，请咨询专业人士。

---

<div align="center">
  Made with ❤️ by szhshp x 熊猫智研社
</div>
