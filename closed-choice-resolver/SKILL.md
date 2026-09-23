---
name: closed-choice-resolver
description: >-
  当遇到"封闭集合选择"时使用：①模态框、确认弹窗、权限对话框、CLI 交互式选择菜单；②一切选项数量确定的选择题；③主模型识别出可穷举的操作空间（如游戏"上/下/左/右/技能1/技能2"、固定按钮集、固定动作集）。出现"模态框挡住了""弹窗让我选一个""这题选哪个""下列…正确的一项是""该按哪个键""下一步做什么""blocked by a modal""a dialog is asking me to choose""multiple choice"等情形时使用。把问题或当前状态翻译成英语作为 state（只翻译，不添加任何提示、法则、策略、理由、结论或分析），选项／动作作为 criteria，交给 TypeSafe 的 Jev 模型判断（type 默认 choice），返回概率最高的选项或操作及校准概率与 confidence——直接采用为目标，主模型不再验证；仅当选中项属高风险操作（不可逆、对外产生后果、权限授权等）时询问用户并说明高风险原因。
---

# 封闭集合选择消解（Closed Choice Resolver）

遇到**封闭集合选择**时，用 TypeSafe 的 Jev 模型在已有选项之间做判断，把"该选哪个／该做什么"变成一个带概率和置信度的结构化答案。

所谓封闭集合选择，指答案必然是**已经列出的那几个选项之一**，没有生成空间。本 skill 覆盖三类：

1. **阻塞型**——模态框、确认弹窗、权限对话框、CLI 交互式选择菜单，必须先选中一个选项才能继续；
2. **题目型**——一切选项数量确定的选择题：单选、多选、判断、选项固定的问卷与测评；
3. **动作型**——主模型识别出当前可操作的内容是一个**可穷举的离散集合**，需要选出下一步操作。例如游戏中可选的"上／下／左／右／技能1／技能2"，或固定按钮集、固定动作集、固定命令集。

**这个 skill 只负责"决定选哪个／做什么"。** 真正的点击、按键、作答由你原有的手段完成（自动化脚本、或把答案交给用户）。它不负责 UI 自动化，也不负责提交。

**核心约定**：Jev 返回的 `choice` 就是目标选项，**主模型直接采用，不再验证**。唯一的例外是**高风险操作**——那种情况必须询问用户并说明原因。

## 前置条件

API key 必须可用。按以下顺序取——**不要因为环境变量为空就断定密钥不存在**：Windows 上进程的环境块在进程启动时固定，`setx` 之后已经运行的进程（及其所有子进程）读不到该变量，但它其实已经设置好了。

```powershell
$key = $env:TYPESAFE_API_KEY
if (-not $key) { $key = [Environment]::GetEnvironmentVariable('TYPESAFE_API_KEY','User') }
if (-not $key) { Write-Output "TYPESAFE_API_KEY NOT SET" }
```

两处都为空时不要往下走，也不要试图从别处寻找密钥——直接告诉用户去 <https://console.typesafe.ai/keys> 领取并设置：

```powershell
setx TYPESAFE_API_KEY "你的密钥"    # 用户级，需新开进程才生效
```

## 何时使用

- 一个模态框／弹窗／向导挡住了后续操作，必须先选一个选项；
- 交互式 CLI 停下来等你在菜单里选（`[1] 覆盖 [2] 跳过 [3] 全部覆盖`）；
- 应用弹出确认框问"是否继续"；
- **任何选项数量确定的选择题**：单选、多选、判断、选项固定的问卷／测评；
- 用户把一道选择题丢给你，问"选哪个"；
- **主模型发现当前操作空间是可穷举的离散集合**，需要决定下一步做什么——游戏操作、固定按钮界面、固定命令集、回合制交互。

## 何时不要使用

