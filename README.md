<div align="center">
  <img src="docs/icon.png" alt="App Icon" width="100" />

  <h1>rikkahub-for-subagent</h1>

一个基于 [RikkaHub](https://github.com/rikkahub/rikkahub) 的 Android LLM 聊天客户端 **fork**。

在这个 fork 里，**子代理（Subagent）是一等公民**：主 agent 可以把任务委派给独立角色的子代理 —— 单个、一次并行多个、或者串成链；也可以直接丢到后台跑，跑完自动回调汇报，主对话不用干等。

独立包名 `me.linklink256.rikkahub`，可与官方版共存。

</div>

---

## 📥 下载

| 渠道 | 说明 |
|:---|:---|
| [Releases · `subagent-release`](https://github.com/linklink256/rikkahub-for-subagent/releases) | 子代理版，master 有新提交时自动构建（Pre-release） |
| [Releases · `nightly`](https://github.com/linklink256/rikkahub-for-subagent/releases) | 每日构建，每天有提交才打包 |
| 自行编译 | 见文末「构建」 |

> 开发版本，可能不稳定，仅供测试。

---

## 🧩 核心能力：子代理

主 agent 通过 `subagent` 工具把任务交出去。子代理跑在**独立会话**里：用自己的 system prompt、自己的工具白名单、可以指定自己的模型，**不继承主对话历史**——只拿到一个最小上下文包。

### 三种委派模式

| 模式 | 行为 |
|:---|:---|
| `single` | 一个角色干一个任务 |
| `parallel` | 同一角色扇出多个子任务，**最多 8 个、并发 4**，各自独立日志 |
| `chain` | 顺序 handoff，**上一步的结果自动注入下一步的 context** |

委派时可以给出的约束：`task`（任务本身）、`boundary`（不许做什么）、`context`（从对话里抽取的最小上下文）、`acceptance`（验收标准）。

### 后台任务 + 完成回调

给 `subagent` 加 `background: true`，调用会**立刻返回一个 `taskId`**，主 agent 可以继续和用户聊天；子代理跑完后，结果以 `[Background Task Callback]` 消息**自动注入原对话**并触发主 agent 汇报。

- 回调做了 1 秒防抖合并，并会等在途生成结束（上限 60 秒）再注入，避免打断正在生成的回复
- 用 `background_tasks` 工具可以 list / get / cancel 正在跑和已完成的后台任务
- 同样适用于工作区命令：`workspace_shell` 也支持 `background: true`，长命令走一次性进程，不会占住常驻 shell 的锁

### 自定义角色：一个 `AGENT.md` 就是一个子代理

角色定义放在应用私有目录：

```
files/agents/<角色名>/AGENT.md          # 角色本体（frontmatter + system prompt 正文）
files/agents/_groups/<组名>.md          # 可选：分组说明（frontmatter 只需 name + description）
```

frontmatter 支持字段：

```markdown
---
name: scout
description: 只读调研，压缩调查结果
group: research              # 可选，分组名；不填归入默认组
tools: workspace_read_file, workspace_shell, search   # 可选；tools: none = 禁用全部；不声明 = 继承全部工具池
model: openai:gpt-4o         # 可选，支持 '供应商:模型ID' / '供应商/模型ID' / 裸模型ID；不填继承主 agent
reasoningLevel: high         # 可选，off/auto/low/medium/high/xhigh/max，默认 off（兼容 reasoning 键）
resultFormat: Summary, Findings, Risks   # 可选，自定义输出契约段落名
maxSteps: 30                 # 可选，最大工具调用轮数，默认 30
stepTimeout: 120             # 可选，单步工具执行超时（秒），默认 120
toolOutputLimit: 20000       # 可选，单条工具输出截断字符数，默认 20000；<=0 不截断
streaming: false             # 可选，默认 false（非流式更省时）
timeout: 300                 # 可选，任务总超时（秒），默认 600
---
<角色 system prompt 正文>
```

在 **设置 → 扩展管理 → Subagents** 里管理角色；也可以让 agent 直接改工作区里的 `AGENT.md`，工具描述与系统提示里的 `<available_subagents>` 每轮动态重读，不会用到过期快照。

### 跑得稳、不空耗

- **三步防跑空**：`maxSteps` 步数上限 / 同一工具同一参数连续 5 次判为死循环并停止 / 单步工具超时不让整个任务崩掉
- **结果不膨胀**：超长工具输出按 `toolOutputLimit` 截断并提示定向读取；空结果自动重试
- **角色级记忆**：`/workspace/.cache/subagent-memory/<角色名>.md` 跨任务持久，并行写入用锁串行化，不会互相覆盖
- **默认非流式生成**：子代理的中间流式文本本来就没人消费，改掉后省下大量消息重建开销

---

## ⚡ 工作区（proot）

- **常驻 shell**：每个工作区只起一个长期存活的 proot + bash 进程，命令走 stdin 哨兵协议；不再每条命令都新起一次 PRoot（Android 上每次 1~3 秒）。stderr 落固定文件、用 bash 内建 `$(< file)` 读回，协议尾零 fork
- 会话隔离（子 shell 包住 `cd`/`export`）、每会话命令串行、超时看门狗杀进程树、会话死了重建一次再退化到一次性执行
- 同轮多个工具调用**并行执行**，不再顺次排队；文件写入直接走 JVM IO，不再绕 PRoot
- 工作区里带完整工具链（`workspace_shell` / `workspace_read_file` / `workspace_write_file` 等），本身就是个 Linux agent 环境

## 🔧 本地工具

- 全盘文件访问（All Files Access）：`read_file` / `write_file` / `list_directory` / `file_info` / `delete_file` / `move_file` / `create_directory` / `copy_file` / `search_files` / `get_system_info`，操作绝对路径；Android 11+ 首次使用会引导去授权
- reasoning effort 会按厂商实际规则显示生效值（例如 DeepSeek 系列对 `xhigh` / `max` 的映射），不会给了参数却发不出去

## 🧬 继承自上游 RikkaHub 的能力

多供应商（OpenAI / Google / Anthropic 兼容接口，可自定义 API / URL / 模型） · 多模态输入（图片 / 文本文件 / PDF / Docx） · MCP · 工作区 · 搜索（Exa / Tavily / 智谱 / LinkUp / Brave / Perplexity 等） · Markdown 渲染（代码高亮 / LaTeX / 表格 / Mermaid） · 消息分支 · 提示词变量 · ChatGPT 式记忆 · AI 翻译 · 自定义请求头与请求体 · SillyTavern 角色卡导入 · Web 端访问 · Material You 与深色模式

---

## 🛠 构建

环境：JDK 17、Android SDK、Node 22 + pnpm 11（`web` 模块的 preBuild 会跑前端构建）。

```bash
# 1) web-ui 依赖（buildWebUi 只 build 不 install，需要先装）
cd web-ui && pnpm install --frozen-lockfile && cd ..

# 2) 签名配置（local.properties）
#    storeFile / storePassword / keyAlias / keyPassword

# 3) google-services.json 必须存在于 app/ 下（可用占位文件绕过 Firebase 校验）

# 4) 打包
./gradlew :app:assembleRelease     # APK
./gradlew :app:bundleRelease       # AAB
```

CI：`.github/workflows/build-check.yml` 在 master 有新提交时构建并把 APK 发到 pre-release；`daily-build.yml` 每天检查有无新提交，有才打包。

## 📄 许可与上游

- 本项目 fork 自 [rikkahub/rikkahub](https://github.com/rikkahub/rikkahub)，遵循 **[AGPL-3.0](LICENSE)** 发布；上游版权归原作者及贡献者所有，本仓库的修改同样以 AGPL-3.0 开源。
- 上游的官网、应用商店、赞助与社群渠道与本 fork 无关；使用本 fork 遇到的问题请提到**本仓库的 Issues**。
- 上游明确不接受新特性类 PR，因此本 fork 的这类改动不会回流上游。
