<div align="center">
  <img src="./docs/assets/panda-notebooklm.png" width="200" alt="Panda Video NotebookLM Logo">
  
  # Panda Video Automation - NotebookLM Integration

  **熊猫视频自动化生态下的 NotebookLM 笔记转视频自动化工具集**

  [![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
  [![TypeScript](https://img.shields.io/badge/TypeScript-5.9-blue)](https://www.typescriptlang.org/)
</div>

---

<div align="center" style="margin: 12px 0 20px;">
  <a href="https://panda.szhshp.org" title="Panda Video 项目主页" style="display: inline-block; padding: 12px 28px; border: 2px solid #0969da; border-radius: 8px; font-weight: 700; font-size: 1.05em; text-decoration: none; color: #fff; background: #0969da;">🌐 项目主页</a>
</div>

<a id="demo"></a>
## ❇️ 功能演示

<a href="https://www.bilibili.com/video/BV1ZsGU6YEo3/">
  <img src="docs/assets/cover.png" alt="工作流演示 · 点击在 Bilibili 播放" width="480" />
</a>

**🤖 工作流演示** — 工具操作全流程

---

## 上游项目

- V1: 一切的起源-熊猫视频自动化引擎
  - [Panda Video Generator](https://github.com/szhshp/panda-video-generator)
- V2: 已开源-熊猫视频自动化发布工具
  - [panda-video-automations-publisher](https://github.com/szhshp/panda-video-automations-publisher)

## ✨ 核心特性

<table cellspacing="20" cellpadding="0" border="0">
  <tr valign="top">
    <td width="34%" valign="top">
      <h3>📝 <mark>一键</mark> NotebookLM 研究</h3>
      <p>使用 <code>notebooklm-pipeline</code> 技能一站式完成: 创建笔记本, 深度研究, 导入来源, 自动完成 NotebookLM 研究流程. </p>
    </td>
    <td width="33%" valign="top">
      <h3>🎬 <mark>一键</mark>视频生成</h3>
      <p>从 NotebookLM 笔记本一键导出视频制品, 自动下载并准备上传. 全程无需手动操作. </p>
    </td>
    <td width="33%" valign="top">
      <h3>🚀 <mark>一键</mark>多平台发布</h3>
      <p>驱动浏览器自动化上传; B 站, 抖音, 微信视频号, 快手等主流平台全支持. </p>
    </td>
  </tr>
</table>

## 接入方式

<table cellspacing="20" cellpadding="0" border="0">
  <tr valign="top">
    <td width="100%" valign="top">
      <h3>🤖 Agent Skills 方式 (推荐) </h3>
      <ul>
        <li>在 AI Agent 中执行 <code>notebooklm-pipeline</code> 技能, 一行指令完成全流程. </li>
        <li>支持 Claude Code, Cursor, Copilot 等常用 AI Agent. </li>
        <li>首次运行 <code>/setup-pva-notebooklm</code> 技能可自动完成依赖检查和安装. </li>
      </ul>
    </td>
  </tr>
</table>

---

## 📖 简介

把 NotebookLM 里的研究成果一键变成视频, 再自动发到 B 站, 抖音, 视频号——这就是 **Panda Video — NotebookLM**. 

它脱胎于 [Panda Video Generator](https://github.com/szhshp/panda-video-generator) 的视频生产流水线, 上传模块来自 [panda-video-automations-publisher](https://github.com/szhshp/panda-video-automations-publisher). 从创建笔记本, 做研究, 导出视频, 到多平台发布, **一条指令全搞定**. 不用手动下载, 不用反复登录, 不用逐个平台传. 

---

<a id="showcase"></a>
## ❇️ 成品展示

<a href="https://www.bilibili.com/video/BV1StG66bEne/">
  <img src="docs/assets/demo-1.png" alt="成品展示 · 点击在 Bilibili 播放" width="480" />
</a>

**🎬 成品展示** — NotebookLM 成品视频示例

---

## 🚀 快速开始

向 Agent 发送以下指令即可完成完整的工作流:

### 一站式流程 (推荐) 

```text
/notebooklm-pipeline 帮我做一个关于「AI 泡沫还能持续多久?」的视频并发布到 B 站和抖音
```

Agent 会自动执行: 深度研究 → 视频生成 → 裁剪 → 封面 → 元数据 → 发布, 全程无需人工干预. 

### 初次使用

如果首次使用, 先运行:

```text
/setup-pva-notebooklm
```

这会自动安装 `notebooklm-py` , `@panda-video-automation/pva` 等所有依赖, 并引导完成 NotebookLM 登录. 

### 先决条件

- **Python 3**
  — [notebooklm-py](https://pypi.org/project/notebooklm-py/)
- **Node.js 20+**
  — [@panda-video-automation/pva](https://www.npmjs.com/package/@panda-video-automation/pva)
- **ffmpeg**
  — 视频裁剪 ( `brew install ffmpeg` / `apt install ffmpeg` ) 

---

## 🧩 可用技能

| 技能 | 说明 |
|------|------|
| `notebooklm-pipeline` | **[入口]** 全流程一站式: 研究 → 视频 → 裁剪 → 封面 → 元数据 → 发布 |
| `setup-pva-notebooklm` | 检查并安装所有前置依赖 (Python, Node.js, notebooklm-py, PVA) |
| `notebooklm-status` | 查看所有笔记本及制品状态 |
| `notebooklm-research` | 创建笔记本并执行深度研究 (子步骤) |
| `notebooklm-video` | 从笔记本生成视频 (子步骤) |
| `get-notebooklm-video` | 下载视频制品 (子步骤) |
| `notebooklm-prep-upload` | 准备上传内容到 input/ 文件夹 (子步骤) |
| `minimalist-academic-cover` | 生成极简学术风格封面 (子步骤) |
| `pva-publisher` | 跨平台视频上传 (子步骤) |

> 提示: 大多数场景下只需使用 `notebooklm-pipeline` 一个技能即可完成全部工作. 其他技能适用于需要手动控制其中某个环节的场景. 


---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request!

---

## 📄 许可证

本项目采用 MIT 许可证. 详见 [LICENSE](LICENSE) 文件. 

---

## 👤 作者

**szhshp**

- Email: 24031shp@sina.com
- GitHub: [@szhshp](https://github.com/szhshp)

---

## ⚠️ 免责声明

本项目按「原样」提供, 作者不对因使用本软件而产生的任何直接, 间接或附带损失承担责任. 你在使用爬虫, 文本转语音, 视频生成, 浏览器自动化上传等功能时, 须**自行确保**符合适用法律法规, 各内容/社交平台的服务条款, robots 规则及版权与隐私要求; 请勿将本工具用于未经授权的抓取, 侵权转载或垃圾信息传播. 本仓库与第三方平台**无任何隶属或合作关系**; 相关商标与产品名称归各自权利人所有. 以上说明不构成法律意见; 如有合规疑虑, 请咨询专业人士. 

---

<div align="center">
  Made with ❤️ by szhshp x 熊猫智研社
</div>