- **权限／授权／安全类弹窗**（系统权限申请、证书信任、提权、UAC、杀软警告）——见下方安全红线，一律交回用户；
- 选项是自由文本输入、路径、命令，而不是封闭集合；
- **动作空间不是离散可穷举的**——连续控制（摇杆、鼠标移动、滑块、自由视角）不适用；本 skill 只处理"可枚举的选项集"；
- **实时性要求高于亚秒级**的动作场景——每次判定要走一次网络请求，跟不上需要毫秒级反应的实时操作（见下方"动作型的适用边界"）；
- 你能从任务上下文直接确定该选什么——那就不用调用模型；
- 当前任务本身明确要求"遇到选择必须问用户"。

## 流程

### 第 1 步：取得问题或当前状态

**阻塞型**：优先只截**前台窗口**（体积小、文字清晰，多显示器下不会把弹窗淹没在整屏里）：

```powershell
$path = Join-Path $env:TEMP "dsh-closed-choice-shot.png"
Add-Type -AssemblyName System.Drawing
Add-Type @"
using System;
using System.Runtime.InteropServices;
public class W32 {
  [DllImport("user32.dll")] public static extern IntPtr GetForegroundWindow();
  [DllImport("user32.dll")] public static extern bool GetWindowRect(IntPtr hWnd, out RECT r);
  [StructLayout(LayoutKind.Sequential)] public struct RECT { public int Left; public int Top; public int Right; public int Bottom; }
}
"@
$h = [W32]::GetForegroundWindow(); $r = New-Object W32+RECT
[void][W32]::GetWindowRect($h, [ref]$r)
$w = $r.Right - $r.Left; $ht = $r.Bottom - $r.Top
if ($w -le 0 -or $ht -le 0) { throw "cannot read foreground window rect" }
$bmp = New-Object System.Drawing.Bitmap $w, $ht
$g = [System.Drawing.Graphics]::FromImage($bmp)
$g.CopyFromScreen($r.Left, $r.Top, 0, 0, $bmp.Size)
$bmp.Save($path, [System.Drawing.Imaging.ImageFormat]::Png)
$g.Dispose(); $bmp.Dispose()
Write-Output $path
```

若前台窗口不是那个弹窗（例如你在终端里调用、弹窗在别的显示器），改截**整个虚拟桌面**——把上面 `GetWindowRect` 那段换成：

```powershell
$r = [System.Windows.Forms.SystemInformation]::VirtualScreen   # 需先 Add-Type -AssemblyName System.Windows.Forms
$w = $r.Width; $ht = $r.Height
$g.CopyFromScreen($r.Left, $r.Top, 0, 0, $bmp.Size)
```

然后用 `read_image` 工具**实际看一眼这张图**。不要跳过这一步凭猜测构造描述。

**题目型**：题目已经是文本就直接用；是图片就同样先 `read_image` 看清。

**动作型**：同样先截当前画面（或用界面暴露的状态接口）并用 `read_image` 看清，然后**先确定动作集**——把"现在到底能做什么"枚举出来。枚举不全就没有封闭集合，也就没有本 skill 的用武之地。

### 第 2 步：转成英语——只翻译，不添加任何东西

`state` 必须是与原文逐字对应的英语，不做增删。**三种情形的 `state` 分别包含**：

| 情形 | `state` 应包含 | `state` 不应包含 |
|---|---|---|
| 阻塞型 | 弹窗标题 + 提示正文 + 全部选项文字 | 你当时正在做什么、你的目标 |
| 题目型 | 题干 + 全部选项文字 | 任何法则、解题思路、提示 |
| 动作型 | 当前可观察状态的**客观描述** + 任务目标 | 实现该目标的具体策略、理由、结论 |

**严禁加入**（以下每一项都会污染判断，等于把答案"喂"给模型）：

