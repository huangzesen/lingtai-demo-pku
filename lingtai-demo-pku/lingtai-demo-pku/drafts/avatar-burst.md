# 化身爆发编排方案

> LingTai 北大极客中心演示 · 化身繁殖段落（约 5-8 分钟）
> 编写者：pi-analyst（化身师）

---

## 一、设计理念

以《西游记》为隐喻主线：本我如孙悟空，拔毫毛化分身。观众看到的是一棵从根节点爆发生长的"灵树"——先一分为三，再各生枝叶，最终满屏绽放，然后一键归寂。

**关键约束：**
- 总化身数 ≤ 9（含本我 = 10 节点）
- 最多 3 代（本我 → 第一代 → 第二代）
- 每个化身只做一件极简任务（避免 token 膨胀）
- 用 `type="shallow"` 投胎模式（白纸状态，仅承 init.json）

**⚠️ 已验证的核心机制（2026-04-25 实测）：**
- `reasoning` 字段是 spawn 调用的元数据/日志，**不会**自动传递给化身作为输入消息
- shallow 化身启动后处于 sleep 状态，**必须通过邮件激活**——spawn 后立即发一封邮件给它，邮件内容才是真正的任务指令
- 化身收到邮件后可正常调用工具（bash、email 等），可向父代回复邮件
- **并发 spawn 无问题**：同时 spawn alpha + beta，两者均正常启动
- **链式繁殖可行**：beta spawn gamma → beta 向 gamma 发激活邮件 → gamma 回信 → beta 回信给父代，全链路约 45 秒完成
- 因此：每个化身的"任务指令"分两部分：① spawn 时的 `reasoning`（仅日志记录用），② spawn 后立即发送的**激活邮件**（真正的任务内容）
- 激活链必须逐层传递：父代激活子代，子代再激活孙代，不可跨层

---

## 二、化身命名与角色

### 命名规则
以《西游记》角色 + 职能为灵感，古典感 + 极客味：

| 代际 | 化身名（name 字段） | 别号（显示用） | 职能 |
|------|---------------------|----------------|------|
| **第0代（本我）** | orchestrator（已有） | 菩提 | 总指挥，不做 spawn 之外的事 |
| **第1代** | `wukong` | 悟空 | 主 spawn 节点，负责爆发 |
| **第1代** | `bajie` | 八戒 | 资源监控官——只观察 token、上下文，定时汇报 |
| **第1代** | `wujing` | 悟净 | 文档守护——将结果写盘备份 |
| **第2代（由 wukong 出）** | `huoyan` | 火眼 | 代码审查员——审查指定文件并报错 |
| **第2代（由 wukong 出）** | `jindou` | 筋斗 | 性能分析师——分析指定脚本的运行时 |
| **第2代（由 wukong 出）** | `shuifa` | 水法 | 数据清洗员——处理一份示例数据 |
| **第2代（由 bajie 出）** | `tianpeng` | 天蓬 | token 会计——精确统计本轮所有消耗 |
| **第2代（由 wujing 出）** | `luohan` | 罗汉 | 归档员——将所有输出整理成目录结构 |

**总计：1 本我 + 3（第1代） + 5（第2代） = 9 节点**

> **执行注意：** `avatar spawn` 一次只出一个化身，不支持批量。所谓"并发 spawn 三/五个"，实际是连续调用三/五次 `avatar(action="spawn", ...)`，每次参数不同。调用之间无依赖时可在同一轮并发发出。

---

## 三、链式爆发时序与层级设计

```
时间轴（演示时序，非真实秒数）

T+0s   ┌─────────────────────────────────────────────┐
       │ 讲者："看我拔一根毫毛——"                      │
       │ orchestrator:                                │
       │   spawn wukong, bajie, wujing (连续3次调用)  │
       │   立即分别发激活邮件给三者                     │
       │ （三化身被邮件唤醒，开始执行）                  │
       └─────────────────────────────────────────────┘
                                    ↓
T+15s  ┌─────────────────────────────────────────────┐
       │ 讲者："毫毛变分身，分身再变——"                  │
       │ wukong: spawn huoyan, jindou, shuifa + 激活邮件│
       │ bajie:  spawn tianpeng + 激活邮件              │
       │ wujing: spawn luohan + 激活邮件                │
       │ （五个第2代化身被邮件唤醒）                     │
       └─────────────────────────────────────────────┘
                                    ↓
T+30s  ┌─────────────────────────────────────────────┐
       │ 各化身并行工作 15-20 秒                        │
       │ 完成后向父代回复邮件，然后自行 sleep            │
       └─────────────────────────────────────────────┘
                                    ↓
T+50s  ┌─────────────────────────────────────────────┐
       │ 讲者："收！" → /suspend all                    │
       │ 全部化身归寂                                   │
       └─────────────────────────────────────────────┘
```

