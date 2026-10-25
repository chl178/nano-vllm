# 学习画板使用说明

本地画板：**http://127.0.0.1:3333**（端口不是 3000，因为 Windows WinNAT 保留了 2964–3063 端口段，bind 会报 EACCES）

工具：[mcp_excalidraw](https://github.com/yctimlin/mcp_excalidraw) v2.1.2（全局安装：`npm i -g mcp-excalidraw-server`，需 Node ≥ 20）

## 核心概念

- **画布是内存态**，重启即清空；`.excalidraw` 文件才是持久层（v2.1.2 尚未实现 README 宣称的 `EXCALIDRAW_DATA_DIR` 重启恢复）
- 画图命令会自动拉起画布服务，无需手动 start

## 常用命令（Git Bash）

```bash
export EXPRESS_SERVER_URL=http://127.0.0.1:3333

# 打开学习笔记
mcp-excalidraw-server import notes/tp-qkv-split.excalidraw

# 画完落盘（然后可 git commit）
mcp-excalidraw-server export --out notes/<名字>.excalidraw

# 查看 / 截图
mcp-excalidraw-server describe
mcp-excalidraw-server screenshot --out /tmp/x.png --format png
```

## AI 直连

ZCode 已配置 MCP server（`~/.zcode/cli/config.json` → `mcp.servers.excalidraw`，下个会话生效）。
之后在任意会话直接说「把 XX 概念画到画板」即可，AI 可调用 26 个工具（元素 CRUD / 对齐分组 / 截图自检 / 快照）。

## 建议工作流

学完一个主题 → 让 AI 画图 → `export` 存入 `notes/` → commit。
每张图 = 一节课的沉淀，`.excalidraw` 是 JSON，可直接 diff。
