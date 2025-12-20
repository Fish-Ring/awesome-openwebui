# 异步上下文压缩过滤器 - 工作流程指南

## 📋 目录
1. [概述](#概述)
2. [系统架构](#系统架构)
3. [工作流程详解](#工作流程详解)
4. [Token 计数机制](#token-计数机制)
5. [递归摘要机制](#递归摘要机制)
6. [配置指南](#配置指南)
7. [最佳实践](#最佳实践)

---

## 概述

异步上下文压缩过滤器是一个高性能的消息压缩插件，通过以下方式降低长对话的 Token 消耗：

- **智能摘要**：将历史消息压缩成高保真摘要
- **递归更新**：新摘要合并旧摘要，保证历史连贯性
- **异步处理**：后台生成摘要，不阻塞用户响应
- **灵活配置**：支持全局和模型特定的阈值配置

### 核心指标
- **压缩率**：可达 65% 以上（取决于对话长度）
- **响应时间**：inlet 阶段 <10ms（无计算开销）
- **摘要质量**：高保真递归摘要，保留关键信息

---

## 系统架构

```
┌─────────────────────────────────────────────────────┐
│                  用户请求流程                        │
└────────────────┬────────────────────────────────────┘
                 │
    ┌────────────▼──────────────┐
    │   inlet（请求前处理）       │
    │  ├─ 加载摘要记录           │
    │  ├─ 注入摘要到首条消息     │
    │  └─ 返回压缩消息列表       │ ◄─ 快速返回 (<10ms)
    └────────────┬──────────────┘
                 │
    ┌────────────▼──────────────┐
    │     LLM 处理消息           │
    │  ├─ 调用语言模型           │
    │  └─ 生成回复               │
    └────────────┬──────────────┘
                 │
    ┌────────────▼──────────────┐
    │   outlet（响应后处理）      │
    │  ├─ 启动后台异步任务       │
    │  └─ 立即返回（不阻塞）     │ ◄─ 返回响应给用户
    └────────────┬──────────────┘
                 │
    ┌────────────▼──────────────┐
    │  后台处理（asyncio 任务）   │
    │  ├─ 计算 Token 数          │
    │  ├─ 检查压缩阈值           │
    │  ├─ 生成递归摘要           │
    │  └─ 保存到数据库           │
    └────────────┬──────────────┘
                 │
    ┌────────────▼──────────────┐
    │    数据库持久化存储         │
    │  ├─ 摘要内容               │
    │  ├─ 压缩进度               │
    │  └─ 时间戳                 │
    └────────────────────────────┘
```

---

## 工作流程详解

### 1️⃣ inlet 阶段：消息注入与压缩视图构建

**目标**：快速应用已有摘要，构建压缩消息视图

**流程**：

```
输入：所有消息列表
  │
  ├─► 从数据库加载摘要记录
  │     │
  │     ├─► 找到 ✓ ─────┐
  │     └─► 未找到 ───┐ │
  │                  │ │
  ├──────────────────┴─┼─► 存在摘要？
  │                    │
  │                ┌───▼───┐
  │                │  是   │  否
  │                └───┬───┴───┐
  │                    │       │
  │        ┌───────────▼─┐   ┌─▼─────────┐
  │        │ 构建压缩视图  │   │ 使用原始 │
  │        │ [H] + [T]   │   │ 消息列表 │
  │        └───────┬─────┘   └─┬────────┘
  │                │          │
  │    ┌───────────┴──────────┘
  │    │
  │    └─► 组合消息：
  │           • 头部（keep_first）
  │           • 摘要注入到首条
  │           • 尾部（keep_last）
  │
  └─────► 返回压缩消息列表
           ⏱️ 耗时 <10ms
```

**关键参数**：
- `keep_first`：保留前 N 条消息（默认 1）
- `keep_last`：保留后 N 条消息（默认 6）
- 摘要注入位置：首条消息的内容前

**示例**：
```python
# 原始：20 条消息
消息1: [系统提示]
消息2-14: [历史对话]
消息15-20: [最近对话]

# inlet 后（存在摘要）：7 条消息
消息1: [系统提示 + 【历史摘要】...]  ◄─ 摘要已注入
消息15-20: [最近对话]  ◄─ 保留后6条
```

---

### 2️⃣ outlet 阶段：后台异步处理

**目标**：计算 Token 数、检查阈值、生成摘要（不阻塞响应）

**流程**：

```
LLM 响应完成
  │
  └─► outlet 处理
      │
      └─► 启动后台异步任务（asyncio.create_task）
          │
          ├─► 立即返回给用户 ✓
          │   （不等待后台任务完成）
          │
          └─► 后台执行 _check_and_generate_summary_async
              │
              ├─► 在后台线程中计算 Token 数
              │   (await asyncio.to_thread)
              │
              ├─► 获取模型阈值配置
              │   • 优先使用 model_thresholds 中的配置
              │   • 回退到全局 compression_threshold_tokens
              │
              ├─► 检查是否触发压缩
              │   if current_tokens >= threshold:
              │
              └─► 触发摘要生成流程
```

**时序图**：
```
时间线：
│
├─ T0: LLM 响应完成
│
├─ T1: outlet 被调用
│       └─► 启动后台任务
│       └─► 立即返回 ✓
│
├─ T2: 用户收到响应 ✓✓✓
│
└─ T3-T10: 后台任务执行
            ├─ 计算 Token
            ├─ 检查阈值
            ├─ 调用 LLM 生成摘要
            └─ 保存到数据库
```

**关键特性**：
- ✅ 用户响应不受影响
- ✅ Token 计算不阻塞请求
- ✅ 摘要生成异步进行

---

### 3️⃣ Token 计数与阈值检查

**工作流程**：

```
后台线程执行 _check_and_generate_summary_async
│
├─► Step 1: 计算当前 Token 总数
│   │
│   ├─ 遍历所有消息
│   ├─ 处理多模态内容（提取文本部分）
│   ├─ 使用 o200k_base 编码计数
│   └─ 返回 total_tokens
│
├─► Step 2: 获取模型特定阈值
│   │
│   ├─ 模型 ID: gpt-4
│   ├─ 查询 model_thresholds
│   │
│   ├─ 存在配置？
│   │   ├─ 是 ✓ 使用该配置
│   │   └─ 否 ✓ 使用全局参数
│   │
│   ├─ compression_threshold_tokens（默认 64000）
│   └─ max_context_tokens（默认 128000）
│
└─► Step 3: 检查是否触发压缩
    │
    if current_tokens >= compression_threshold_tokens:
    │   └─► 触发摘要生成
    │
    else:
        └─► 无需压缩，任务结束
```

**Token 计数细节**：

```python
def _count_tokens(text):
    if tiktoken_available:
        # 使用 o200k_base（统一编码）
        encoding = tiktoken.get_encoding("o200k_base")
        return len(encoding.encode(text))
    else:
        # 回退：字符估算
        return len(text) // 4
```

**模型阈值优先级**：
```
优先级 1: model_thresholds["gpt-4"]
优先级 2: model_thresholds["gemini-2.5-flash"]
优先级 3: 全局 compression_threshold_tokens
```

---

### 4️⃣ 递归摘要生成

**核心机制**：将旧摘要与新消息合并，生成更新的摘要

**工作流程**：

```
触发 _generate_summary_async
│
├─► Step 1: 加载旧摘要
│   │
│   ├─ 从数据库查询
│   ├─ 获取 previous_summary
│   └─ 获取 compressed_message_count（上次压缩进度）
│
├─► Step 2: 确定待压缩消息范围
│   │
│   ├─ start_index = max(compressed_count, keep_first)
│   ├─ end_index = len(messages) - keep_last
│   │
│   ├─ 提取 messages[start_index:end_index]
│   └─ 这是【新增对话】部分
│
├─► Step 3: 构建 LLM 提示词
│   │
│   ├─ 【已有摘要】= previous_summary
│   ├─ 【新增对话】= 格式化的新消息
│   │
│   └─ 提示词模板：
│       "将【已有摘要】和【新增对话】合并..."
│
├─► Step 4: 调用 LLM 生成摘要
│   │
│   ├─ 模型选择：summary_model（若配置）或当前模型
│   ├─ 参数：
│   │   • max_tokens = max_summary_tokens（默认 4000）
│   │   • temperature = summary_temperature（默认 0.3）
│   │   • stream = False
│   │
│   └─ 返回 new_summary
│
├─► Step 5: 保存摘要到数据库
│   │
│   ├─ 更新 chat_summary 表
│   ├─ summary = new_summary
│   ├─ compressed_message_count = end_index
│   └─ updated_at = now()
│
└─► Step 6: 记录日志
    └─ 摘要长度、压缩进度、耗时等
```

**递归摘要示例**：

```
第一轮压缩：
  旧摘要: 无
  新消息: 消息2-14（13条）
  生成: Summary_V1
  
  保存: compressed_message_count = 14

第二轮压缩：
  旧摘要: Summary_V1
  新消息: 消息15-28（从14开始）
  生成: Summary_V2 = LLM(Summary_V1 + 新消息14-28)
  
  保存: compressed_message_count = 28

结果：
  ✓ 早期信息得以保留（通过 Summary_V1）
  ✓ 新信息与旧摘要融合
  ✓ 历史连贯性维护
```

---

## Token 计数机制

### 编码方案

```
┌─────────────────────────────────┐
│   _count_tokens(text)           │
├─────────────────────────────────┤
│ 1. tiktoken 可用？              │
│    ├─ 是 ✓                      │
│    │  └─ use o200k_base         │
│    │     (最新模型适配)          │
│    │                             │
│    └─ 否 ✓                      │
│       └─ 字符估算               │
│          (1 token ≈ 4 chars)   │
└─────────────────────────────────┘
```

### 多模态内容处理

```python
# 消息结构
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "描述图片..."},
        {"type": "image_url", "image_url": {...}},
        {"type": "text", "text": "更多描述..."}
    ]
}

# Token 计数
提取所有 text 部分 → 合并 → 计数
图片部分被忽略（不消耗文本 token）
```

### 计数流程

```
_calculate_messages_tokens(messages, model)
│
├─► 遍历每条消息
│   │
│   ├─ content 是列表？
│   │   ├─ 是 ✓ 提取所有文本部分
│   │   └─ 否 ✓ 直接使用
│   │
│   └─ _count_tokens(content)
│
└─► 累加所有 Token 数
```

---

## 递归摘要机制

### 保证历史连贯性的核心原理

```
传统压缩方式（有问题）：
时间线：
  消息1-50 ─► 生成摘要1 ─► 保留 [摘要1 + 消息45-50]
              │
              消息51-100 ─► 生成摘要2 ─► 保留 [摘要2 + 消息95-100]
                           └─► ❌ 摘要1 丢失！早期信息无法追溯

递归摘要方式（本实现）：
时间线：
  消息1-50 ──► 生成摘要1 ──► 保存
              │
              摘要1 + 消息51-100 ──► 生成摘要2 ──► 保存
                                     └─► ✓ 摘要1 信息融入摘要2
                                     ✓ 历史信息连贯保存
```

### 工作机制

```
inlet 阶段：
  摘要库查询
    │
    ├─ previous_summary（已有摘要）
    └─ compressed_message_count（压缩进度）

outlet 阶段：
  如果 current_tokens >= threshold:
    │
    ├─ 新消息范围：
    │  [compressed_message_count : len(messages) - keep_last]
    │
    └─ LLM 处理：
       Input:  previous_summary + 新消息
       Output: 更新的摘要（含早期信息 + 新信息）
       
  保存进度：
    └─ compressed_message_count = end_index
       （下次压缩从这里开始）
```

---

## 配置指南

### 全局配置

```python
Valves(
    # Token 阈值
    compression_threshold_tokens=64000,  # 触发压缩
    max_context_tokens=128000,           # 硬性上限
    
    # 消息保留策略
    keep_first=1,      # 保留首条（系统提示）
    keep_last=6,       # 保留末6条（最近对话）
    
    # 摘要模型
    summary_model="gemini-2.5-flash",  # 快速经济
    
    # 摘要参数
    max_summary_tokens=4000,
    summary_temperature=0.3,
)
```

### 模型特定配置

```python
model_thresholds = {
    "gpt-4": {
        "compression_threshold_tokens": 8000,
        "max_context_tokens": 32000
    },
    "gemini-2.5-flash": {
        "compression_threshold_tokens": 10000,
        "max_context_tokens": 40000
    },
    "llama-70b": {
        "compression_threshold_tokens": 20000,
        "max_context_tokens": 80000
    }
}
```

### 配置选择建议

```
场景1：长对话成本优化
  compression_threshold_tokens: 32000  ◄─ 更早触发
  keep_last: 4                         ◄─ 保留少一些
  
场景2：质量优先
  compression_threshold_tokens: 100000 ◄─ 晚触发
  keep_last: 10                        ◄─ 保留多一些
  max_summary_tokens: 8000             ◄─ 更详细摘要
  
场景3：平衡方案（推荐）
  compression_threshold_tokens: 64000  ◄─ 默认
  keep_last: 6                         ◄─ 默认
  summary_model: "gemini-2.5-flash"   ◄─ 快速经济
```

---

## 最佳实践

### 1️⃣ 摘要模型选择

```
推荐模型：
  ✅ gemini-2.5-flash    快速、经济、质量好
  ✅ deepseek-v3         成本低、速度快
  ✅ gpt-4o-mini         通用、质量稳定

避免：
  ❌ 流水线（Pipe）模型  可能不支持标准 API
  ❌ 本地模型            容易超时、影响体验
```

### 2️⃣ 阈值调优

```
Token 计数验证：
  1. 启用 debug_mode
  2. 观察实际 Token 数
  3. 根据需要调整阈值
  
  # 日志示例
  [🔍 后台计算] Token 数: 45320
  [🔍 后台计算] 未触发压缩阈值 (Token: 45320 < 64000)
```

### 3️⃣ 消息保留策略

```
keep_first 配置：
  通常值: 1（保留系统提示）
  某些场景: 0（系统提示在摘要中）
  
keep_last 配置：
  通常值: 6（保留最近对话）
  长对话: 8-10（更多最近对话）
  短对话: 3-4（节省 Token）
```

### 4️⃣ 监控与维护

```
关键指标：
  • 摘要生成耗时
  • Token 节省率
  • 摘要质量（通过对话体验）
  
数据库维护：
  # 定期清理过期摘要
  DELETE FROM chat_summary
  WHERE updated_at < NOW() - INTERVAL '30 days'
  
  # 统计压缩效果
  SELECT 
    COUNT(*) as total_summaries,
    AVG(compressed_message_count) as avg_compressed
  FROM chat_summary
```

### 5️⃣ 故障排除

```
问题：摘要未生成
  检查项：
    1. Token 数是否达到阈值？
       → debug_mode 查看日志
    2. summary_model 是否配置正确？
       → 确保模型存在且可用
    3. 数据库连接是否正常？
       → 检查 DATABASE_URL

问题：inlet 响应变慢
  检查项：
    1. keep_first/keep_last 是否过大？
    2. 摘要数据是否过大？
    3. 消息数是否过多？
    
问题：摘要质量下降
  调整方案：
    1. 增加 max_summary_tokens
    2. 降低 summary_temperature（更确定性）
    3. 更换摘要模型
```

---

## 性能参考

### 时间开销

```
inlet 阶段：
  ├─ 数据库查询: 1-2ms
  ├─ 摘要注入: 2-3ms
  └─ 总计: <10ms ✓ (不影响用户体验)

outlet 阶段：
  ├─ 启动后台任务: <1ms
  └─ 立即返回: ✓ (无等待)

后台处理（不阻塞用户）：
  ├─ Token 计数: 10-50ms
  ├─ LLM 调用: 1-5 秒
  ├─ 数据库保存: 1-2ms
  └─ 总计: 1-6 秒 (后台进行)
```

### Token 节省示例

```
场景：20 条消息对话

未压缩：
  总消息: 20 条
  预估 Token: 8000 个

压缩后（keep_first=1, keep_last=6）：
  头部消息: 1 条 (1600 Token)
  摘要: ~800 Token (嵌入在头部)
  尾部消息: 6 条 (3200 Token)
  总计: 7 条有效输入 (~5600 Token)
  
节省：8000 - 5600 = 2400 Token (30% 节省)

随对话变长，节省比例可达 65% 以上
```

---

## 数据流图

```
用户消息
  ↓
[inlet] 摘要注入器
  ├─ 数据库 ← 查询摘要
  ├─ 摘要注入到首条消息
  └─ 返回压缩消息列表
  ↓
LLM 处理
  ├─ 调用语言模型
  ├─ 生成响应
  └─ 返回给用户 ✓✓✓
  ↓
[outlet] 后台处理（asyncio 任务）
  ├─ 计算 Token 数
  ├─ 检查阈值
  ├─ [if 需要] 调用 LLM 生成摘要
  │  ├─ 加载旧摘要
  │  ├─ 提取新消息
  │  ├─ 构建提示词
  │  └─ 调用 LLM
  ├─ 保存新摘要到数据库
  └─ 记录日志
  ↓
数据库持久化
  └─ chat_summary 表更新
```

---

## 总结

| 阶段 | 职责 | 耗时 | 特点 |
|------|------|------|------|
| **inlet** | 摘要注入 | <10ms | 快速、无计算 |
| **LLM** | 生成回复 | 变量 | 正常流程 |
| **outlet** | 启动后台 | <1ms | 不阻塞响应 |
| **后台处理** | Token 计算、摘要生成、数据保存 | 1-6s | 异步执行 |

**核心优势**：
- ✅ 用户响应不受影响
- ✅ Token 消耗显著降低
- ✅ 历史信息连贯保存
- ✅ 灵活的配置选项
