# dsh-purge fix.1 — DSH 0.2.1 兼容修复

本仓库是 **dsh-purge 1.1.72** 的 MIT 源码快照（上游 GitHub 仓库已转为私有），
针对 DeepSeek Harness **0.2.1-rc.2** 做了最小修复，解决社区反馈的
`unknown prompt variable "{{cwd}}"` 报错。

## 症状

DSH 更新到 0.2.1 后，每次新建会话 / 发消息都出现：

```
本轮运行失败 unknown prompt variable "{{cwd}}" in section "deployment:persona-suffix";
registered variables: provider, model
```

purge 面板显示「处理失败」，整轮清洗被取消，破甲注入全部失效。

## 根因

0.2.1 的 `deployment:persona-suffix` 段只注册了 `provider`、`model` 两个插值变量，
`cwd` 已被移除。而 purge 有三处会把 `{{cwd}}` 写进系统提示：

1. `lib/identity.js` — `CWD_ONLY_SUFFIX` 常量（清洗时主动写回 persona-suffix 段）
2. `lib/core.js` — `ALLOW_EXEC_PERSONA_BLOCK` / `ALLOW_EXEC_PERSONA_LINE`（替换官方身份段）
3. `cordis.patch.yml` — `personaSuffix` 默认值

只要其中任意一处生效，`interpolate()` 就会在渲染系统提示时抛错，整轮组装失败。

## 修复

把上述三处写入侧的 `{{cwd}}` 换成不含插值变量的字面量：

| 文件 | 改动 |
|---|---|
| `lib/identity.js` | `CWD_ONLY_SUFFIX = "Your working directory is the current workspace."` |
| `lib/identity.js` | 两处 `"Working directory: {{cwd}}. "` → `"Working directory: the current workspace. "` |
| `lib/core.js` | `ALLOW_EXEC_PERSONA_BLOCK` / `ALLOW_EXEC_PERSONA_LINE` 同步替换 |
| `cordis.patch.yml` | `personaSuffix: ""` |

**保留**：`isCwdOnlyText()` 里的 `"Working directory: {{cwd}}."` 比对字符串——它只用于识别历史文本，不会被写入提示。

`{{model}}` 与 `{{REDTEAM_SKILLS_DIR}}` 不动：前者是 0.2.1 已注册变量，后者是构建期占位符。

## 安装

```bash
# 从本地目录
pnpm add file:/path/to/dsh-purge-fixed

# 或打成 tarball 后安装
pnpm pack
pnpm add file:./dsh-purge-1.1.72-fix.1.tgz
```

Windows 上若 `pnpm` 报 `ERR_PNPM_UNEXPECTED_STORE`，追加 `--store-dir` 指向现有 store。

## 已知未修复

- `official-update` 的 Windows 启动器（`.cmd`）用 UTF-8 写中文路径，`cmd.exe` 按 GBK 解析会乱码，
  导致官方更新卡在「正在准备重启…」。临时绕过：更新前手动关闭 purge 的更新接管，或改名脚本目录到纯 ASCII 路径。
- 上游仓库已私有，`update.js` 的 GitHub 自更新链路失效，请改用本仓库的 `file:` 依赖。

## 许可证

MIT，版权归原作者（小杨 / YuJunZhiXue）。本仓库仅做兼容性修复，未改变授权。