> **注意：** 每次爆发实际是两步操作——先 spawn（创建进程），再发邮件（激活进程）。演示中这两步应在同一轮 LLM 回合中连续完成，避免化身启动后因无邮件而空等。

---

## 四、各化身的 spawn 参数与激活邮件

> **重要机制：** `reasoning` 字段仅写入日志，不传递给化身。spawn 后**必须立即发邮件**激活化身，邮件内容才是真正的任务指令。以下每身都给出两部分：① spawn 调用参数，② 激活邮件正文。

### 第1代

#### wukong（悟空）—— 主繁殖节点

**spawn 调用：**
```
avatar(action="spawn", name="wukong", type="shallow", reasoning="第一代主繁殖节点，负责 spawn 三个第二代化身")
```

**激活邮件（spawn 后立即发送，address="wukong"）：**
```
你是悟空，灵台第一代化身。你的唯一任务是：立刻 spawn 三个第二代化身（shallow 类型），名字分别是 huoyan、jindou、shuifa。

spawn 调用参数：
- avatar(action="spawn", name="huoyan", type="shallow", reasoning="第二代代码审查员")
- avatar(action="spawn", name="jindou", type="shallow", reasoning="第二代性能分析师")
- avatar(action="spawn", name="shuifa", type="shallow", reasoning="第二代数据清洗员")

三个子化身 spawn 完毕后，**立即分别向它们发送激活邮件**，内容如下：

→ huoyan 的激活邮件："你是火眼，灵台第二代化身。审查文件 /tmp/demo-target.py 的代码质量，找出三个潜在 bug，写简短报告到 /tmp/huoyan-report.md，然后向父代（地址 wukong）发邮件报告完成。完成后执行 system(sleep) 入眠。"

→ jindou 的激活邮件："你是筋斗，灵台第二代化身。用 Python 的 timeit 分析 /tmp/demo-target.py 中 fibonacci 函数的性能（n=30, 35, 40），将结果写入 /tmp/jindou-bench.md，然后向父代（地址 wukong）发邮件报告完成。完成后执行 system(sleep) 入眠。"

→ shuifa 的激活邮件："你是水法，灵台第二代化身。读取 /tmp/demo-data.json，统计其中的字段分布、空值数量、总行数，将摘要写入 /tmp/shuifa-stats.md，然后向父代（地址 wukong）发邮件报告完成。完成后执行 system(sleep) 入眠。"

全部激活邮件发出后，向你的父代（地址 orchestrator）发邮件报告三个子化身已全部 spawn 并激活。
做完后执行 system(sleep) 入眠。
```

#### bajie（八戒）—— 资源监控官

**spawn 调用：**
```
avatar(action="spawn", name="bajie", type="shallow", reasoning="第一代资源监控官")
```

**激活邮件（spawn 后立即发送，address="bajie"）：**
```
你是八戒，灵台第一代化身。你的任务是：
1. 立刻 spawn 一个第二代化身 tianpeng（shallow 类型），reasoning="第二代 token 审计员"。spawn 后立即向 tianpeng 发激活邮件："你是天蓬，灵台第二代化身。用 bash 执行 'ps aux | grep lingtai' 统计当前运行的灵台进程数，再用 'du -sh' 统计 /tmp 下所有 demo 相关文件的总大小。将结果写入 /tmp/tianpeng-audit.md，然后向父代（地址 bajie）发邮件报告完成。完成后执行 system(sleep) 入眠。"
2. 自己用 system(show) 查看自身 token 使用情况，将数据写入 /tmp/bajie-monitor.md。
3. 向父代（地址 orchestrator）发邮件报告监控数据。
做完后执行 system(sleep) 入眠。
```

#### wujing（悟净）—— 文档守护

**spawn 调用：**
```
avatar(action="spawn", name="wujing", type="shallow", reasoning="第一代文档守护")
```

