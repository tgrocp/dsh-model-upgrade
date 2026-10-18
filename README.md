# dsh-model-upgrade

把 DeepSeek Harness（DSH）的模型目录与默认模型升级到最新 DeepSeek 模型的 **Skill**。

- 通过 `$DSH_HOME/settings.yaml`（默认 `~/.dsh/settings.yaml`）热更新，**无需重启、无需改动安装包**
- 覆盖完整流程：配置分层原理 → 确认最新模型 ID → 修改设置 → schema 校验 → 验证 → 回滚
- 附带 2026-10 实战案例：API 实际支持 `deepseek-flash` / `deepseek-v4-pro`（教训：以 API 报错列出的支持名称为准，网上流传的 `deepseek-v4.1-*` 不可用）

## 安装

把本仓库放进任意 DSH skill 根目录的 `dsh-model-upgrade` 文件夹即可（DSH 热发现，无需重启）：

```bash
mkdir -p ~/.dsh/skills
git clone https://github.com/tgrocp/dsh-model-upgrade.git ~/.dsh/skills/dsh-model-upgrade
```

也可放到项目根的 `.dsh/skills/` 或 `.agents/skills/`（仅该项目的会话可见）。

## 触发方式

直接对 agent 说：

- "更新 / 升级 DSH 的模型到最新版"
- "接入最新的 DeepSeek 模型（如 deepseek-flash）"
- "模型选择器里缺少新模型"

## 文件

- `SKILL.md` — skill 本体：升级流程与校验规则
- `README.md` — 本说明
- `LICENSE` — MIT
