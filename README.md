# dbskill 审稿台

双击 `index.html` 即可使用。不需要 Python、不需要服务器、不需要联网部署。

- 模型：`deepseek-v4-pro`
- 端点：`https://api.deepseek.com/anthropic`（Anthropic 兼容格式）
- API Key 存在浏览器 localStorage，不在本仓库任何文件里

## 流程

1. 粘稿子 → 勾维度 → **开始审稿**（六个维度各自出诊断报告）
2. **生成综合结论**（把各维度报告合成一份决断：能不能发 / 必须改哪几条 / 维度之间打架时站哪边）
3. **标记式改稿**（在原文上叠加 `~~删除线~~` 和 `🆕` 标记，末尾附改动清单表格）
4. **复制 TXT** / **下载 .txt**（改稿卡片上直接点，一键去标记拿纯文本定稿）
   想先看一眼再复制，就点 **生成干净版**，它会单独出一张卡，上面同样有这两个按钮

第 4 步是纯本地字符串处理，不再调模型、不花 token。

### TXT 输出的细节

- 换行统一成 CRLF，粘进记事本 / Word / 剪映不会挤成一行
- 下载的 .txt 带 UTF-8 BOM，老一点的 Windows 工具才认得出中文
- 剪贴板走 `navigator.clipboard`，失败自动退回 `execCommand` 兜底（`file://` 在 Chrome 里算安全上下文，两条路都可用）

改稿范围可选「只改必须改的」（只处理 🔴 高风险）或「所有建议都改」。

### 改稿的边界

改稿只管「怎么说」，不改「说什么」——不动论点、案例、数据，也不会替你编你没说过的事实。
如果审稿报告指出的是「选题有问题」「核心不明确」这类内容层面的问题，
它会在改动清单里单独列一条写明「需要作者自己决定，我没动」，而不是自己编内容去补。

### 干净版的提取规则

`stripMarks()` 在本地做这几件事：砍掉「改动清单」和「决断」两节；丢弃整行删除线的段落；
去掉行内删除线的内容；剥掉 `🆕` 前缀；丢弃 `⚠️` 元信息行；
丢弃形如 `🆕（此处已删）` 的纯说明行。已对 12 种边界情况做过断言测试。

## ⚠️ prompt 是冻结快照，不会自动跟随 dbskill 更新

`index.html` 里的六份 system prompt 是从本机 `~/.claude/skills/` 手工浓缩的，**运行时不读 SKILL.md**。
跑过 `/dbs-update` 之后，这里的判定标准不会跟着变，也不会报错——它会静默沿用下表这个版本。

冻结于 **2026-07-27**，源文件状态：

| 维度 | skill | SKILL.md 改动日期 | md5 前 12 位 |
|---|---|---|---|
| AI 味检测 | `dbs-ai-check` | 2026-07-17 | `6da6f047b783` |
| 共鸣诊断 | `dbs-resonate` | 2026-07-17 | `805b67922727` |
| 逻辑延续 | `dbs-script-flow` | 2026-07-17 | `ea402c4e1741` |
| 内容五维 | `dbs-content` | 2026-07-17 | `ed92ef36d272` |
| 开头诊断 | `dbs-hook` | 2026-07-17 | `44119d40f029` |
| 传播解码 | `dbs-spread` | 2026-07-17 | `d7a2f43b6395` |

### 怎么检查有没有漂移

```bash
cd ~/.claude/skills
for s in dbs-ai-check dbs-resonate dbs-script-flow dbs-content dbs-hook dbs-spread; do
  printf "%-18s %s\n" "$s" "$(md5sum $s/SKILL.md | cut -c1-12)"
done
```

对不上就说明该重新生成 prompt 了。

## 已知坑

**传了 DeepSeek 不认识的模型名，后端会静默降级到 `deepseek-v4-flash`，不报错。**
所以「模型」那栏改错了不会有任何提示，只会觉得审得变糙了。

## 相关实现

`~/projects/book-workflow/modules/content/dbskill_review.py` 是另一份 dbskill 审稿实现，
架构不同：**运行时读取 SKILL.md**，所以不存在漂移问题；但走的是 `deepseek-chat` + OpenAI 格式端点，
流程是「`dbs-content` 入口 → 路由到专项 skill → 落盘 → 停下等人决策」，不是六维并列。
注意 `book-workflow-optimized` 里没有这个模块。
