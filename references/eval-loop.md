# 评测闭环（Eval Loop · 本地适配版）

> 来源：官方 skill-creator 2026-08 "Test, measure, refine" 大更新（第 6 轮吸收）。原版脚本绑死 `claude -p` CLI，本版方法不变、载体换成本地 Harness 的 subagent 与聊天内评审。适用：A 模式造完新 Skill、C 模式大改之后。简单/主观型技能可跳过（与官方口径一致）。
> 前置：§3 若借用官方 skill-creator 的 `agents/` 提示词，需本地装有该技能；未安装时以聊天内评审替代，方法本身不受影响。

## 一、效果评测（with-skill vs 裸跑对照）

1. **出 2-3 条真实测试题**：像真用户会说的话（含错别字/口语/具体文件名），先给用户过目。
2. **同轮并发两组 subagent**：A 组给技能路径并要求按技能执行；B 组裸跑同题。改版场景 B 组跑旧版快照。
3. **产出落工作区**：`<skill名>-workspace/iteration-N/<测试名>/{with_skill,without_skill}/`，每组记 token 与耗时。
4. **断言（assertions）**：趁 subagent 跑着时起草——必须客观可判（文件存在/字段正确/格式合规），主观质量交给用户；每条一句人话名。
5. **评分**：逐断言判 pass/fail 并留证据（能跑脚本验证的写脚本，不靠眼看）。
6. **汇总基准表**：

| 配置 | 通过率 | token（均值±波动） | 耗时 |
|---|---|---|---|
| with_skill | | | |
| without_skill / old | | | |
| Δ 提升 | | | |

7. **人工评审**：本地 Harness 无浏览器评审台时用聊天内评审（逐例展示 prompt+产出+评分，用户逐条反馈）；可选用官方 skill-creator 的 `eval-viewer/generate_review.py --static` 生成独立 HTML 交用户。
8. **迭代**：按反馈改技能 → 下一轮 iteration-N+1（含基线重跑）→ 直到用户满意或反馈全空。

## 二、触发评测与描述优化（description optimizer 手动版）

> 原理（官方 2026-08 实测口径）：模型有"欠触发"倾向，description 要写得"pushy"一点；且模型只对"自己搞不定的复杂任务"查技能，太简单的查询测不出触发。

1. **造 20 条查询**：8-10 条应触发（同意图多种说法：正式/口语/不点名技能的）+ 8-10 条不应触发的**近似误伤题**（同关键词但真意是别的技能/别的工具）。负例必须刁钻，"明显无关题"测不出东西。
2. **用户过目**后存 `evals/trigger-eval.json`。
3. **跑 3 遍取稳定值**（每条查询 3 个新会话/subagent），按 60/40 分训练/保留测试集。
4. **改 description → 只看保留集分数选优**，防过拟合；最多 5 轮。
5. **采纳**：把最优 description 写回 frontmatter，报告前后分数。官方报告此法触发优化幅度可达 83.3%（[官方博客](https://claude.com/blog/improving-skill-creator-test-measure-and-refine-agent-skills)，官方库语境，本地待实测）。

## 三、盲测对比（可选，重要技能用）

两个版本的产出抹去署名交给独立 subagent 评高下（官方 comparator/analyzer 模式）；再分析赢的原因。官方 skill-creator 的 `agents/{grader,comparator,analyzer}.md` 可直接借为 subagent 提示词。

## 四、升级铁律（官方经验的辩证吸收）

- **从反馈泛化，不过拟合**：技能要服务千万次调用，不为这几道测试题打补丁；顽固问题换比喻/换模式，别堆 MUST。
- **保持精瘦**：删掉不出力的指令；读 transcript 不只读产出——发现技能让模型空转就砍那一段。
- **解释 why 优于堆 MUST**（新官方哲学，与老宪法"纯祈使"的裁量）：**低自由度机械段**（脚本调用、格式、安全红线）保持硬祈使零解释；**高自由度判断段**（何时收敛、为何如此设计）写清 why，让模型理解后自行泛化。满篇 ALWAYS/NEVER 大写是黄旗。
- **重复劳动→收编脚本**：3 个测试里 subagent 各自写了同款脚本 = 强信号，固化进 `scripts/`。
