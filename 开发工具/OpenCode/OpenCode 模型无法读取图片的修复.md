---
title: OpenCode 模型无法读取图片的修复
type: guide
permalink: my-kb/notes/open-code-模型无法读取图片的修复-1
tags:
- opencode
- multimodal
- 配置
- 图片输入
created: 2026-08-19
---

# OpenCode 模型无法读取图片的修复

## 问题

在 OpenCode 里给 GPT/Claude 附上 PNG/JPG 图片，模型回复"无法读取图片"或只提到文件名。同一张图直接在 ChatGPT/Claude 中却正常。模型其实支持视觉，问题在 OpenCode 未把图片作为多模态输入发送。

## 根因

OpenCode 逐个模型决定能否发送附件及是否可为图片，信息来自 opencode.jsonc 的 provider 配置。若模型条目只有 `"name"`，OpenCode 便无信号表明其支持图片，于是把附件当作普通文件引用（如 `[attachment: diagram.png]`），丢掉了图像像素。

## 快速修复

模型条目加两个字段：

```json
"gpt-5.5": {
  "name": "gpt-5.5",
  "attachment": true,
  "modalities": {
    "input": ["text", "image"],
    "output": ["text"]
  }
}
```

- `"attachment": true`：允许接收附件
- `"modalities": { "input": ["text", "image"] }`：声明接受图片输入，OpenCode 会将文件 base64 编码为多模态 `image_url` 内容块

配置位于 `~/.config/opencode/opencode.jsonc`（Windows 为 `C:\Users\<username>\.config\opencode\opencode.jsonc`）。可只给某个模型单独开启。

## 注意事项

- **需重启 OpenCode**：配置在启动时加载，重启或新建会话才生效。
- **本地上传优先**：本地 PNG/JPG 会转成 base64 data URL，最可靠；公网图片 URL 可能因大小/MIME/防盗链被上游拒绝，报 `Error while downloading file. Upstream status code: 400.` 时应改用本地文件。

## Observations

- [insight] 模型报"不支持图片"通常是客户端（OpenCode）配置问题，而非模型或 API 限制
- [decision] 视觉模型统一在 opencode.jsonc 中声明 `attachment: true` + image modality 即可启用图片输入
- [fact] gpt-5.5、claude-sonnet-4-6、claude-opus-4-7、claude-opus-4-8、claude-haiku-4-5 均可经 OpenAI 兼容 Chat Completions API 接收图片