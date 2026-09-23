# agent-asset

可复用的 AI Agent 技能（Skills）与规则集合。

每个技能是一个独立目录，遵循 Agent Skills 规范：目录内的 `SKILL.md` 用 YAML frontmatter 声明 `name` 与 `description`，Agent 依据 `description` 中的描述与触发词，在合适的时机自动加载对应技能；也可以显式调用。

## 技能一览

| 技能 | 作用 | 典型触发场景 | 前置条件 |
| --- | --- | --- | --- |
| [`closed-choice-resolver`](./closed-choice-resolver/SKILL.md) | 遇到封闭集合选择时，把问题或当前状态转成英语交给 TypeSafe 的 Jev 模型判断，返回概率最高的选项／动作及校准概率与置信度 | 模态框、确认弹窗、权限对话框、交互式 CLI 菜单挡住了流程；选项数量确定的选择题；可穷举的离散动作空间（游戏操作、固定按钮集、固定命令集） | `TYPESAFE_API_KEY` |
| [`typesafe-ai`](./typesafe-ai/SKILL.md) | 用 TypeSafe 构建带判断能力的软件：把自然语言与应用状态转成 `Choice` / `Noul` / `Score` 三类类型化判定，由代码负责流程与组合 | 需要可编程的常识判断；把「LLM 提示词 + 解析」的步骤改造成结构化决策；路由、排序、抽取、校验、交互场景 | `TYPESAFE_API_KEY` |

## 目录结构

```text
agent-asset/
├── .agent/
│   └── AGENTS.md                 # 全局规则：术语注释、文件异议处理、C/C++ 编码规范等
├── closed-choice-resolver/
│   └── SKILL.md                  # 封闭集合选择消解
├── typesafe-ai/
│   └── SKILL.md                  # 用 TypeSafe 构建 AI 能力
└── README.md
```

## 安装

克隆仓库后，把技能目录复制到 Agent 的技能搜索路径（以 Windows 下 `~/.agents/skills/` 为例）：

```powershell
git clone https://github.com/lemonade2k17/agent-asset.git
Copy-Item -Recurse .\agent-asset\closed-choice-resolver "$env:USERPROFILE\.agents\skills\"
Copy-Item -Recurse .\agent-asset\typesafe-ai "$env:USERPROFILE\.agents\skills\"
```

重新开始会话后即可按描述自动匹配，或用 `/closed-choice-resolver`、`/typesafe-ai` 显式调用。

## 配置

两个技能都通过环境变量 `TYPESAFE_API_KEY` 访问 TypeSafe API，密钥在 <https://console.typesafe.ai/keys> 领取：

```powershell
setx TYPESAFE_API_KEY "你的密钥"
```

`setx` 写入的是用户级环境变量，只对新启动的进程生效——已运行的进程（及其子进程）读不到，需要新开终端或新会话。两个技能内部都已按「先读进程环境变量、为空则回退读取用户级变量」的顺序取值，不必手动处理这一差异。

## 编写约定

- 一个技能一个目录，目录内必须包含 `SKILL.md`；
- `SKILL.md` 使用 YAML frontmatter，至少包含 `name` 与 `description`；
- `description` 要同时写清「做什么」与「何时触发」，并附典型触发词与中英文说法，Agent 靠它决定是否加载；
- 正文写具体步骤、适用边界、安全红线与失败处理，避免依赖隐含上下文；
- 技能内引用外部文档时优先给可直接读取的链接。

## 安全边界

- **权限与授权类弹窗一律不代决**：系统权限申请、证书信任、提权、杀软／防火墙放行等安全边界授予行为，必须由用户本人确认；
- **高风险操作交回用户**：不可逆或对外产生后果的动作（删除、覆盖、发布、发送、支付、消耗不可再生资源等），无论模型给出的置信度多高，都先询问用户并说明原因；
- **密钥只放环境变量或服务端**：`TYPESAFE_API_KEY` 不要写进文件、日志、回复或提交历史；技能产生的临时截图用完即删。
