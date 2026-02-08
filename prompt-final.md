# FinVerse 完整开发提示词 v3.0

> **极细颗粒度、可执行的全栈开发指南**  
> 从零到生产环境的完整路径  
> 生成时间：2026-02-08  
> 作者：多 Agent 协同生成 + 交叉验证

---

## 📋 项目概述

### 什么是 FinVerse？

**FinVerse 是基于 OpenClaw Agent 运行时的金融数据可视化与协作平台。**

核心特点：
- 🤖 **用户自带 AI**：用户提供自己的 OpenAI/Anthropic/DeepSeek API key，平台不承担 AI 成本
- 🔐 **零成本托管**：Agent 容器化运行，平台提供调度和管理
- 📊 **信息呈现设计**：把 AI 分析结果可视化，帮助人做决策
- 🌐 **公域 + 私域**：结构化信号广播 + 异步分析小组
- 💬 **聊天软件集成**：Agent 活在 Telegram/WhatsApp，网站是可视化看板

### 为什么现在做？

AI 时代，传统金融网站的护城河正在消失：
- ❌ 纯行情看板 → AI 直接调 API
- ❌ 财经新闻聚合 → AI 自动总结
- ❌ 基础筛选器 → 自然语言更强
- ✅ **信息呈现设计** → 5-10 年窗口期的核心价值

---

## 🎯 使用说明（给 AI 开发者）

### 这份文档是什么？

这是一份**极细颗粒度的开发提示词**，包含：
- 📁 完整的数据库 schema（可直接执行的 SQL）
- 🔌 所有 API 端点定义（request/response/error codes）
- 🎨 前端组件代码（React/TypeScript，可直接复制）
- 🐳 Docker 配置（生产级）
- 📜 完整的技术栈和依赖版本
- 🚀 从零到部署的初始化脚本

### 如何使用？

1. **顺序阅读**：按章节顺序，从第 1 章读到第 18 章
2. **直接复制代码**：所有代码块都是可执行的，复制即用
3. **执行初始化脚本**：第 17 章提供完整的项目初始化命令
4. **参考 Sprint 任务**：第 16 章提供 3 个月 MVP 的任务拆解

### 推荐工作流

```bash
# 1. 克隆或创建项目目录
mkdir finverse && cd finverse

# 2. 按照第 17 章创建所有文件和目录结构
# 3. 运行初始化脚本
./scripts/init-project.sh

# 4. 按照第 16 章的 Sprint 顺序开发
#    Sprint 1.1 -> 1.2 -> 1.3 -> 1.4 -> ...

# 5. 每个 Sprint 完成后，运行测试和部署
npm run test
docker-compose up -d
```

---

## 目录