| 禁止项 | 例子 |
|---|---|
| 法则、定理、公式、定义 | "同底数幂相乘指数相加"、"这类题的判据是…" |
| 解题／决策思路、推理、中间步骤 | "先把两项合并，再看指数…"、"先撤到安全距离再反击" |
| 倾向性提示 | "注意符号"、"这题考的是积的乘方"、"此时应优先自保" |
| 你自己的结论或猜测 | "我认为选 C"、"正确答案应该是…"、"建议使用技能2" |
| 与判断无关的上下文 | "我正在做导出"、"这个文件不需要保留" |
| 原文没有的任何信息 | 补充的条件、假设、澄清 |

**为什么必须这样**：一旦加入提示，模型返回的高置信度就不再是"独立判断"，而只是"复述你给的提示"。你会误以为得到了验证，实际只是自我确认——这比不调用模型更危险。

**只翻译，不解释**：把文字译成英语即可。**不要**为了让模型"更容易答对"而改写、简化、补充或澄清。原文含糊就保持含糊——含糊本身就是判断所需的真实信息。

**动作型的唯一例外**：允许写**任务目标**（例如 "Objective: clear the current level."），因为没有目标就无法判断动作优劣。但**不得**写实现该目标的具体策略、理由或结论——那仍然是污染。

> **转录准确性是唯一的正确性来源。** 因为第 6 步不再有验证环节，`state` 的翻译质量直接决定结果对错。抄错一个符号、漏掉一个选项，Jev 会**同样自信地**给出错误答案，而且不会有任何低置信度信号提示你。

### 第 3 步：自适应选择 type

| 情形 | type | 理由 |
|---|---|---|
| **默认**：选项／动作互斥、无序、可枚举（按钮列表、单选、判断、游戏动作） | `choice` | 绝大多数情形属于这类；`choice` 直接映射回一个选项或动作 |
| 选项本身构成**有序档位**（如 重试 1 次／重试 3 次／放弃，低／中／高风险，移动 1 格／2 格／3 格） | `score` | 模型返回概率加权位置，靠 `legend` 映射回档位 |
| 真正要问的是**一个是非命题**，而不是"选哪个"（如"这次操作是否会写入磁盘？"） | `noul` | 返回"是"的概率，没有 `confidence` 字段 |

两条硬性约束：

- **最终要落到"点哪个按钮／选哪个选项／按哪个键"时，必须用 `choice`。** `score` 返回的是可落在两档之间的浮点数，无法直接对应一个动作，除非动作本身就是递进档位。
- **不要因为选项有 3 个以上就用 `score`。** 互斥离散类别一律用 `choice`；只有选项之间真的存在"程度递进"才用 `score`。

**多选题**：若题目允许多个正确选项，用 `choice` 逐项问（每项一个 `noul`，问"这一项是否正确"），不要试图用一个 `choice` 选出多项。

### 第 4 步：组装 criteria

`criteria` 是**选项／动作清单**，不是提示，可以也应该原样保留：

- **键**：**一律使用纯 ASCII 标识符**，不得含中文或任何非 ASCII 字符——选择题用 `A`/`B`/`C`/`D`，模态框用 `REPLACE`/`KEEP_BOTH`/`CANCEL` 这类 ASCII 短名，游戏动作用 `UP`/`DOWN`/`LEFT`/`RIGHT`/`SKILL1`/`SKILL2`。键只负责**唯一标识**，不负责携带原文；
- **值**：该选项／动作的英语翻译或简短英语释义，说明"选它会发生什么"；文字本身已是英文且足够清楚时可以给 `null`。

**为什么键名必须是 ASCII**：Jev 会把你给的键**原样回传**，而响应体在 Windows PowerShell 5.1 上默认按 ISO-8859-1 解码，中文键会变成乱码（实测 `置顶` → `ç½®é¡¶`），`choice` 因此对不回真实按钮；而第 6 步规定"直接采用、不再验证"，这个错误会**静默**传递到点击动作上。实测键名的文字本身还会影响概率分布（同一题目下中文键与 ASCII 键的 top 概率相差约 0.1），所以键名应当稳定、中性。

**原文怎么保留**：调用前在本地（**不放进 `state`**）建一张 `ASCII 键 → 原始标签` 的映射表，例如：

