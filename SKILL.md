---
name: dsh-model-upgrade
description: 把 DeepSeek Harness（DSH）的模型目录与默认模型升级到最新 DeepSeek 模型——通过 $DSH_HOME/settings.yaml 热更新，无需重启、无需改动安装包
whenToUse: 当用户要求"更新/升级 DSH 的模型到最新版"、"接入新发布的 DeepSeek 模型（如 V4.1 Flash / Pro）"、或 Web 模型选择器里缺少新模型时使用
---

# DSH 模型升级

把 Harness 的模型目录（Web 模型选择器列表）和默认模型升级到最新的 DeepSeek 模型。

## 背景：配置分层（先读懂再动手）

模型配置由三层组成，后面的层覆盖前面的层：

1. **安装包内置默认值（不要改）**
   - `dsh-base` bundle 补丁里的 `agent-default-model` 行：`{ provider: deepseek-official, model: deepseek-v4-flash }`，是新安装的默认模型。
   - `@deepseek-ai/dsh-llm-deepseek` 的 `DEFAULT_MODELS`（`lib/index.js`）：模型目录默认值 `deepseek-v4-flash` / `deepseek-v4-pro` / `deepseek-v4-flash-vision-exp`。升级 DSH 包时这些会被覆盖，因此用户级配置才是设计好的覆盖面。
2. **用户设置文档 `$DSH_HOME/settings.yaml`（默认 `~/.dsh/settings.yaml`）——本次要改的只有它**
   - `agent-default-model:` 段覆盖默认模型（新建会话时读取；已有会话的模型在会话创建时已固定）。
   - `llm-deepseek:` 段覆盖 DeepSeek 适配器配置，其中 `models:` 数组**整体替换**默认目录（数组不合并）。
   - 该文件**热加载**：修改后无需重启进程；Web 的 Models 设置页读写的就是这两个段。
3. profile 的 `cordis.patch.yml` 与 `--patch` 覆盖层（一般不需要动）。

解析规则（`dsh-settings`）：schema 默认值 → 插件组合 base → 用户段；对象递归合并、**数组整体替换**。用户段校验失败时保留"最后一份好配置"并在日志告警，不会崩溃。

## 步骤

### 1. 确认要升级到的最新模型 ID

- 先查官方文档确认**精确**的 API model id（DeepSeek API Docs「Models & Pricing」/更新日志：https://api-docs.deepseek.com/quick_start/pricing ）。适配器是透传的：id 原样发给 API，写错 id 会在调用时 400。
- **以 API 实际响应为准**：拿不准时发一次小请求，直接看报错里的 "The supported API model names are …" 列表——这是最权威的清单。
- 实战案例（2026-10 教训）：网上/文档写的是 `deepseek-v4.1-flash`，但 API 实际只接受 `deepseek-flash` 与 `deepseek-v4-pro` 两个名称（`deepseek-v4.1-*` 全部 400）。网上流传的 id 不一定等于 API 支持的 id，必须实测确认。

### 2. 读当前设置

```bash
echo $DSH_HOME          # 不是 ~/.dsh 时以它为准
cat ~/.dsh/settings.yaml
```

### 3. 改 settings.yaml

```yaml
agent-default-model:
  provider: deepseek-official
  model: deepseek-v4-pro            # 新默认模型（保留用户原来的档位）
  reasoningEffort: max              # off | low | high | max，可选

llm-deepseek:
  models:
    - id: deepseek-flash            # 只列 API 实际支持的名称（2026-10）
      name: DeepSeek-Flash
      contextWindow: 1000000
    - id: deepseek-v4-pro
      name: DeepSeek-V4-Pro
      contextWindow: 1000000
```

### 4. 校验规则（对照适配器 schema）

- `models` 数组整体替换：想保留的模型必须**全部列出**，id 目录内唯一。
- 每个条目：`id` 必填、非空；`name` 非空；`contextWindow` 正整数；`inputModalities` 只允许 `text`/`image` 且不重复。
- 图片参数（`imagePixelBudget` / `imageMaxBytes` / `imageDetail`）**只能**出现在 `inputModalities` 含 `image` 的条目上。
- 若 API 支持列表里有视觉模型，保留其条目（删掉后图片输入会报 "does not accept image input"）；若支持列表里没有（2026-10 现状），目录就不该有视觉条目。
- `agent-default-model` 段不校验 id 是否在目录里——第 1 步确认 id 真实存在很重要。
- `baseURL` / `apiKeyEnv` 有默认值不用写；API key 走凭据存储，与本次升级无关。

### 5. 验证

- 改完重读文件确认 YAML 结构无误。
- **无需重启**：文件 watcher 热加载。检查日志没有 "keeping last good … after invalid stored section" 告警（`~/.dsh/logs`）。
- Web 模型选择器里应出现新模型；新建会话使用新默认模型；**当前会话模型不变**（改用 `/model` 切换或新开会话）。

## 回滚 / 兜底

- API 报 "model not found"（或列出 supported names）：把 `agent-default-model.model` 换成 API 列表里的 id（2026-10 可用：`deepseek-v4-pro` / `deepseek-flash`），热加载立即生效。
- 想恢复出厂默认：删除 settings.yaml 里的 `llm-deepseek` 段与 `agent-default-model` 段（或用 Models 页面重置），即回到安装包默认值。
- 不要改 node_modules 里的 `DEFAULT_MODELS` / bundle 补丁：升级时会被覆盖，且不属于用户配置面。

## 已知坑

- 改配置**不能**切换当前会话的模型（per-session 固定），只能影响新会话。
- Web Models 页面保存时会用页面状态**重写** `llm-deepseek` 段，可能覆盖手工改的 `models` 列表——手工改文件与页面操作二选一。
- 用户段校验失败不报错，只回退旧值并写日志；升级后务必看日志或打开选择器确认。
- `contextWindow` 填官方文档数值（V4 系当前 1e6 = 1000000），不要猜。
- 网上流传的 id（如 `deepseek-v4.1-flash`）可能不被当前 API 接受；先实测或看 API 报错里列出的支持列表，再写进配置。