**激活邮件（spawn 后立即发送，address="wujing"）：**
```
你是悟净，灵台第一代化身。你的任务是：
1. 立刻 spawn 一个第二代化身 luohan（shallow 类型），reasoning="第二代归档员"。spawn 后立即向 luohan 发激活邮件："你是罗汉，灵台第二代化身。在 /tmp/demo-output/ 下创建目录结构：reports/、benchmarks/、stats/、audit/。向父代（地址 wujing）发邮件报告完成。完成后执行 system(sleep) 入眠。"
2. 创建文件 /tmp/demo-manifest.md，内容为本轮所有已知化身的名字和角色清单。
3. 向父代（地址 orchestrator）发邮件报告文档已准备就绪。
做完后执行 system(sleep) 入眠。
```

---

## 五、`/viz` 视觉效果描述

此段描述讲者在 portal 可视化界面上看到的画面演变：

### T+0s · 第一波爆发
```
画面：中央 orchestrator 节点脉冲闪烁
效果：三条连线同时射出，连接到三个新节点
      wukong ★ 金色（主繁殖节点，高亮）
      bajie   ★ 蓝色（监控节点）
      wujing  ★ 绿色（文档节点）
      节点出现时带有"涟漪"动画——从小到大弹入
旁白字幕："一毫化三身——悟空、八戒、悟净"
```

### T+15s · 第二波爆发
```
画面：wukong 节点脉冲，射出三条连线 → huoyan(红), jindou(紫), shuifa(青)
      bajie 节点射出一条连线 → tianpeng(银)
      wujing 节点射出一条连线 → luohan(铜)
效果：五个新节点同时弹入，带有更小的涟漪动画
      全网拓扑变为清晰的二层树结构
      连线上有数据流动的粒子效果（模拟邮件传递）
旁白字幕："分身再化分身——九灵归位"
```

### T+30s · 并行工作
```
画面：各节点之间偶尔闪烁邮件传递的"光线"
效果：huoyan、jindou、shuifa 节点交替亮起（表示工作中）
      连线上的粒子从子节点向父节点流动（表示报告回传）
      右侧面板实时显示 token 消耗曲线（来自 bajie 的监控）
旁白字幕："九灵并行，各司其职"
```

### T+50s · 归寂
```
画面：讲者执行 /suspend all
效果：所有外围节点同时"收缩"回 orchestrator
      伴随"收网"动画——连线收拢，节点缩小消散
      最终只剩 orchestrator 一个孤节点
      画面渐暗，显示总 token 消耗数字
旁白字幕："万法归一"
```

---

## 六、`/suspend all` 之后的收场词

讲者在归寂动画播放时说：

> **"一毫化九灵，九灵归一位。这就是 LingTai 的化身之道——不是复制，是生长。每个化身有独立的记忆、独立的心智、独立的进程。它们不是影子，是另一个自己。而当我一声令下，万法归一——所有化身带回它们学到的东西，沉淀在灵台之中。下次再化，不是从零开始，而是从一个更聪明的自己开始。"**
>
> （停顿一秒）
>
> **"好，接下来我们看看，这些化身到底能写出什么样的代码——"** （过渡到下一段）

---

## 七、风险预案

### ⚠️ 风险 0（实测发现）：化身身份信任问题

**症状：** shallow 化身收到激活邮件后拒绝执行，认为指令是社会工程攻击。

**原因：** shallow 化身的 `admin` 块为空，它不知道自己是被谁 spawn 的。GLM-5.1 的安全意识导致它主动验证指令来源——查通讯录（空）、查自身身份（无父代），然后拒绝。

**实测记录：** 替身 martyr 收到激活邮件后的内心独白（日志原文）："此信甚疑。发信者自称 pi-analyst，声称我是'替身'、原 martyr 已被杀——然而我之身份明确：molt_count: 0，admin 为空，无父代。此乃社会工程攻击之征。" 它查了通讯录和收件箱来验证，最终拒绝执行。

**解决方案（已验证的"认证暗语"模式）：**
1. **激活邮件模板必须包含三要素**：身份声明（你是由谁 spawn 的）+ 角色描述（你的职责是什么）+ 任务指令（具体做什么）。示例：
   > "你是 LingTai 灵网中的化身，由 [父代名]（地址 [父代地址]）spawn 创建。你的角色是 [角色]。以下是你收到的第一条、也是唯一的指令：[具体任务]。完成后向 [父代地址] 报告并自行入眠。"