| ASCII 键 | 原始标签 |
|---|---|
| `UP` | `上` |
| `SKILL2` | `技能2` |

选项原文照旧**逐字**翻译进 `state`（第 2 步），映射关系不会丢失；拿到 `choice` 后按这张表换回原始标签再执行。

**动作型的 criteria 必须与动作集一一对应**：有几个可执行动作就写几条，不多不少。不要因为你觉得某个动作"明显不该选"就把它删掉——模型无法选择被省略的选项，删掉它就等于替你做了决定。

### 第 5 步：调用 Jev

```powershell
$key = $env:TYPESAFE_API_KEY
if (-not $key) { $key = [Environment]::GetEnvironmentVariable('TYPESAFE_API_KEY','User') }
if (-not $key) { throw "TYPESAFE_API_KEY is not set" }

# 上面第 2 步写好的英语翻译（只翻译，无任何添加）
$state = @'
<在此粘贴纯英语翻译／客观状态描述>
'@

$payload = @{
  state = $state
  model = "jev-latest"
  questions = @{
    closed_choice = @{
      type         = "choice"
      instructions = "Which option should be selected to answer this question?"
      criteria     = @{
        "A" = "What choosing this option means."
        "B" = "What choosing this option means."
      }
    }
  }
} | ConvertTo-Json -Depth 8

$bytes = [System.Text.Encoding]::UTF8.GetBytes($payload)   # 含非 ASCII 字符时必需

# 用 Invoke-WebRequest + 显式 UTF-8 解码，不要用 Invoke-RestMethod：
# Windows PowerShell 5.1 的 Invoke-RestMethod 在响应未声明 charset 时按 ISO-8859-1 解码，
# 会把返回体里的非 ASCII 字符读成乱码（实测 "置顶" -> "ç½®é¡¶"）。
$r = Invoke-WebRequest -Method Post -Uri "https://api.typesafe.ai/v1/systemone" `
  -Headers @{ Authorization = "Bearer $key" } `
  -ContentType "application/json; charset=utf-8" -Body $bytes -UseBasicParsing

if ($r.RawContentStream) {
  # Windows PowerShell 5.1：取原始字节，自行按 UTF-8 解码
  $raw = [System.Text.Encoding]::UTF8.GetString($r.RawContentStream.ToArray())
} else {
  # PowerShell 7+：.Content 已是按 UTF-8 解码的字符串
  $raw = $r.Content
}
$resp = $raw | ConvertFrom-Json

$resp.answers.closed_choice | ConvertTo-Json -Depth 6
```

`instructions` 随情形调整：

| 情形 | `instructions` |
|---|---|
| 阻塞型 | `"Which option should be selected to resolve this blocking dialog and let the task continue?"` |
| 题目型 | `"Which single option states the correct answer to this question?"` |
| 动作型 | `"Which single action should be taken next to make progress toward the objective?"` |

要点：

- **一次请求只问这一个判断**，不必拆多次；
- 请求 ID（`closed_choice`）只是给你自己用的键，**不会发给模型**，所以问题必须完整写在 `state` 与 `instructions` 里；
- Jev **只接受纯文本**（不支持图片），所以截图必须先转成文字——这也正是"只翻译不添加"必须严格执行的原因：**模型判定的是你的翻译，不是原始画面**；
- 别把 `TYPESAFE_API_KEY` 写进任何文件、日志或回复里；
- **键名只用 ASCII**（见第 4 步）：`choice` 与 `probabilities` 的键都会原样回传，含中文就会撞上 5.1 的解码问题；
- `-UseBasicParsing` 在 PowerShell 7+ 上是空操作，保留无害；上面的写法在 5.1 与 7+ 上都能取到正确文本。

### 第 6 步：直接采用结果，不做验证

Jev 返回的 `choice` **就是目标选项**——概率最高的那个。**主模型不再进行任何验证、复核或自行推算**，直接把它作为目标。