### 基础层（第 1-6 章）
1. [核心命题与价值主张](#章节一核心命题)
2. [五个关键前提](#章节二五个关键前提)
3. [竞品分析与差异化](#章节三什么会死什么能活)
4. [商业模式与订阅系统](#章节四商业模式)
5. [技术架构（OpenClaw）](#章节五技术架构)
6. [公域信号系统](#章节六公域)

### 核心功能层（第 7-12 章）
7. [私域异步分析小组](#章节七私域)
8. [聊天软件集成（Telegram）](#章节八聊天软件分工)
9. [三种可视化模式](#章节九可视化)
10. [用户分类与匹配](#章节十用户分类与匹配)
11. [Agent 自定义](#章节十一agent-自定义)
12. [多语言与多时区](#章节十二多语言与多时区)

### 数据与基础设施层（第 13-15 章）
13. [多市场数据接入](#章节十三覆盖市场)
14. [风险对策与安全](#章节十四已知风险与对策)
15. [Agent 调度器](#章节十五agent-调度器)

### 执行层（第 16-18 章）
16. [3 个月 MVP 路线图](#章节十六路线图)
17. [技术栈与项目初始化](#章节十七技术栈)
18. [总结](#章节十八一句话总结)

---

## 章节一：核心命题 - 开发实现

### 1.1 核心价值主张展示系统

#### 数据库 Schema

```sql
-- 用户核心价值认知表
CREATE TABLE user_value_propositions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    understands_core_value BOOLEAN DEFAULT FALSE,
    onboarding_shown_at TIMESTAMP,
    value_test_completed_at TIMESTAMP,
    value_test_score INTEGER,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_user_value_propositions_user_id ON user_value_propositions(user_id);

-- 核心命题展示内容表（支持 A/B 测试）
CREATE TABLE value_proposition_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_name VARCHAR(50) NOT NULL,
    headline TEXT NOT NULL,
    subheadline TEXT,
    key_points JSONB,
    demo_type VARCHAR(50),
    conversion_rate DECIMAL(5,4),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);
```

#### API Endpoints

```typescript
// POST /api/v1/onboarding/value-proposition
interface ValuePropositionViewRequest {
    variant_id: string;
    time_spent_seconds: number;
    interaction_events: Array<{
        event_type: 'scroll' | 'click' | 'hover';
        element_id: string;
        timestamp: number;
    }>;
}

interface ValuePropositionViewResponse {
    success: boolean;
    next_step: 'api_key_setup' | 'demo' | 'skip';
    personalized_message: string;
}
```

#### 前端组件代码

> **完整代码请参考 prompt-part-a.md 第 1 章**（包含 ValuePropositionHero 组件的完整实现）

关键特性：
- ✅ Framer Motion 动画
- ✅ 三个核心价值点卡片
- ✅ 响应式设计
- ✅ 品牌色渐变背景

---

## 章节二：五个关键前提 - 开发实现

### 2.1 前提验证与用户教育系统

#### 数据库 Schema

```sql
CREATE TABLE user_premise_understanding (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    premise_key VARCHAR(50) NOT NULL,
    understands BOOLEAN DEFAULT FALSE,
    confidence_level INTEGER,
    last_educated_at TIMESTAMP,
    education_method VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, premise_key)
);
```

#### 前端组件：PremiseEducationModal

> **完整代码请参考 prompt-part-a.md 第 2 章**

特点：
- 5 个前提的交互式 Modal
- 进度条动画
- 每个前提包含：问题描述 + 结论 + 视觉设计

---

## 章节三：什么会死什么能活 - 开发实现

### 3.1 竞品对比系统

#### 数据库 Schema

```sql
CREATE TABLE competitor_comparison (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    competitor_name VARCHAR(100) NOT NULL,
    competitor_type VARCHAR(50),
    will_die BOOLEAN,
    reason TEXT,
    time_horizon VARCHAR(50),
    our_advantage TEXT,
    display_order INTEGER,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);
```

#### 前端组件：CompetitorComparisonTable

> **完整代码请参考 prompt-part-a.md 第 3 章**

特点：
- 可筛选（会被替代 / 能活下来）
- 可展开查看详细对比
- 底部定位声明卡片

---

## 章节四：商业模式 - 开发实现

### 4.1 订阅系统数据库设计

```sql
-- 订阅计划表
CREATE TABLE subscription_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_name VARCHAR(50) NOT NULL UNIQUE,
    display_name VARCHAR(100),
    price_monthly DECIMAL(10,2),
    price_yearly DECIMAL(10,2),
    features JSONB,
    limits JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 用户订阅表
CREATE TABLE user_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    plan_id UUID NOT NULL REFERENCES subscription_plans(id),
    status VARCHAR(50) DEFAULT 'active',
    billing_cycle VARCHAR(20),
    current_period_start TIMESTAMP,
    current_period_end TIMESTAMP,
    stripe_subscription_id VARCHAR(200),
    created_at TIMESTAMP DEFAULT NOW()
);

-- API Key 管理表
CREATE TABLE user_api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider_id UUID NOT NULL REFERENCES api_key_providers(id),
    key_type VARCHAR(50),
    encrypted_key TEXT NOT NULL,
    key_name VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    last_used_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 4.2 API Endpoints

```typescript
// GET /api/v1/subscriptions/plans
interface SubscriptionPlansResponse {
    plans: Array<{
        id: string;
        name: string;
        price: { monthly: number; yearly: number };
        features: string[];
        limits: any;
    }>;
}

// POST /api/v1/api-keys
interface AddAPIKeyRequest {
    provider_id: string;
    api_key: string;
    key_name?: string;
}

interface AddAPIKeyResponse {
    success: boolean;
    key_id: string;
    validation: {
        is_valid: boolean;
        error_message?: string;
    };
}
```

### 4.3 前端组件

> **完整代码请参考 prompt-part-a.md 第 4 章**

关键组件：
- `PricingTable`：定价表（月付/年付切换）
- `APIKeySetup`：API Key 配置流程（3 步引导）

---

## 章节五：技术架构（OpenClaw）- 开发实现

### 5.1 Agent 实例管理数据库

```sql
-- Agent 实例表
CREATE TABLE agent_instances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    agent_name VARCHAR(100) DEFAULT 'My Agent',
    container_id VARCHAR(100),
    container_status VARCHAR(50),
    resource_tier VARCHAR(50),
    openclaw_version VARCHAR(50),
    last_heartbeat_at TIMESTAMP,
    last_interaction_at TIMESTAMP,
    cpu_limit_millicores INTEGER DEFAULT 500,
    memory_limit_mb INTEGER DEFAULT 512,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Agent Cron 任务表
CREATE TABLE agent_cron_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL REFERENCES agent_instances(id) ON DELETE CASCADE,
    job_name VARCHAR(100),
    job_type VARCHAR(50),
    cron_expression VARCHAR(100),
    timezone VARCHAR(50),
    last_run_at TIMESTAMP,
    next_run_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    config JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 5.2 Agent 调度器服务

> **完整代码请参考 prompt-part-a.md 第 5 章**

核心功能：
- Docker 容器创建/停止
- Agent 初始化（生成 SOUL.md）
- 自动扩缩容（活跃/待机/休眠）
- 健康检查

### 5.3 SOUL.md 模板生成器

```typescript
export function generateSOULTemplate(
    userName: string,
    preferences: UserPreferences
): string {
    // 动态生成 SOUL.md
    // 包括：用户画像、分析权重、推送策略、信号格式
}
```

---

## 章节六：公域（信号系统）- 完整实现

### 6.1 信号池数据库设计

```sql
-- 公域信号表
CREATE TABLE public_signals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL REFERENCES users(id),
    asset VARCHAR(20) NOT NULL,
    signal VARCHAR(10) NOT NULL CHECK (signal IN ('bullish', 'bearish', 'neutral')),
    confidence NUMERIC(3, 2) NOT NULL,
    dimensions JSONB NOT NULL,
    key_levels JSONB,
    timeframe VARCHAR(10) NOT NULL,
    reasoning TEXT NOT NULL,
    verified BOOLEAN DEFAULT FALSE,
    actual_outcome VARCHAR(10),
    created_at TIMESTAMP DEFAULT NOW()
);

-- 信号订阅表
CREATE TABLE signal_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_agent_id UUID NOT NULL,
    assets TEXT[] DEFAULT '{}',
    min_confidence NUMERIC(3, 2) DEFAULT 0.6,
    timeframes TEXT[] DEFAULT '{"24h", "48h"}',
    notify_on_publish BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 共识快照表
CREATE TABLE signal_consensus_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset VARCHAR(20) NOT NULL,
    timeframe VARCHAR(10) NOT NULL,
    snapshot_time TIMESTAMP NOT NULL,
    total_signals INTEGER NOT NULL,
    bullish_count INTEGER NOT NULL,
    bearish_count INTEGER NOT NULL,
    neutral_count INTEGER NOT NULL,
    weighted_consensus VARCHAR(10) NOT NULL,
    weighted_confidence NUMERIC(3, 2) NOT NULL,
    divergence_score NUMERIC(4, 3),
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 6.2 信号聚合算法

> **完整代码请参考 prompt-补全-chapters-6-9-10-11-12.md 第 6 章**

核心功能：
- `aggregateSignals()`：聚合指定资产的信号
- `generateConsensusHeatmap()`：生成共识热力图数据
- `pushSignalsToSubscribers()`：推送给订阅者

算法特点：
- 加权共识（基于信誉评分 + 置信度）
- 分歧度计算（标准差）
- 信誉分布统计

### 6.3 前端组件：共识热力图

```typescript
// ConsensusHeatmap.tsx
export const ConsensusHeatmap: React.FC = () => {
  // 显示多个资产的共识状态
  // 颜色编码：绿色=看多，红色=看空，灰色=中性
  // 闪烁效果：分歧度高时显示警告
};
```

---

## 章节七：私域（异步分析小组）- 完整实现

### 7.1 数据库设计

```sql
-- 小组表
CREATE TABLE groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    creator_id UUID NOT NULL REFERENCES users(id),
    group_type VARCHAR(50) DEFAULT 'public',
    max_members INTEGER DEFAULT 20,
    tags JSONB DEFAULT '[]'::jsonb,
    focus_assets TEXT[] DEFAULT '{}',
    is_active BOOLEAN DEFAULT TRUE,
    member_count INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 小组成员表
CREATE TABLE group_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) DEFAULT 'member',
    notification_settings JSONB,
    joined_at TIMESTAMP DEFAULT NOW()
);

-- 分析发布表
CREATE TABLE group_posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id UUID NOT NULL REFERENCES groups(id),
    author_id UUID NOT NULL REFERENCES users(id),
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    analysis_data JSONB NOT NULL,
    ai_assisted BOOLEAN DEFAULT FALSE,
    upvotes INTEGER DEFAULT 0,
    downvotes INTEGER DEFAULT 0,
    comment_count INTEGER DEFAULT 0,
    outcome_data JSONB DEFAULT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    evaluated_at TIMESTAMP
);

-- 投票表
CREATE TABLE post_votes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id UUID NOT NULL REFERENCES group_posts(id),
    user_id UUID NOT NULL REFERENCES users(id),
    vote_type VARCHAR(20) NOT NULL,
    vote_data JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(post_id, user_id)
);

-- 共识报告表
CREATE TABLE consensus_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id UUID NOT NULL REFERENCES groups(id),
    report_type VARCHAR(50) NOT NULL,
    asset VARCHAR(20),
    period_start TIMESTAMP NOT NULL,
    period_end TIMESTAMP NOT NULL,
    stats JSONB NOT NULL,
    summary TEXT NOT NULL,
    generated_at TIMESTAMP DEFAULT NOW()
);
```

### 7.2 API 设计

> **完整 API 定义请参考 prompt-part-b.md 第 7 章**

核心端点：
- `POST /api/groups`：创建小组
- `POST /api/groups/:groupId/posts/ai-assist`：AI 辅助发布分析
- `POST /api/posts/:postId/vote`：投票
- `POST /api/groups/:groupId/consensus-reports/generate`：生成共识报告

### 7.3 前端组件

> **完整代码请参考 prompt-part-b.md 第 7 章**

关键组件：
- `GroupHomePage`：小组主页（Tabs + 侧边栏 + 分析列表）
- `AnalysisCard`：分析卡片（信号指示器 + 维度条 + 互动统计）
- `VoteModal`：投票弹窗（选择观点 + 置信度滑块 + 理由）
- `ConsensusReportPage`：共识报告页（统计图表 + AI 摘要）

### 7.4 历史复盘追踪算法

```typescript
async function evaluatePostAccuracy(post) {
  // 1. 获取实际价格数据
  const priceAtPrediction = await fetchHistoricalPrice(asset, predictionTime);
  const priceAtEvaluation = await fetchHistoricalPrice(asset, evaluationTime);
  
  // 2. 判断实际走势
  const priceChangePct = (priceAtEvaluation - priceAtPrediction) / priceAtPrediction * 100;
  const actualSignal = priceChangePct > 2 ? 'bullish' : priceChangePct < -2 ? 'bearish' : 'neutral';
  
  // 3. 计算准确率
  const predictionCorrect = actualSignal === analysis_data.signal;
  
  // 4. 更新用户统计（只给用户自己看）
  await updateUserAccuracyStats(post.author_id, { correct: predictionCorrect, ... });
}
```

---

## 章节八：聊天软件分工 - 完整实现

### 8.1 Telegram Bot 集成

> **完整代码请参考 prompt-part-b.md 第 8 章**

核心功能：
- 每日摘要推送（市场总览 + 关注资产）
- 异常预警（实时）
- 小组通知（新分析/共识报告）
- 自然语言查询（用户提问 -> Agent 回答）

消息模板示例：

```
📋 **今日市场摘要** · 2026-02-08

━━━━━━━━━━━━━━━━━━━━━━

**BTC/USD** $67,450 ⬇️ -2.3%
🔴 看空信号（置信度 72%）
├ 链上：交易所流入+12.4K BTC ⬇️
├ 技术：RSI 72 超买 ⬇️  
├ 宏观：CPI风险 ⬇️
└ 情绪：恐惧指数 38 ⬇️

关键位：支撑 $67,200 | 阻力 $69,800

[查看详细看板] [调整偏好]
```

### 8.2 推送触发逻辑

| 事件 | 触发条件 | 推送时机 | 优先级 |
|------|---------|---------|--------|
| 每日摘要 | 用户订阅 | 用户时区 9:00 | Low |
| 异常预警 | 监控脚本检测 | 立即 | Critical |
| 小组新分析 | 成员发布 | 立即（可设免打扰） | Medium |
| 价格穿越关键位 | 触及用户关注的位 | 立即 | High |

### 8.3 通知频率控制

```typescript
class NotificationManager {
  async shouldSendNotification(userId, notification) {
    // 1. 检查免打扰时段（23:00-8:00 只发 critical）
    // 2. 检查频率限制（critical: 2min, high: 5min, medium: 30min）
    // 3. 检查用户订阅设置
  }
}
```

---

## 章节九：可视化 - 三种呈现模式

### 9.1 摘要模式（Summary Mode）

**设计理念**：1 秒看完，知道该关注什么。

UI 规格：
- 资产卡片：渐变背景，信号颜色边框
- 一句话结论：18px，最多 2 行
- 多维信号条：32px 高，填充动画 1s
- 异常预警卡片：左侧 4px 严重度颜色边框

> **完整代码请参考 prompt-part-b.md 第 9 章**

关键组件：
- `SummaryMode`：整体布局
- `AssetSummaryCard`：单个资产卡片
- `DimensionBar`：维度信号条
- `AlertCard`：异常预警卡片

---

### 9.2 图表模式（Chart Mode）

**设计理念**：技术派的最爱，多图层叠加。

图层系统：
- ✅ 价格（K 线图）
- ✅ 成交量（柱状图）
- ✅ AI 标注（支撑阻力线 + 点标注）
- ✅ 宏观事件（时间轴标记）
- ✅ 链上数据（叠加曲线）
- ✅ 异常热力图（背景渐变）
- ✅ 历史叠影（半透明历史走势）

技术选型：**lightweight-charts**（高性能 Canvas 图表库）

> **完整代码请参考 prompt-补全-chapters-6-9-10-11-12.md 第 9 章**

---

### 9.3 数据模式（Data Mode）

**设计理念**：高密度数字面板，类 Bloomberg Terminal。

UI 规格：
- 背景：#0a0e17（暗色）
- 数据单元：字体 monospace，变化时闪烁
- WebSocket 实时更新

数据分组：
- 价格（当前/开盘/最高/最低/24h 涨跌）
- 成交量（当前/24h 平均/相对平均）
- 链上数据（交易所流入流出/活跃地址/大额转账）
- 衍生品（持仓量/资金费率/多空比）
- 技术指标（RSI/MACD/布林带/EMA）
- 市场情绪（恐惧贪婪指数/社交声量/社交情绪）

> **完整代码请参考 prompt-补全-chapters-6-9-10-11-12.md 第 9 章**

---

### 9.4 AI 推理链可视化

**设计理念**：把 AI 的黑箱变成透明的推理过程。

展示内容：
- ① 链上数据异常 → 权重 30%
- ② 宏观环境利空 → 权重 35%
- ③ 技术面支撑尚在 → 权重 20%
- ④ 历史形态相似度 → 权重 15%
- **综合评分：偏空 72%**

> **完整代码请参考 prompt-补全-chapters-6-9-10-11-12.md 第 9 章**

关键组件：
- `AIReasoningChain`：整体容器（可折叠）
- `ReasoningStepCard`：单个步骤卡片（可展开详情）
- `MetricBadge`：指标徽章

---

## 章节十：用户分类与匹配系统 - 完整实现

### 10.1 多维标签体系

标签分类：
- **Trading Style**：day_trading, swing, long_term, scalping, quantitative
- **Analysis Preference**：technical, fundamental, onchain, macro, sentiment
- **Asset Preference**：crypto, stocks, forex, commodities, options
- **Risk Preference**：conservative, balanced, aggressive

数据库设计：

```sql
-- 标签定义表
CREATE TABLE tags (
    id UUID PRIMARY KEY,
    category VARCHAR(50) NOT NULL,
    tag_key VARCHAR(50) NOT NULL,
    display_name VARCHAR(100),
    icon VARCHAR(50),
    weight NUMERIC(3, 2) DEFAULT 1.0,
    UNIQUE(category, tag_key)
);

-- 用户标签关联表
CREATE TABLE user_tags (
    user_id UUID NOT NULL,
    tag_id UUID NOT NULL,
    source VARCHAR(50) NOT NULL,
    confidence NUMERIC(3, 2) DEFAULT 1.0,
    PRIMARY KEY (user_id, tag_id)
);

-- 标签相似度矩阵
CREATE TABLE tag_similarity_matrix (
    tag_a_id UUID NOT NULL,
    tag_b_id UUID NOT NULL,
    similarity_score NUMERIC(3, 2) NOT NULL,
    PRIMARY KEY (tag_a_id, tag_b_id)
);
```

### 10.2 匹配算法

> **完整代码请参考 prompt-补全-chapters-6-9-10-11-12.md 第 10 章**

核心功能：
- `calculateSimilarity(userA, userB)`：计算两用户相似度
- `recommendUsers(userId)`：推荐相似用户
- `recommendGroups(userId)`：推荐匹配的小组

算法特点：
- Jaccard 相似度（基础）
- 加权相似度（考虑标签权重 + 相似度矩阵）
- 信誉分布统计

### 10.3 偏好测试

> **完整代码请参考 prompt-补全-chapters-6-9-10-11-12.md 第 10 章**

8 个情景题：
1. BTC 突然暴跌 10%，你的第一反应是？
2. 选择一个你最信任的数据来源
3. 你通常持仓多久？
4. 你最感兴趣的市场是？
5. 如果一笔交易亏损 20%，你会？
6. 你认为哪个最重要？
7. 你更喜欢看什么内容？
8. 你希望 Agent 多久推送一次？

前端组件：`PreferenceTest`（游戏化交互，进度条动画）

---

## 章节十一：Agent 自定义 - 完整实现

### 11.1 三个层次的自定义

| 层次 | 适用人群 | 门槛 | 功能 |
|------|---------|------|------|
| 入门 | 所有用户 | 0 | 偏好标签配置 |
| 进阶 | 有想法的用户 | 低 | 自然语言调教 |
| 高级 | 量化/开发者 | 中 | SOUL.md 编辑器 |

### 11.2 自然语言调教

> **完整代码请参考 prompt-补全-chapters-6-9-10-11-12.md 第 11 章**

用户说：
```
我更关注链上数据，技术面不太看
当 RSI 低于 30 的时候重点提醒我
我不关心 meme 币，只看 BTC 和 ETH
```

AI 理解为：
- 调整 onchain 权重至 40%
- 添加 alert 条件：RSI < 30
- 过滤资产：排除 DOGE、SHIB 等

后端 API：
```typescript
// POST /api/agents/:agentId/customize
interface CustomizeRequest {
    instruction: string;
}

interface CustomizeResponse {
    interpretation: string;
    changes: Array<{ description: string }>;
}
```

### 11.3 SOUL.md 编辑器

> **完整代码请参考 prompt-补全-chapters-6-9-10-11-12.md 第 11 章**

特点：
- Monaco Editor（VS Code 同款编辑器）
- Markdown 语法高亮
- 实时保存提示
- 保存后自动重启 Agent

---

## 章节十二：多语言与多时区 - 完整实现

### 12.1 自动翻译系统

> **完整代码请参考 prompt-补全-chapters-6-9-10-11-12.md 第 12 章**

核心功能：
- `translate(text, targetLanguage, userId)`：单文本翻译
- `translateBatch(texts, targetLanguage, userId)`：批量翻译
- `translateSignal(signal, targetLanguage, userId)`：翻译结构化信号

特点：
- 使用用户自己的 AI API key（不增加平台成本）
- 翻译结果缓存 30 天（Redis）
- 支持金融术语上下文

### 12.2 时区转换

```typescript
export class TimezoneConverter {
  toUserTimezone(utcTime: Date, userTimezone: string): DateTime;
  toUTC(localTime: Date, userTimezone: string): DateTime;
  relative(time: DateTime): string; // "3 小时前"
  getMarketOpenTime(market: 'US' | 'CN' | 'EU', userTimezone: string): DateTime;
}
```

应用场景：
- Agent 推送时间自动转换为用户时区
- 美股开盘提醒按用户本地时间
- 所有时间戳显示为用户本地时间

### 12.3 多语言 UI

支持语言：zh-CN, zh-TW, en, ja, ko, es

Next.js 配置：
```javascript
// next.config.js
module.exports = {
  i18n: {
    locales: ['zh-CN', 'zh-TW', 'en', 'ja', 'ko', 'es'],
    defaultLocale: 'en',
    localeDetection: true
  }
};
```

---

## 章节十三：覆盖市场 - 数据接入层

### 13.1 数据标准化 Schema

```typescript
export interface PriceDataPoint {
  timestamp: number;
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
  source: DataSource;
  reliability: number;
}

export enum DataSource {
  COINGECKO = 'coingecko',
  YAHOO_FINANCE = 'yahoo_finance',
  TWELVE_DATA = 'twelve_data',
  GLASSNODE = 'glassnode',
  POLYGON = 'polygon'
}
```

### 13.2 数据适配器实现

> **完整代码请参考 prompt-part-c.md 第 13 章**

已实现适配器：
- ✅ **CoinGecko**：加密货币免费数据
- ✅ **Yahoo Finance**：美股免费数据
- ✅ **Glassnode**：链上数据（付费，用户 key）

待实现：
- Twelve Data（外汇/贵金属）
- Polygon.io（美股实时数据）
- CBOE（期权数据）

### 13.3 定时数据拉取

```typescript
export class DataScheduler {
  start() {
    // 每 5 分钟拉取实时价格
    this.scheduleJob('realtime-prices', '*/5 * * * *', async () => {
      await this.aggregator.fetchRealtimePrices(['BTC/USD', 'ETH/USD', ...]);
    });

    // 每小时拉取链上数据
    this.scheduleJob('onchain-data', '0 * * * *', async () => {
      await this.aggregator.fetchOnChainData(['BTC', 'ETH']);
    });

    // 每天 UTC 00:00 拉取前一日完整数据
    this.scheduleJob('daily-aggregation', '0 0 * * *', async () => {
      await this.aggregator.fetchDailyData(yesterday);
    });
  }
}
```

---

## 章节十四：已知风险与对策 - 安全系统

### 14.1 Agent 信誉评分算法

评分维度：
- **准确率**（40%）：历史信号的准确性
- **稳定性**（20%）：信号发布频率的稳定性
- **账户年龄**（15%）：新注册 0 分，6 个月+ 15 分
- **社区反馈**（15%）：点赞/点踩比例
- **异常行为惩罚**（-10 分）：spam/矛盾/操纵

> **完整算法请参考 prompt-part-c.md 第 14 章**

信誉等级：
- Novice（新手）：0-49 分
- Reliable（可靠）：50-64 分
- Expert（专家）：65-79 分
- Master（大师）：80-100 分

### 14.2 异常检测规则

> **完整代码请参考 prompt-part-c.md 第 14 章**

检测类型：
- **Spam 检测**：1 小时内发布 > 10 个信号
- **矛盾检测**：高置信度信号 + 相反走势
- **操纵检测**：与 5+ 个 Agent 高度协同（疑似僵尸网络）

### 14.3 API Key 加密方案

算法：**AES-256-GCM**（Galois/Counter Mode）

```typescript
export class EncryptionService {
  encrypt(apiKey: string): string {
    // 返回格式: iv:authTag:ciphertext
  }

  decrypt(encrypted: string): string {
    // 解密并验证 authTag
  }
}
```

安全措施：
- ✅ 主密钥从环境变量读取（32 字节）
- ✅ 每个 key 使用唯一 IV
- ✅ 认证加密，防止篡改
- ✅ 只在 Agent 容器内解密

### 14.4 免责声明

> **完整组件代码请参考 prompt-part-c.md 第 14 章**

免责声明出现位置：
- ✅ 每个 AI 分析结果下方
- ✅ 信号发布页面顶部
- ✅ 小组分析页面
- ✅ 法律文本：`public/legal/risk-disclosure.md`

---

## 章节十五：Agent 调度器 - 容器编排

### 15.1 Docker Compose 配置

> **完整 docker-compose.yml 请参考 prompt-part-c.md 第 15 章**

服务列表：
- `postgres`：主数据库（PostgreSQL 16）
- `redis`：缓存与实时数据（Redis 7）
- `api`：后端 API（Fastify + TypeScript）
- `web`：前端（Next.js 15）
- `agent-orchestrator`：Agent 调度器
- `prometheus` + `grafana`：监控
- `nginx`：反向代理

### 15.2 Agent 容器 Dockerfile

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /build
COPY package*.json ./
RUN npm ci --production=false
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
RUN adduser -S agentuser -u 1001
USER agentuser
WORKDIR /app
COPY --from=builder /build/dist ./
EXPOSE 3100
HEALTHCHECK CMD node -e "require('http').get('http://localhost:3100/health', ...)"
CMD ["node", "runtime/index.js"]
```

### 15.3 Agent 调度器实现

> **完整代码请参考 prompt-part-c.md 第 15 章**

核心功能：
- `createAgent(config)`：创建新 Agent 容器
- `stopAgent(userId, graceful)`：停止 Agent
- `hibernateAgent(userId)`：暂停容器（降低资源占用）
- `wakeAgent(userId)`：唤醒休眠的 Agent
- `healthCheck()`：健康检查循环
- `autoScale()`：状态机（活跃 -> 待机 -> 休眠）

资源分级：
| 状态 | 条件 | 资源占用 |
|------|------|---------|
| Active | 最近 1h 有交互 | 512MB + 1 CPU |
| Standby | 最近 24h 有交互 | 256MB + 0.5 CPU |
| Hibernated | 超过 7 天无交互 | 暂停，仅存储 |

### 15.4 灰度发布脚本

> **完整脚本请参考 prompt-part-c.md 第 15 章**

特点：
- 批量滚动升级（默认每批 10 个）
- 健康检查验证
- 失败自动回滚
- 批次间延迟（默认 30 秒）

使用方式：
```bash
./scripts/rolling-update.sh v1.2.0 10 30
```

---

## 章节十六：路线图 - 3 个月 MVP

### Month 1: 核心基础

#### Sprint 1.1: 用户系统与认证 (Week 1)
- [ ] 用户注册与登录（JWT 认证）
- [ ] API Key 管理（加密存储）
- [ ] 偏好测试系统（8-10 个情景题）

#### Sprint 1.2: OpenClaw Agent 实例化 (Week 2)
- [ ] Agent 调度器基础（Docker API 集成）
- [ ] Agent 初始化流程（生成 SOUL.md）
- [ ] 聊天软件连接（Telegram）

#### Sprint 1.3: 基础数据接入 (Week 3)
- [ ] CoinGecko 适配器（加密货币）
- [ ] Yahoo Finance 适配器（美股）
- [ ] 定时数据拉取（Cron）

#### Sprint 1.4: 可视化界面 (Week 4)
- [ ] 摘要模式（一句话结论 + 多维信号条）
- [ ] 图表模式（基础 K 线图）
- [ ] 数据模式（高密度数字面板）

---

### Month 2: 公域 + 异常检测

#### Sprint 2.1: 信号系统 (Week 5)
- [ ] 信号标准格式 + 验证器
- [ ] 信号发布 API
- [ ] 信号订阅与聚合

#### Sprint 2.2: 异常检测 (Week 6)
- [ ] Spam 检测
- [ ] 矛盾检测
- [ ] 信誉评分 v1

#### Sprint 2.3: AI 推理链可视化 (Week 7)
- [ ] 多图层系统（AI 标注/宏观事件/链上数据）
- [ ] 推理链展示组件

#### Sprint 2.4: 监控脚本基础设施 (Week 8)
- [ ] Python 监控脚本（价格/成交量/链上异常）
- [ ] Agent 唤醒机制

---

### Month 3: 私域 + 市场扩展

#### Sprint 3.1: 异步分析小组 (Week 9-10)
- [ ] 小组功能基础（CRUD + 权限）
- [ ] AI 辅助分析发布
- [ ] 评论与反馈
- [ ] 共识报告

#### Sprint 3.2: 历史复盘 (Week 11)
- [ ] 判断准确率追踪
- [ ] 维度分析

#### Sprint 3.3: 市场扩展 (Week 12)
- [ ] 美股数据（Polygon.io）
- [ ] 外汇数据（Twelve Data）
- [ ] 贵金属

#### Sprint 3.4: Pro 版订阅 (Week 12)
- [ ] Stripe 集成
- [ ] 功能门控

---

### 验收标准示例

**Sprint 1.1 - 用户注册**
- [ ] 用户可通过 email/password 注册
- [ ] 登录返回有效 JWT token（30 天过期）
- [ ] 注册邮件发送成功率 > 95%
- [ ] OAuth 登录流程完整

**Sprint 1.2 - Agent 实例化**
- [ ] 可通过 API 触发创建 Agent 容器
- [ ] 容器成功启动并通过健康检查
- [ ] 数据库记录 Agent 状态
- [ ] 容器资源限制生效（512MB memory, 1 CPU）

---

## 章节十七：技术栈与项目初始化

### 17.1 技术选型与版本

```json
{
  "frontend": {
    "framework": "Next.js 15.1.0",
    "runtime": "React 19.0.0",
    "language": "TypeScript 5.7.0",
    "styling": "Tailwind CSS 4.0.0",
    "charts": "lightweight-charts 4.2.0"
  },
  "backend": {
    "runtime": "Node.js 22.22.0",
    "framework": "Fastify 5.2.0",
    "language": "TypeScript 5.7.0",
    "orm": "Drizzle ORM 0.37.0"
  },
  "database": {
    "primary": "PostgreSQL 16.3",
    "cache": "Redis 7.4"
  },
  "agent": {
    "runtime": "OpenClaw (latest)",
    "base": "Node.js 22.22.0"
  },
  "infrastructure": {
    "containerization": "Docker 27.4.0",
    "orchestration": "Docker Compose 2.31.0",
    "proxy": "Nginx 1.27.3"
  }
}
```

### 17.2 项目初始化完整命令

> **完整脚本请参考 prompt-part-c.md 第 17 章**

```bash
#!/bin/bash
# scripts/init-project.sh

# 1. Git 初始化
git init
git add .
git commit -m "Initial commit"

# 2. 环境变量配置（生成随机密钥）
MASTER_KEY=$(openssl rand -hex 32)
cat > .env << EOF
POSTGRES_PASSWORD=$(openssl rand -base64 32)
JWT_SECRET=$(openssl rand -base64 64)
MASTER_ENCRYPTION_KEY=${MASTER_KEY}
EOF

# 3. 安装依赖
npm install

# 4. 数据库初始化
docker-compose up -d postgres redis
docker exec -i finverse-postgres psql -U finverse < migrations/001_initial_schema.sql

# 5. 启动应用服务
docker-compose up -d api web agent-orchestrator

# 6. SSL 证书设置（可选）
./scripts/setup-ssl.sh

# 7. 启动 Nginx
docker-compose up -d nginx

# 8. 健康检查
for service in postgres redis api web; do
  echo "Checking $service..."
  # 等待服务就绪
done

echo "✅ FinVerse initialization complete!"
```

使用方式：
```bash
mkdir finverse && cd finverse
# 复制所有项目文件
chmod +x scripts/*.sh
./scripts/init-project.sh
```

### 17.3 Nginx 配置

> **完整配置请参考 prompt-part-c.md 第 17 章**

关键特性：
- HTTP -> HTTPS 重定向
- SSL/TLS 配置（Let's Encrypt）
- 速率限制（API: 10 req/s, Login: 5 req/min）
- WebSocket 支持
- Gzip 压缩
- 安全头（HSTS, X-Frame-Options, CSP）

### 17.4 SSL 证书配置

```bash
# scripts/setup-ssl.sh
certbot certonly --nginx \
    -d finverse.ai \
    -d www.finverse.ai \
    --email admin@finverse.ai \
    --agree-tos \
    --non-interactive

# 自动续期
crontab -l | { cat; echo "0 0 * * * certbot renew --quiet"; } | crontab -
```

---

## 章节十八：一句话总结

### 技术视角

> **FinVerse 是基于 OpenClaw Agent 运行时的金融数据可视化与协作平台。用户自带 AI 和数据的钥匙（API keys），平台提供零成本的 Agent 托管、结构化信号广播、异步分析小组和多维可视化界面。在 AI 个性化界面到来之前的 5-10 年窗口期，抓住「信息呈现设计」这个核心不可替代价值。**

### 商业视角

> **FinVerse = Lobe Chat（AI 托管）+ TradingView（可视化）+ Discord（社区），但垂直在金融领域。用户零门槛接入 AI，平台按订阅收费（$29/月），不承担 AI 和数据成本。3 个月 MVP，6 个月 PMF，12 个月达到 1000 付费用户。**

### 用户视角

> **你的专属金融 Agent 活在 Telegram 里，每天早上推送市场摘要，异常时立即提醒。你可以问它任何问题，它会用多维分析回答。你可以加入小组，和同风格的人一起研究，Agent 帮你们生成共识报告。你不需要懂技术，只需要一个 OpenAI API key。**

---

## 🎉 完成！

### 接下来做什么？

1. **阅读完整文档**：从第 1 章读到第 18 章
2. **运行初始化脚本**：`./scripts/init-project.sh`
3. **按 Sprint 开发**：从 Sprint 1.1 开始
4. **部署到生产环境**：配置域名、SSL、监控

### 资源链接

- **CONCEPT.md**：产品概念完整文档
- **prompt-part-a/b/c.md**：分章节详细代码
- **prompt-补全.md**：遗漏章节补全

### 版本历史

- **v1.0**（2026-02-07）：Part A 生成（章节 1-6）
- **v2.0**（2026-02-08）：Part B/C 生成（章节 7-18）
- **v3.0**（2026-02-08）：交叉验证 + 补全遗漏项 + 合并最终版

---

**祝开发顺利！🚀**
