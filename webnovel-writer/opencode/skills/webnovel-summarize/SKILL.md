---
name: webnovel-summarize
description: 生成章节结构化摘要。写完章节后调用，为后续章节提供上下文。支持单章或批量生成。
tools: Read Write Edit Grep Bash
---

# 章节摘要生成 (Summary Generator v1.0)

## 目标

- 为每个已完成章节生成结构化摘要
- 摘要用于替代 RAG，为新章节写作提供上下文
- 输出格式标准化，便于 context-agent 读取

## 使用方式

```bash
/webnovel-summarize 42           # 生成第42章摘要
/webnovel-summarize 1-50         # 批量生成第1-50章摘要
/webnovel-summarize --latest     # 生成最新章节摘要
```

## 执行原则

1. 必须读取完整章节正文才能生成摘要
2. 摘要字数控制在 300-500 字
3. 结构化字段必须完整填写
4. 伏笔动态必须与 state.json 中的 plot_threads 对应
5. 生成后立即写入 `.webnovel/summaries/ch{NNNN}.md`

## 引用加载

- 必读：`references/summary-schema.md`（摘要格式定义）
- 必读：`${OPENCODE_PLUGIN_ROOT}/references/summary-schema.md`（完整 schema）

## 输出位置

```
{project_root}/.webnovel/summaries/ch{NNNN}.md
```

## 执行流程

### Step 1: 确定目标章节

```bash
# 解析用户输入
# 单章: 42 → chapter_list = [42]
# 范围: 1-50 → chapter_list = [1, 2, ..., 50]
# 最新: --latest → 从 state.json 读取 current_chapter
```

### Step 2: 读取章节正文

```bash
# 定位章节文件
cat "{project_root}/正文/第{NNNN}章.md"

# 或按卷组织
cat "{project_root}/正文/第{volume}卷/第{NNNN}章.md"
```

### Step 3: 读取相关状态

```bash
# 读取 state.json 获取：
# - plot_threads.foreshadowing（伏笔列表）
# - chapter_meta（如有）
# - protagonist_state

python "${OPENCODE_PLUGIN_ROOT}/../scripts/webnovel.py" \
  --project-root "{project_root}" \
  state show
```

### Step 4: 读取章节大纲（可选）

```bash
# 获取本章大纲中的预设信息
cat "{project_root}/大纲/第{volume}卷/章纲.md" | grep -A 20 "第{chapter}章"
```

### Step 5: 生成结构化摘要

基于正文内容，提取以下信息：

#### 5.1 YAML Frontmatter

```yaml
---
chapter: {NNNN}                    # 章节编号，4位
title: "{章节标题}"                 # 从正文提取
time_anchor: "{时间锚点}"           # 如"末世第7天 黄昏"
location: "{主要地点}"              # 本章主场景
characters: ["{角色1}", "{角色2}"]  # 出场角色列表
pov: "{视角角色}"                   # 主视角
word_count: {字数}                  # 正文字数
emotion_arc: "{情绪弧线}"           # 如"紧张 → 震惊 → 悲壮"
hook:
  type: "{钩子类型}"                # 悬念钩/危机钩/情感钩/反转钩
  content: "{钩子内容}"             # 章末钩子描述
  strength: "{强度}"                # strong/medium/weak
ending_state:
  emotion: "{结束情绪}"             # 本章结束时的情绪基调
  tension: {紧张度}                 # 1-10
---
```

#### 5.2 正文部分

```markdown
## 剧情要点

- {事件1：简洁描述，一句话}
- {事件2}
- {事件3}
- {事件4}（最多5条）

## 角色变化

| 角色 | 变化 | 原因 |
|------|------|------|
| {角色} | {状态A} → {状态B} | {原因} |

## 伏笔动态

- [新增] {伏笔内容} → 待回收（紧急度:{N}）
- [推进] {伏笔内容}（第{X}章埋下）→ 进度 {N}%
- [回收] {伏笔内容}（第{X}章埋下）→ 已闭合

## 关系变化

- {角色A} → {角色B}: {旧关系} → {新关系}

## 承接要点

下章需要：
- {需处理事项1}
- {需处理事项2}
```

### Step 6: 写入摘要文件

```bash
# 确保目录存在
mkdir -p "{project_root}/.webnovel/summaries"

# 写入文件
# 文件名格式：ch{NNNN}.md，如 ch0042.md
```

### Step 7: 更新 state.json（可选）

如果生成过程中发现 state.json 中缺少某些信息（如 chapter_meta），可以补充：

```bash
python "${OPENCODE_PLUGIN_ROOT}/../scripts/webnovel.py" \
  --project-root "{project_root}" \
  state update-meta --chapter {NNNN} --hook-type "{type}" --hook-content "{content}"
```

## 质量检查清单

生成摘要后，检查：

- [ ] YAML frontmatter 格式正确
- [ ] 所有必填字段已填写
- [ ] 剧情要点不超过 5 条
- [ ] 伏笔动态与 plot_threads 对应
- [ ] 字数在 300-500 范围内
- [ ] 钩子类型符合规范（悬念钩/危机钩/情感钩/反转钩）

## 批量生成注意事项

批量生成时（如 1-50）：

1. 按顺序处理，确保伏笔状态连贯
2. 每 10 章输出一次进度
3. 遇到章节文件缺失时跳过并报告
4. 完成后输出统计：成功数、跳过数、失败数

## 示例输出

```markdown
---
chapter: 0042
title: "悬崖对峙"
time_anchor: "末世第7天 黄昏"
location: "断魂崖"
characters: ["林凡", "张三"]
pov: "林凡"
word_count: 2300
emotion_arc: "紧张 → 震惊 → 悲壮"
hook:
  type: "悬念钩"
  content: "张三跌落悬崖，生死不明"
  strength: "strong"
ending_state:
  emotion: "悲壮"
  tension: 8
---

## 剧情要点

- 林凡追踪张三至断魂崖
- 张三揭露真实身份：前朝太子
- 张三使用禁术「血焚」，代价是三十年寿命
- 激战中张三跌落悬崖

## 角色变化

| 角色 | 变化 | 原因 |
|------|------|------|
| 林凡 | 愤怒 → 理解 | 得知张三身世 |
| 张三 | 隐藏 → 暴露身份 | 被逼至绝境 |

## 伏笔动态

- [新增] 禁术「血焚」来源 → 待回收（紧急度:60）
- [推进] 神秘玉佩发光（第30章埋下）→ 进度 50%
- [回收] 张三失踪之谜（第15章埋下）→ 已闭合

## 关系变化

- 林凡 → 张三: 敌对 → 复杂（亦敌亦友）

## 承接要点

下章需要：
- 交代张三生死
- 处理林凡的情绪转变
- 玉佩发光后续
```