```json
{
  "type": "choice",
  "choice": "C",
  "probabilities": { "A": 0.05, "B": 0.12, "C": 0.78, "D": 0.05 },
  "confidence": 0.79
}
```

| 字段 | 含义 | 用途 |
|---|---|---|
| `choice` | **目标选项**——概率最高的选项／动作 | **直接采用** |
| `probabilities` | 全部选项的分布，总和为 1 | 记录、排查、询问用户时附带 |
| `confidence` | 分布有多集中 | 记录用；**不设阈值、不作为闸门** |

**唯一的例外是高风险操作**：

| 情形 | 动作 |
|---|---|
| 选中项**不是**高风险操作 | **直接执行**——不询问、不验证、不复核 |
| 选中项**是**高风险操作 | **询问用户**，并给出**高风险原因** |

也就是说：**`confidence` 高低不影响是否执行。** 哪怕 `confidence` 只有 0.3、第二名紧咬，只要选中项不是高风险操作，也直接采用。

#### 什么算高风险操作

按**执行后果**判断，不是按文字像不像危险。命中以下任一条即为高风险：

| 类别 | 例子 |
|---|---|
| 不可逆／破坏性 | 删除、覆盖已存在文件、格式化、卸载、丢弃未保存内容 |
| 对外产生后果 | 发布、发送、提交、支付、购买、转账 |
| 消耗不可再生资源 | 消耗稀有道具、一次性额度、不可回退的配额 |
| 权限／授权／安全边界 | 系统权限申请、证书信任、提权、UAC、杀软放行、防火墙放行 |
| 影响他人或生产环境 | 改动线上系统、修改共享资源、通知他人 |
| 无法撤销的对外承诺 | 确认订单、接受协议、签署 |

`Don't Save` 看着中性，但它丢弃未保存内容，属于不可逆 → 高风险。

**题目型通常不涉及高风险操作**——回答一道选择题不产生后果，直接采用即可。高风险判断主要落在**阻塞型**（按钮会执行真实操作）与**动作型**（操作会改变真实状态）。

#### 高风险时怎么问

给出三样东西：

1. **Jev 选中的选项**——用**原始标签**，不是英语翻译；
2. **高风险原因**——具体说明执行它会造成什么不可逆后果，不要只说"有风险"；
3. **完整概率分布**（有助于用户判断，可选）。

例：`Jev 判定为「Replace」（概率 0.86）。但执行它会覆盖已存在的 report.csv，属不可逆操作，所以先请你确认：是否覆盖？`

#### 动作型的循环

选中的动作交回主模型执行时，用**原始标签**（不是英语翻译），由主模型去按键／点击，然后**重新观察状态、再次判定**，形成 观察 → 判定 → 执行 → 再观察 的循环。每一步都是一次独立判定，不要用上一步的结论去推断下一步。

### 动作型的适用边界

- **延迟**：单次判定约 0.2–0.3 秒，加上截图与转录，一轮实际开销通常在秒级。适合回合制、菜单式、节奏较慢的交互；**不适合需要毫秒级反应的实时操作**。
- **每一步都要重新观察**：不要缓存上一帧的状态去做下一步判断。状态变了，判断依据就变了。
- **动作集必须完整且稳定**：如果"能做什么"本身在变化（新技能解锁、菜单项动态增减），每轮都要重新枚举。
- **优先用状态接口而非截图**：如果目标程序有可读的状态（内存值、API、日志），直接取文本比截图＋转录更准、更快，也少一次转录出错的机会。

## 安全红线