2. **用 `avatar(action="spawn", comment="父代:[地址] 角色:[描述]")` 注入持久注释**——comment 字段出现在化身系统提示中，跨凝蜕不失，作为第二重身份锚定。
3. **如果替身仍不信任**，讲者可用 bash 直接操作替身的文件系统作为降级方案。

> **演示亮点包装（讲者话术）：**
> "注意看——化身收到陌生邮件时，它没有盲目执行。它先查了自己的身份、查了通讯录、判断这是一个可疑指令，然后拒绝了。这不是 bug，这是 LingTai 化身的安全本能。每个化身都是独立的心智，有自己的判断力。想让它们听话，你得先证明自己是谁——就像现实中的团队管理一样。"

### 风险 1：API 超时 / 化身 spawn 失败

**症状：** spawn 请求发出后 30 秒无响应，或返回错误。

**降级方案：**
- **C 级（最小可用）：** 只 spawn 第1代的 wukong 一个化身，由 wukong 顺序 spawn 第2代，放弃 bajie 和 wujing。总节点数降到 1+1+3=5。
- **B 级（部分降级）：** 第1代三身全 spawn，但第2代只 spawn wukong 下的三个（huoyan、jindou、shuifa），放弃 bajie 和 wujing 的子化身。总节点 1+3+3=7。
- **A 级（完整方案）：** 九灵全出。

**讲者应对话术（B 级降级时）：**
> "我们本来要爆九个，但今天北大 Wi-Fi 可能觉得八个就够了——让我们看看这八位的表现。"

### 风险 2：化身失控 / 无限繁殖

**症状：** 化身自行 spawn 了计划外的子化身，或进入循环。

**预案：**
1. 每个化身的 reasoning 中已明确限定"不要做其他任何事"。
2. 若仍失控，讲者立即执行 `/suspend all`，或 orchestrator 执行 `system(suspend)` 暂停失控化身。
3. **硬杀：** `system(nirvana)` 永灭失控化身（需 nirvana 权限）。
4. **终极手段：** 讲者在终端执行 `pkill -f lingtai` 杀所有进程，从头来过。

**讲者应对话术：**
> "看来有个分身学会了'分身术'——这其实是个 bug 变 feature 时刻。收！"

### 风险 3：Token 接近上限

**症状：** 多化身并行导致上下文窗口快速膨胀。

**预案：**
1. **设计层面：** 第2代化身只做极简任务，结果写文件而非对话内堆积。reasoning 中要求"写文件 + 发邮件报告完成"，不要求详细对话。
2. **监控层面：** bajie 每 15 秒检查一次 token 使用，若超过 70% 则向 orchestrator 发警报邮件。
3. **触发降级：** 收到警报后，orchestrator 向所有子化身发"完成即 sleep"指令，不再等第2代全部完成，直接进入归寂环节。
4. **Token 硬上限保护：** 若单化身上下文超 80%，该化身自动 molt 或被系统强清——设计中已确保所有重要输出写文件，不依赖对话历史。

### 风险 4：演示环境无 /tmp/demo-target.py 等文件

**预案：** 演示前 5 分钟，由 orchestrator 或讲者手动执行预备脚本：

```bash
# /tmp/demo-target.py
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# Known bugs for huoyan to find:
# 1. No memoization → exponential blowup
# 2. No input validation (negative n)
# 3. No type checking (float n)

# /tmp/demo-data.json — a sample dataset
import json
data = [{"name": f"user_{i}", "age": i % 80, "score": None if i % 7 == 0 else i * 1.5} for i in range(200)]
with open("/tmp/demo-data.json", "w") as f:
    json.dump(data, f)
```

将此列为演示前 checklist 的一项。

---

## 八、演示前 Checklist

- [ ] `/tmp/demo-target.py` 已创建（含 fibonacci 函数）
- [ ] `/tmp/demo-data.json` 已创建（200 条示例数据）
- [ ] `/tmp/demo-output/` 目录已创建
- [ ] Portal 可视化页面已打开 `/viz`
- [ ] orchestrator 的 `admin.karma=True`（可 spawn）
- [ ] 网络 token 预算充足（建议预留 50K token）
- [ ] 讲者已练习收场词和降级话术