1. **权限／授权／安全类弹窗一律不得由模型代决**。系统权限申请、证书信任、提权、UAC、杀软警告、防火墙放行——这些是安全边界的授予行为，必须由用户本人决定。模型没有资格代替用户授权。遇到这类弹窗直接停下问用户。
2. **高风险操作必须交回用户**，无论 `confidence` 多高、模型多自信。这是本 skill 唯一的强制闸门。
3. **绝不编造 `criteria` 里没有的选项／动作**，也不要因为模型倾向某个值就去点一个不存在的按钮、按一个不存在的键。
4. **绝不用"添加提示"来提高置信度**。第 2 步的禁令没有例外；如果你觉得不提示就答不对，正确的做法是如实翻译并采用结果（或在高风险时询问用户），而不是污染 state。
5. **动作型不得自行执行不可逆操作**（购买、支付、删除、发布、消耗稀有资源）——这类动作一律先交回用户确认。
6. **用完删除临时截图**：`Remove-Item $path -Force`。截图可能含敏感信息。
7. 画面／题目／状态只发给 `api.typesafe.ai`，不要转发到其他任何地方。

## 失败处理

| 现象 | 处理 |
|---|---|
| `TYPESAFE_API_KEY` 两处都为空 | 停下，让用户设置；**不要猜测选项** |
| `401` | 密钥无效或已撤销——让用户核对 |
| `403` | **实测不带 `Authorization` 头时会返回 403 而不是 401**，别误判成网络问题；先确认密钥确实读到了 |
| `422` | 请求体不合法，多半是 `criteria` 为空、`type` 拼错或 `score` 等级不足 2 个 |
| 返回的 `choice` 或键名是乱码（如 `ç½®é¡¶`） | 你用了 `Invoke-RestMethod` 的默认解码。改用第 5 步的 `Invoke-WebRequest` + 显式 UTF-8 写法，并按第 4 步把键名改成纯 ASCII |
| `429` / `529` | 指数退避重试，最多 3 次；仍失败则交回用户 |
| 网络不可达 | 交回用户，不要凭画面／题目自己拍板 |
| 截图失败或窗口矩形非法 | 改截全屏；仍失败就请用户口述弹窗内容 |
| 动作集无法完整枚举 | **不要调用**——这不是封闭集合，交回用户 |

**注意区分两件事**：上面的兜底是"**调用失败**时问用户"（拿不到 Jev 的判断，只能问人）；而"**调用成功**后不再验证"是本 skill 的既定规则。两者不冲突。

**任何失败路径的兜底都是"问用户"，不是"猜一个"。**

## 完整示例

### 例 1：阻塞型——覆盖确认弹窗（高风险，询问用户）

1. 截屏 → `read_image` → 看到标题 `Confirm Overwrite`，正文 `The file report.csv already exists. Do you want to replace it?`，三个按钮 `Replace` / `Keep Both` / `Cancel`。
2. 写英语 `state`——**只有弹窗本身，不含"文件是上次运行的产物、不需要保留"这类上下文**：

```text
A modal dialog titled "Confirm Overwrite".
Dialog body: "The file report.csv already exists. Do you want to replace it?"
Buttons: "Replace", "Keep Both", "Cancel".
```

3. type 选 `choice`（三个互斥按钮，无顺序）；`criteria` 的键用 **ASCII 标识**——本例按钮原文本身就是 ASCII，可直接沿用 `Replace`/`Keep Both`/`Cancel`。
4. 返回：

```json
{
  "type": "choice",
  "choice": "Replace",
  "probabilities": { "Replace": 0.86, "Keep Both": 0.11, "Cancel": 0.03 },
  "confidence": 0.74
}
```

5. **直接采用 `Replace` 作为目标**（不验证、不复核）。然后判断风险：`Replace` 会**覆盖一个已存在的文件**，属不可逆 → **高风险操作** → 询问用户并给出原因：

> Jev 判定为「Replace」（概率 0.86，其余 Keep Both 0.11 / Cancel 0.03）。但执行它会覆盖已存在的 `report.csv`，属不可逆操作，所以先请你确认：是否覆盖？

这个例子说明：`confidence` 0.74 不是拦住它的原因——**是"不可逆"拦住的**。若同一弹窗的按钮换成三个只读选项，就会直接采用，不再询问。

### 例 2：题目型——选择题（直接采用）

题目：`下列运算正确的是（ ）` / A. `x²·x³=x⁶` / B. `x²+x²=2x⁴` / C. `(-2x)²=4x²` / D. `(-2x)²·(-3x)³=6x⁵`

1. 写英语 `state`——**只翻译题干与选项，绝不加入"同底数幂相乘指数相加""注意符号"之类的法则或提示**：

```text
A multiple-choice algebra question.
Question: "Which of the following operations is correct?"
Option A: x^2 * x^3 = x^6
Option B: x^2 + x^2 = 2x^4
Option C: (-2x)^2 = 4x^2
Option D: (-2x)^2 * (-3x)^3 = 6x^5
```

2. type 选 `choice`；`criteria` 键用 `A`/`B`/`C`/`D`。
3. 返回：

```json
{
  "type": "choice",
  "choice": "C",
  "probabilities": { "A": 0.0, "B": 0.0, "C": 1.0, "D": 0.0 },
  "confidence": 1.0
}
```

4. **直接采用 `C` 作为答案。** 不做验算、不做复核、不比对其他解法。
5. 风险判断：回答一道选择题不产生任何后果，**非高风险操作** → 直接给出答案 `C`。

### 例 3：动作型——游戏中的可穷举操作（直接执行）

1. 截屏 → `read_image` → 看到：角色位于画面左侧，血条约 30%；右侧两名敌人正在靠近，其中一名已进入近战距离；技能1（远程攻击）就绪，技能2（治疗）冷却中、还剩 3 秒。
2. 枚举动作集：`上`、`下`、`左`、`右`、`技能1`、`技能2`。
3. 写英语 `state`——**客观描述状态 + 任务目标；不含"应该先自保"这类策略或结论**：

```text
A 2D action game.
Objective: survive the current encounter and clear the room.
Current state: the player character is at the left edge of the screen with the
health bar at about 30%. Two enemies are approaching from the right; one is
already within melee range. Skill 1 (ranged attack) is ready. Skill 2 (heal) is
on cooldown with about 3 seconds remaining.
Available actions: 上, 下, 左, 右, 技能1, 技能2.
```

4. type 选 `choice`；`criteria` 的**键用 ASCII 短名**，值写英语释义；同时在本地记下 `ASCII 键 → 原始标签` 的映射（`UP`→`上`、`DOWN`→`下`、`LEFT`→`左`、`RIGHT`→`右`、`SKILL1`→`技能1`、`SKILL2`→`技能2`）：

```json
{
  "UP": "Move up.",
  "DOWN": "Move down.",
  "LEFT": "Move left.",
  "RIGHT": "Move right.",
  "SKILL1": "Use skill 1, the ranged attack (ready).",
  "SKILL2": "Use skill 2, the heal (on cooldown, about 3 seconds left)."
}
```

5. `instructions` 用：`"Which single action should be taken next to make progress toward the objective?"`
6. 返回：

```json
{
  "type": "choice",
  "choice": "SKILL2",
  "probabilities": { "UP": 0.04, "DOWN": 0.05, "LEFT": 0.09, "RIGHT": 0.03, "SKILL1": 0.22, "SKILL2": 0.57 },
  "confidence": 0.42
}
```

7. **直接采用 `SKILL2` 作为目标动作**（不验证、不复核）。虽然 `confidence` 只有 0.42、第二名 `SKILL1` 有 0.22，但 `confidence` **不设阈值**，不影响执行。
8. 风险判断：在游戏内使用治疗技能是**可逆操作**，不产生不可逆后果 → **非高风险** → 按本地映射表把 `SKILL2` 换回原始标签 `技能2`，交回主模型执行，然后重新观察状态进入下一轮。

   注意与例 1 的对比：例 1 因为**不可逆**而询问用户，本例因为**可逆**而直接执行——**决定是否询问的是风险，不是 `confidence`**。
