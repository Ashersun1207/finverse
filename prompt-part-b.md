# FinVerse 开发提示词 Part B（第 7-12 章）

> 极细颗粒度可执行开发指南 · Agent B
> 生成时间：2026-02-08

---

## 七、私域——异步分析小组（方案 C）

### 7.1 数据库设计

#### 7.1.1 groups 表（分析小组）

```sql
CREATE TABLE groups (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  description TEXT,
  creator_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  -- 小组类型和设置
  group_type VARCHAR(50) DEFAULT 'public', -- public, private, invite_only
  max_members INTEGER DEFAULT 20,
  
  -- 标签匹配
  tags JSONB DEFAULT '[]'::jsonb, -- ["crypto", "short-term", "technical"]
  focus_assets TEXT[] DEFAULT '{}', -- ["BTC/USD", "ETH/USD"]
  trading_style VARCHAR(50), -- day-trading, swing, long-term
  analysis_preference VARCHAR(50), -- technical, fundamental, on-chain
  
  -- 状态
  is_active BOOLEAN DEFAULT true,
  member_count INTEGER DEFAULT 1,
  
  -- 统计
  total_posts INTEGER DEFAULT 0,
  total_consensus_reports INTEGER DEFAULT 0,
  
  -- 时间
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  
  -- 索引
  CONSTRAINT valid_group_type CHECK (group_type IN ('public', 'private', 'invite_only'))
);

CREATE INDEX idx_groups_creator ON groups(creator_id);
CREATE INDEX idx_groups_tags ON groups USING GIN(tags);
CREATE INDEX idx_groups_active ON groups(is_active) WHERE is_active = true;
CREATE INDEX idx_groups_focus_assets ON groups USING GIN(focus_assets);
```

#### 7.1.2 group_members 表（成员关系）

```sql
CREATE TABLE group_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  -- 成员角色
  role VARCHAR(20) DEFAULT 'member', -- creator, admin, member
  
  -- 权限
  can_post BOOLEAN DEFAULT true,
  can_vote BOOLEAN DEFAULT true,
  can_comment BOOLEAN DEFAULT true,
  
  -- 通知设置
  notification_settings JSONB DEFAULT '{
    "new_post": true,
    "new_comment": true,
    "consensus_report": true,
    "mention": true
  }'::jsonb,
  
  -- 统计
  total_posts INTEGER DEFAULT 0,
  total_votes INTEGER DEFAULT 0,
  total_comments INTEGER DEFAULT 0,
  
  -- 时间
  joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  last_active_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  
  UNIQUE(group_id, user_id)
);

CREATE INDEX idx_group_members_group ON group_members(group_id);
CREATE INDEX idx_group_members_user ON group_members(user_id);
CREATE INDEX idx_group_members_role ON group_members(role);
```

#### 7.1.3 group_posts 表（分析发布）

```sql
CREATE TABLE group_posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
  author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  -- 内容（结构化）
  title VARCHAR(200) NOT NULL,
  content TEXT NOT NULL, -- Markdown格式的详细分析
  
  -- 分析结构化数据
  analysis_data JSONB NOT NULL DEFAULT '{}'::jsonb,
  /* 
  {
    "asset": "BTC/USD",
    "signal": "bearish",
    "confidence": 0.72,
    "timeframe": "48h",
    "dimensions": {
      "on_chain": {"signal": "bearish", "confidence": 0.78, "summary": "..."},
      "macro": {"signal": "bearish", "confidence": 0.72, "summary": "..."},
      "technical": {"signal": "neutral", "confidence": 0.50, "summary": "..."},
      "sentiment": {"signal": "bearish", "confidence": 0.65, "summary": "..."}
    },
    "key_levels": {
      "support": [67200, 64800],
      "resistance": [69800]
    },
    "reasoning": "详细推理过程...",
    "chart_config": {
      "layers": ["price", "volume", "ai_annotations"],
      "timeframe": "1d",
      "range": "30d"
    }
  }
  */
  
  -- AI辅助标记
  ai_assisted BOOLEAN DEFAULT false, -- 是否由AI辅助生成
  ai_model VARCHAR(50), -- 使用的AI模型
  original_prompt TEXT, -- 用户的原始输入
  
  -- 附件
  attachments JSONB DEFAULT '[]'::jsonb, -- [{type: "image", url: "..."}, {type: "chart", config: {...}}]
  
  -- 互动统计
  upvotes INTEGER DEFAULT 0,
  downvotes INTEGER DEFAULT 0,
  comment_count INTEGER DEFAULT 0,
  view_count INTEGER DEFAULT 0,
  
  -- 状态
  is_published BOOLEAN DEFAULT true,
  is_pinned BOOLEAN DEFAULT false,
  
  -- 复盘数据（事后填充）
  outcome_data JSONB DEFAULT NULL,
  /*
  {
    "actual_result": "bearish", // 实际走势
    "prediction_accuracy": 0.85, // 准确率
    "evaluated_at": "2026-02-15T00:00:00Z",
    "price_change_pct": -5.2,
    "hit_support": true,
    "hit_resistance": false
  }
  */
  
  -- 时间
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  evaluated_at TIMESTAMP WITH TIME ZONE DEFAULT NULL -- 复盘时间（通常是发布后7天）
);

CREATE INDEX idx_group_posts_group ON group_posts(group_id);
CREATE INDEX idx_group_posts_author ON group_posts(author_id);
CREATE INDEX idx_group_posts_created ON group_posts(created_at DESC);
CREATE INDEX idx_group_posts_asset ON group_posts((analysis_data->>'asset'));
CREATE INDEX idx_group_posts_signal ON group_posts((analysis_data->>'signal'));
CREATE INDEX idx_group_posts_pinned ON group_posts(is_pinned) WHERE is_pinned = true;
```

#### 7.1.4 post_votes 表（投票）

```sql
CREATE TABLE post_votes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id UUID NOT NULL REFERENCES group_posts(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  
  -- 投票类型
  vote_type VARCHAR(20) NOT NULL, -- upvote, downvote, agree, disagree, insightful
  
  -- 投票附加数据
  vote_data JSONB DEFAULT '{}'::jsonb,
  /*
  {
    "own_signal": "bullish", // 用户自己的观点（可选）
    "confidence": 0.6,
    "reasoning": "我认为..."
  }
  */
  
  -- 时间
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  
  UNIQUE(post_id, user_id),
  CONSTRAINT valid_vote_type CHECK (vote_type IN ('upvote', 'downvote', 'agree', 'disagree', 'insightful'))
);

CREATE INDEX idx_post_votes_post ON post_votes(post_id);
CREATE INDEX idx_post_votes_user ON post_votes(user_id);
CREATE INDEX idx_post_votes_type ON post_votes(vote_type);
```

#### 7.1.5 post_comments 表（评论）

```sql
CREATE TABLE post_comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id UUID NOT NULL REFERENCES group_posts(id) ON DELETE CASCADE,
  author_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  parent_comment_id UUID REFERENCES post_comments(id) ON DELETE CASCADE, -- 支持嵌套回复
  
  -- 内容
  content TEXT NOT NULL,
  
  -- AI辅助
  ai_assisted BOOLEAN DEFAULT false,
  original_prompt TEXT,
  
  -- 结构化数据（可选）
  comment_data JSONB DEFAULT '{}'::jsonb,
  /*
  {
    "highlights_dimension": "on_chain", // 评论重点关注的维度
    "adds_data_point": {
      "metric": "exchange_inflow",
      "value": 15000,
      "source": "Glassnode"
    }
  }
  */
  
  -- 互动
  upvotes INTEGER DEFAULT 0,
  
  -- 时间
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_post_comments_post ON post_comments(post_id);
CREATE INDEX idx_post_comments_author ON post_comments(author_id);
CREATE INDEX idx_post_comments_parent ON post_comments(parent_comment_id);
CREATE INDEX idx_post_comments_created ON post_comments(created_at DESC);
```

#### 7.1.6 consensus_reports 表（共识报告）

```sql
CREATE TABLE consensus_reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
  
  -- 报告范围
  report_type VARCHAR(50) NOT NULL, -- daily, weekly, topic_based
  asset VARCHAR(20), -- 针对特定资产的报告（可选）
  topic VARCHAR(200), -- 针对特定主题的报告（可选）
  
  -- 时间范围
  period_start TIMESTAMP WITH TIME ZONE NOT NULL,
  period_end TIMESTAMP WITH TIME ZONE NOT NULL,
  
  -- 统计数据
  stats JSONB NOT NULL,
  /*
  {
    "total_posts": 12,
    "total_participants": 8,
    "total_votes": 45,
    
    "signal_distribution": {
      "bullish": 3,
      "bearish": 7,
      "neutral": 2
    },
    
    "avg_confidence": 0.68,
    
    "dimension_breakdown": {
      "on_chain": {"bullish": 2, "bearish": 5, "neutral": 1},
      "technical": {"bullish": 4, "bearish": 3, "neutral": 1},
      "macro": {"bullish": 1, "bearish": 8, "neutral": 0}
    },
    
    "most_cited_data": [
      {"metric": "exchange_inflow", "times_mentioned": 8},
      {"metric": "RSI", "times_mentioned": 6}
    ],
    
    "divergence_points": [
      {
        "dimension": "technical",
        "description": "技术面出现明显分歧",
        "bullish_ratio": 0.5,
        "bearish_ratio": 0.38
      }
    ],
    
    "key_levels_consensus": {
      "support": [67200, 64800],
      "resistance": [69800]
    }
  }
  */
  
  -- AI生成的摘要
  summary TEXT NOT NULL, -- Markdown格式的共识报告
  
  -- 与公域对比
  public_comparison JSONB DEFAULT NULL,
  /*
  {
    "public_signal": "bearish",
    "public_confidence": 0.72,
    "group_signal": "bearish",
    "group_confidence": 0.68,
    "agreement_level": "high",
    "unique_insights": ["小组更关注链上数据", "公域更重视宏观因素"]
  }
  */
  
  -- AI模型
  ai_model VARCHAR(50),
  
  -- 时间
  generated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_consensus_reports_group ON consensus_reports(group_id);
CREATE INDEX idx_consensus_reports_asset ON consensus_reports(asset);
CREATE INDEX idx_consensus_reports_type ON consensus_reports(report_type);
CREATE INDEX idx_consensus_reports_period ON consensus_reports(period_start, period_end);
```

---

### 7.2 API 设计

#### 7.2.1 创建小组

**POST /api/groups**

请求体：
```json
{
  "name": "加密短线技术派",
  "description": "专注于加密货币短线交易，技术分析为主",
  "group_type": "public",
  "max_members": 15,
  "tags": ["crypto", "short-term", "technical"],
  "focus_assets": ["BTC/USD", "ETH/USD", "SOL/USD"],
  "trading_style": "short-term",
  "analysis_preference": "technical"
}
```

响应：
```json
{
  "success": true,
  "data": {
    "id": "uuid-here",
    "name": "加密短线技术派",
    "creator_id": "user-uuid",
    "member_count": 1,
    "created_at": "2026-02-08T01:00:00Z",
    "invite_link": "https://finverse.app/groups/invite/abc123"
  }
}
```

逻辑实现：
```javascript
async function createGroup(userId, groupData) {
  // 1. 验证用户是否有权限创建小组（Pro版用户）
  const user = await db.users.findById(userId);
  if (user.plan !== 'pro' && user.plan !== 'team') {
    throw new Error('需要Pro版订阅才能创建小组');
  }
  
  // 2. 检查用户创建的小组数量限制
  const userGroupsCount = await db.groups.count({ creator_id: userId, is_active: true });
  const maxGroups = user.plan === 'pro' ? 3 : 10;
  if (userGroupsCount >= maxGroups) {
    throw new Error(`最多创建${maxGroups}个小组`);
  }
  
  // 3. 创建小组
  const group = await db.groups.create({
    ...groupData,
    creator_id: userId,
    member_count: 1
  });
  
  // 4. 自动添加创建者为成员（角色：creator）
  await db.group_members.create({
    group_id: group.id,
    user_id: userId,
    role: 'creator'
  });
  
  // 5. 生成邀请链接
  const inviteToken = generateInviteToken(group.id);
  
  // 6. 通知用户的Agent
  await notifyAgent(userId, {
    type: 'group_created',
    group_id: group.id,
    group_name: group.name
  });
  
  return { group, invite_link: `https://finverse.app/groups/invite/${inviteToken}` };
}
```

#### 7.2.2 AI辅助发布分析

**POST /api/groups/:groupId/posts/ai-assist**

请求体：
```json
{
  "user_prompt": "我觉得BTC要跌，交易所流入太多了",
  "include_chart": true,
  "auto_publish": false
}
```

AI处理流程：
```javascript
async function aiAssistedPost(groupId, userId, userPrompt, options) {
  // 1. 获取用户的AI配置
  const user = await db.users.findById(userId);
  const aiConfig = user.ai_config; // { provider: 'openai', api_key: '...', model: 'gpt-4' }
  
  // 2. 构建AI Prompt
  const systemPrompt = `你是金融分析助手，帮助用户将想法转化为结构化分析。

用户所在小组：${groupId}
小组标签：${group.tags.join(', ')}
用户偏好：${user.analysis_preference}

任务：
1. 识别用户提到的资产（如BTC/USD）
2. 识别用户的观点（看多/看空/中性）
3. 提取关键数据点（如"交易所流入"）
4. 补充相关数据支撑用户观点
5. 生成结构化分析JSON + Markdown详细分析

输出格式：
{
  "analysis_data": {
    "asset": "BTC/USD",
    "signal": "bearish",
    "confidence": 0.75,
    "dimensions": {...},
    "key_levels": {...},
    "reasoning": "..."
  },
  "title": "BTC短期看空：交易所流入显著增加",
  "content": "## 分析摘要\n\n...",
  "chart_config": {...}
}
`;

  // 3. 调用用户的AI
  const aiResponse = await callUserAI(aiConfig, systemPrompt, userPrompt);
  
  // 4. 解析AI响应
  const { analysis_data, title, content, chart_config } = JSON.parse(aiResponse);
  
  // 5. 获取实时数据补充
  if (analysis_data.dimensions.on_chain) {
    const onChainData = await fetchOnChainData(analysis_data.asset);
    analysis_data.dimensions.on_chain.data = onChainData;
  }
  
  // 6. 生成图表配置
  if (options.include_chart) {
    chart_config.data = await fetchChartData(analysis_data.asset, chart_config.timeframe);
  }
  
  // 7. 创建草稿或直接发布
  if (options.auto_publish) {
    const post = await db.group_posts.create({
      group_id: groupId,
      author_id: userId,
      title,
      content,
      analysis_data,
      ai_assisted: true,
      ai_model: aiConfig.model,
      original_prompt: userPrompt,
      attachments: options.include_chart ? [{ type: 'chart', config: chart_config }] : []
    });
    
    // 8. 通知小组成员
    await notifyGroupMembers(groupId, {
      type: 'new_post',
      post_id: post.id,
      author_id: userId,
      title
    });
    
    return post;
  } else {
    return { title, content, analysis_data, chart_config };
  }
}
```

AI Prompt模板（详细版）：
```
你是FinVerse平台的金融分析助手。用户会用自然语言描述他们的交易想法，你的任务是将其转化为结构化的分析报告。

## 用户信息
- 姓名：{user.name}
- 分析偏好：{user.analysis_preference}
- 交易风格：{user.trading_style}
- 所在小组：{group.name}
- 小组关注资产：{group.focus_assets}

## 用户输入
"{user_prompt}"

## 你的任务

### 1. 信息提取
- 识别资产（如BTC/USD, ETH/USD, AAPL等）
- 识别方向（看多bullish/看空bearish/中性neutral）
- 识别时间框架（短期/中期/长期，具体小时数）
- 提取关键数据点（如"交易所流入"、"RSI超买"等）

### 2. 数据补充
根据用户提到的数据点，补充最新的实际数据：
- 链上数据：交易所流入流出、大户持仓、MVRV等
- 技术指标：RSI、MACD、布林带、关键支撑阻力位
- 宏观因素：相关经济数据、政策事件
- 情绪指标：恐惧贪婪指数、社交媒体情绪

你可以调用以下工具获取数据：
- fetchOnChainMetrics(asset)
- fetchTechnicalIndicators(asset, timeframe)
- fetchMacroEvents(timeframe)
- fetchSentimentData(asset)

### 3. 置信度评估
根据以下因素综合评估置信度（0-1）：
- 数据支撑强度：多维度数据是否一致？
- 用户表述的确定性：用户是"觉得"还是"确定"？
- 历史准确率：类似情况下的历史成功率？
- 市场环境：当前波动率、流动性是否适合该判断？

### 4. 多维度分析
将分析拆解为4个维度，每个维度单独评估：

**on_chain（链上数据）**：
- signal: bullish/bearish/neutral
- confidence: 0-1
- summary: 一句话概括（30字以内）
- key_metrics: [{name: "交易所净流入", value: 12400, unit: "BTC", change_24h: "+35%"}]

**technical（技术分析）**：
- signal, confidence, summary
- key_levels: {support: [67200, 64800], resistance: [69800]}
- indicators: [{name: "RSI", value: 72, interpretation: "超买"}]

**macro（宏观因素）**：
- signal, confidence, summary
- events: [{event: "CPI数据公布", date: "2026-02-10", expected_impact: "bearish"}]

**sentiment（市场情绪）**：
- signal, confidence, summary
- metrics: [{name: "恐惧贪婪指数", value: 38, level: "恐惧"}]

### 5. 生成内容

**标题**（20字以内）：
- 格式：{资产} + {方向} + {核心理由}
- 例："BTC短期看空：交易所流入激增"

**详细分析**（Markdown格式）：
```markdown
## 分析摘要
{一段话总结，100字左右}

## 核心观点
- **方向**：{看多/看空/中性}
- **置信度**：{百分比}
- **时间框架**：{具体时间}
- **关键位**：支撑 {价格} / 阻力 {价格}

## 分维度分析

### 📊 链上数据（{信号} {置信度}）
{详细分析，150-200字}

**关键指标**：
- 交易所净流入：+12,400 BTC (+35%)
- 大户地址（>1000 BTC）：减持 3.2%
- MVRV比率：2.1（接近历史高位）

### 📈 技术面（{信号} {置信度}）
{详细分析}

**技术指标**：
- RSI(14)：72（超买区域）
- MACD：死叉信号
- 布林带：价格触及上轨

**关键位**：
- 支撑：$67,200 / $64,800
- 阻力：$69,800

### 🌍 宏观环境（{信号} {置信度}）
{详细分析}

**关注事件**：
- 2月10日 CPI数据公布（预期超标概率60%）
- 美联储3月会议预期维持利率

### 💭 市场情绪（{信号} {置信度}）
{详细分析}

**情绪指标**：
- 恐惧贪婪指数：38（恐惧）
- Twitter情绪分数：-0.25（偏空）

## 操作建议
{根据分析给出具体建议，但附加免责声明}

**免责声明**：以上分析仅为个人观点，不构成投资建议。
```

### 6. 输出JSON

最终输出严格的JSON格式：
```json
{
  "analysis_data": {
    "asset": "BTC/USD",
    "signal": "bearish",
    "confidence": 0.72,
    "timeframe": "48h",
    "dimensions": {
      "on_chain": {
        "signal": "bearish",
        "confidence": 0.78,
        "summary": "交易所净流入+12,400 BTC，抛压增加",
        "key_metrics": [...]
      },
      "technical": {...},
      "macro": {...},
      "sentiment": {...}
    },
    "key_levels": {
      "support": [67200, 64800],
      "resistance": [69800]
    },
    "reasoning": "综合链上数据显示交易所流入激增，结合技术面RSI超买和宏观CPI风险，短期看空概率较高。但技术支撑位尚在，需观察$67,200能否守住。"
  },
  "title": "BTC短期看空：交易所流入激增+技术超买",
  "content": "{上面的Markdown内容}",
  "chart_config": {
    "type": "candlestick",
    "asset": "BTC/USD",
    "timeframe": "1h",
    "range": "7d",
    "layers": ["price", "volume", "support_resistance", "ai_annotations"],
    "annotations": [
      {
        "type": "horizontal_line",
        "price": 67200,
        "label": "关键支撑",
        "color": "#22c55e"
      },
      {
        "type": "vertical_zone",
        "start_time": "2026-02-10T14:30:00Z",
        "end_time": "2026-02-10T15:00:00Z",
        "label": "CPI公布",
        "color": "rgba(239, 68, 68, 0.1)"
      }
    ]
  }
}
```

## 注意事项
- 如果用户输入信息不足，不要编造数据，而是标注为"待补充"
- 置信度评估要保守，不确定的事情不要给高置信度
- 技术指标的解读要准确（如RSI>70是超买而非超卖）
- 时间框架要具体（不要只说"短期"，要给出小时数或天数）
- 所有数据来源要可追溯（最好能链接到具体API或图表）
```

#### 7.2.3 投票API

**POST /api/posts/:postId/vote**

请求体：
```json
{
  "vote_type": "agree",
  "own_signal": "bearish",
  "confidence": 0.65,
  "reasoning": "我也看到了交易所流入数据，同意分析"
}
```

逻辑：
```javascript
async function voteOnPost(postId, userId, voteData) {
  // 1. 检查是否已投票（更新而非重复）
  const existing = await db.post_votes.findOne({ post_id: postId, user_id: userId });
  
  if (existing) {
    // 更新投票
    await db.post_votes.update(existing.id, {
      vote_type: voteData.vote_type,
      vote_data: {
        own_signal: voteData.own_signal,
        confidence: voteData.confidence,
        reasoning: voteData.reasoning
      },
      updated_at: new Date()
    });
  } else {
    // 新投票
    await db.post_votes.create({
      post_id: postId,
      user_id: userId,
      vote_type: voteData.vote_type,
      vote_data: {
        own_signal: voteData.own_signal,
        confidence: voteData.confidence,
        reasoning: voteData.reasoning
      }
    });
    
    // 更新帖子统计
    if (voteData.vote_type === 'upvote' || voteData.vote_type === 'agree') {
      await db.group_posts.increment(postId, 'upvotes', 1);
    } else if (voteData.vote_type === 'downvote' || voteData.vote_type === 'disagree') {
      await db.group_posts.increment(postId, 'downvotes', 1);
    }
  }
  
  // 2. 检查是否需要触发共识报告生成
  const post = await db.group_posts.findById(postId);
  const voteCount = await db.post_votes.count({ post_id: postId });
  const group = await db.groups.findById(post.group_id);
  
  // 如果超过50%成员投票，触发共识报告生成
  if (voteCount >= group.member_count * 0.5) {
    await scheduleConsensusReport(post.group_id, post.id);
  }
  
  return { success: true };
}
```

#### 7.2.4 生成共识报告

**POST /api/groups/:groupId/consensus-reports/generate**

请求体：
```json
{
  "report_type": "daily",
  "period_start": "2026-02-07T00:00:00Z",
  "period_end": "2026-02-08T00:00:00Z",
  "asset": "BTC/USD"
}
```

AI生成逻辑：
```javascript
async function generateConsensusReport(groupId, params) {
  // 1. 获取时间范围内的所有分析和投票
  const posts = await db.group_posts.find({
    group_id: groupId,
    created_at: { $gte: params.period_start, $lte: params.period_end },
    ...(params.asset && { 'analysis_data.asset': params.asset })
  });
  
  const postIds = posts.map(p => p.id);
  const votes = await db.post_votes.find({ post_id: { $in: postIds } });
  const comments = await db.post_comments.find({ post_id: { $in: postIds } });
  
  // 2. 统计分析
  const stats = {
    total_posts: posts.length,
    total_participants: new Set(posts.map(p => p.author_id)).size,
    total_votes: votes.length,
    
    signal_distribution: {
      bullish: posts.filter(p => p.analysis_data.signal === 'bullish').length,
      bearish: posts.filter(p => p.analysis_data.signal === 'bearish').length,
      neutral: posts.filter(p => p.analysis_data.signal === 'neutral').length
    },
    
    avg_confidence: posts.reduce((sum, p) => sum + p.analysis_data.confidence, 0) / posts.length,
    
    dimension_breakdown: {},
    most_cited_data: [],
    divergence_points: [],
    key_levels_consensus: {}
  };
  
  // 3. 维度分解统计
  ['on_chain', 'technical', 'macro', 'sentiment'].forEach(dim => {
    stats.dimension_breakdown[dim] = {
      bullish: posts.filter(p => p.analysis_data.dimensions[dim]?.signal === 'bullish').length,
      bearish: posts.filter(p => p.analysis_data.dimensions[dim]?.signal === 'bearish').length,
      neutral: posts.filter(p => p.analysis_data.dimensions[dim]?.signal === 'neutral').length
    };
  });
  
  // 4. 关键位共识（取众数）
  const allSupport = posts.flatMap(p => p.analysis_data.key_levels?.support || []);
  const allResistance = posts.flatMap(p => p.analysis_data.key_levels?.resistance || []);
  stats.key_levels_consensus = {
    support: findMostCommonLevels(allSupport, 2),
    resistance: findMostCommonLevels(allResistance, 2)
  };
  
  // 5. 找出分歧点
  Object.entries(stats.dimension_breakdown).forEach(([dim, signals]) => {
    const total = signals.bullish + signals.bearish + signals.neutral;
    const bullishRatio = signals.bullish / total;
    const bearishRatio = signals.bearish / total;
    
    // 如果看多看空比例都在30-70%之间，说明有分歧
    if (bullishRatio >= 0.3 && bullishRatio <= 0.7 && bearishRatio >= 0.3 && bearishRatio <= 0.7) {
      stats.divergence_points.push({
        dimension: dim,
        description: `${dim}维度出现明显分歧`,
        bullish_ratio: bullishRatio,
        bearish_ratio: bearishRatio
      });
    }
  });
  
  // 6. 获取公域数据对比
  const publicSignals = await fetchPublicSignals(params.asset, params.period_start, params.period_end);
  const publicStats = aggregatePublicSignals(publicSignals);
  
  stats.public_comparison = {
    public_signal: publicStats.signal,
    public_confidence: publicStats.confidence,
    group_signal: stats.signal_distribution.bearish > stats.signal_distribution.bullish ? 'bearish' : 'bullish',
    group_confidence: stats.avg_confidence,
    agreement_level: calculateAgreementLevel(publicStats, stats),
    unique_insights: findUniqueInsights(posts, publicSignals)
  };
  
  // 7. 调用AI生成摘要
  const aiPrompt = buildConsensusReportPrompt(posts, votes, comments, stats);
  const aiSummary = await callGroupCreatorAI(groupId, aiPrompt);
  
  // 8. 保存报告
  const report = await db.consensus_reports.create({
    group_id: groupId,
    report_type: params.report_type,
    asset: params.asset,
    period_start: params.period_start,
    period_end: params.period_end,
    stats,
    summary: aiSummary,
    public_comparison: stats.public_comparison,
    ai_model: 'gpt-4',
    generated_at: new Date()
  });
  
  // 9. 通知小组成员
  await notifyGroupMembers(groupId, {
    type: 'consensus_report_ready',
    report_id: report.id,
    asset: params.asset
  });
  
  return report;
}

function buildConsensusReportPrompt(posts, votes, comments, stats) {
  return `你是FinVerse平台的共识报告生成助手。根据小组成员的分析和讨论，生成一份共识报告。

## 时间范围
${stats.period_start} 至 ${stats.period_end}

## 数据汇总
- 总分析数：${stats.total_posts}
- 参与人数：${stats.total_participants}
- 总投票数：${stats.total_votes}

## 观点分布
- 看多：${stats.signal_distribution.bullish} (${(stats.signal_distribution.bullish/stats.total_posts*100).toFixed(0)}%)
- 看空：${stats.signal_distribution.bearish} (${(stats.signal_distribution.bearish/stats.total_posts*100).toFixed(0)}%)
- 中性：${stats.signal_distribution.neutral} (${(stats.signal_distribution.neutral/stats.total_posts*100).toFixed(0)}%)
- 平均置信度：${(stats.avg_confidence*100).toFixed(0)}%

## 维度分解
${Object.entries(stats.dimension_breakdown).map(([dim, signals]) => `
### ${dim}
- 看多：${signals.bullish}
- 看空：${signals.bearish}
- 中性：${signals.neutral}
`).join('\n')}

## 分歧点
${stats.divergence_points.map(d => `- ${d.description}：看多${(d.bullish_ratio*100).toFixed(0)}% vs 看空${(d.bearish_ratio*100).toFixed(0)}%`).join('\n')}

## 关键位共识
- 支撑：${stats.key_levels_consensus.support.join(', ')}
- 阻力：${stats.key_levels_consensus.resistance.join(', ')}

## 公域对比
- 公域观点：${stats.public_comparison.public_signal}（置信度${(stats.public_comparison.public_confidence*100).toFixed(0)}%）
- 小组观点：${stats.public_comparison.group_signal}（置信度${(stats.public_comparison.group_confidence*100).toFixed(0)}%）
- 一致性：${stats.public_comparison.agreement_level}

## 原始分析摘要
${posts.map(p => `
**${p.title}** (作者：${p.author_id})
- 观点：${p.analysis_data.signal}（置信度${(p.analysis_data.confidence*100).toFixed(0)}%）
- 核心理由：${p.analysis_data.reasoning.substring(0, 100)}...
`).join('\n')}

---

请生成一份Markdown格式的共识报告，包含：

# ${params.asset} 共识报告（${formatDate(params.period_start)} - ${formatDate(params.period_end)}）

## 📊 观点概览
{一句话总结小组共识，如"小组72%成员看空BTC短期走势"}

## 🎯 核心共识
- **方向**：{多数观点}
- **置信度**：{平均置信度}
- **关键位**：支撑 {价格} / 阻力 {价格}
- **时间框架**：{综合时间框架}

## 🔍 分维度分析

### 链上数据（{共识/分歧}）
{总结链上维度的共识或分歧点，150字左右}

### 技术面（{共识/分歧}）
{总结技术面的共识或分歧点}

### 宏观环境（{共识/分歧}）
{总结宏观维度的共识或分歧点}

### 市场情绪（{共识/分歧}）
{总结情绪维度的共识或分歧点}

## ⚡ 主要分歧点
{详细说明小组内的主要分歧在哪里，为什么会有分歧}

## 🌐 与公域对比
小组观点与公域Agent共识的对比：
- **一致性**：{高/中/低}
- **小组独特见解**：{列出小组关注但公域忽视的点}
- **公域独特见解**：{列出公域关注但小组忽视的点}

## 💡 关键洞察
{3-5条从讨论中提炼的关键洞察}

## ⚠️ 风险提示
{基于分歧点和数据，列出需要警惕的风险}

---
*本报告由AI自动生成，汇总小组成员分析。不构成投资建议。*
`;
}
```

---

### 7.3 前端组件设计

#### 7.3.1 小组主页（GroupHomePage.tsx）

**布局结构**：
```
┌─────────────────────────────────────────────────┐
│  Header: 小组名称 + 成员数 + 加入/退出按钮          │
├─────────────────────────────────────────────────┤
│  Tabs: 最新分析 | 共识报告 | 成员 | 设置           │
├─────────────────────────────────────────────────┤
│                                                  │
│  ┌────────────┐  ┌─────────────────────────┐   │
│  │  侧边栏     │  │   主内容区                │   │
│  │            │  │                          │   │
│  │  筛选器     │  │   [分析卡片列表]          │   │
│  │  - 资产     │  │                          │   │
│  │  - 信号     │  │   ┌──────────────────┐  │   │
│  │  - 时间     │  │   │  AnalysisCard    │  │   │
│  │            │  │   │  AnalysisCard    │  │   │
│  │  快捷操作   │  │   │  AnalysisCard    │  │   │
│  │  [发布分析] │  │   └──────────────────┘  │   │
│  │            │  │                          │   │
│  └────────────┘  └─────────────────────────┘   │
│                                                  │
└─────────────────────────────────────────────────┘
```

**组件代码框架**：
```tsx
// GroupHomePage.tsx
import React, { useState, useEffect } from 'react';
import { useParams } from 'react-router-dom';
import { GroupHeader } from './GroupHeader';
import { GroupSidebar } from './GroupSidebar';
import { AnalysisCard } from './AnalysisCard';
import { ConsensusReport } from './ConsensusReport';
import { api } from '@/lib/api';

export function GroupHomePage() {
  const { groupId } = useParams<{ groupId: string }>();
  const [group, setGroup] = useState(null);
  const [posts, setPosts] = useState([]);
  const [activeTab, setActiveTab] = useState('latest');
  const [filters, setFilters] = useState({
    asset: null,
    signal: null,
    timeRange: '7d'
  });

  useEffect(() => {
    loadGroup();
    loadPosts();
  }, [groupId, filters]);

  async function loadGroup() {
    const data = await api.get(`/groups/${groupId}`);
    setGroup(data);
  }

  async function loadPosts() {
    const data = await api.get(`/groups/${groupId}/posts`, { params: filters });
    setPosts(data);
  }

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Header */}
      <GroupHeader group={group} />

      {/* Tabs */}
      <div className="border-b border-gray-200 bg-white">
        <div className="max-w-7xl mx-auto px-4">
          <nav className="flex space-x-8">
            {['最新分析', '共识报告', '成员', '设置'].map(tab => (
              <button
                key={tab}
                onClick={() => setActiveTab(tab)}
                className={`py-4 px-1 border-b-2 font-medium text-sm ${
                  activeTab === tab
                    ? 'border-blue-500 text-blue-600'
                    : 'border-transparent text-gray-500 hover:text-gray-700'
                }`}
              >
                {tab}
              </button>
            ))}
          </nav>
        </div>
      </div>

      {/* Main Content */}
      <div className="max-w-7xl mx-auto px-4 py-6">
        <div className="grid grid-cols-12 gap-6">
          {/* Sidebar */}
          <div className="col-span-3">
            <GroupSidebar
              filters={filters}
              onFilterChange={setFilters}
              onNewPost={() => {/* 打开发布对话框 */}}
            />
          </div>

          {/* Main Area */}
          <div className="col-span-9">
            {activeTab === '最新分析' && (
              <div className="space-y-4">
                {posts.map(post => (
                  <AnalysisCard key={post.id} post={post} />
                ))}
              </div>
            )}

            {activeTab === '共识报告' && (
              <ConsensusReportList groupId={groupId} />
            )}
          </div>
        </div>
      </div>
    </div>
  );
}
```

#### 7.3.2 分析卡片（AnalysisCard.tsx）

**UI规格**：
- 宽度：100%（响应式）
- 背景：#FFFFFF
- 边框：1px solid #E5E7EB，圆角8px
- 内边距：24px
- 悬停效果：阴影 0 4px 6px rgba(0, 0, 0, 0.1)
- 动画：hover时上移2px，过渡0.2s

**布局**：
```
┌─────────────────────────────────────────────────┐
│  👤 作者名  |  📅 2小时前  |  🏷️ BTC/USD          │
├─────────────────────────────────────────────────┤
│  ## 标题（字号18px，字重600，颜色#111827）          │
│                                                  │
│  [信号指示器]  看空 ⬇️  置信度 72%                 │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━          │
│  链上 ⬇️ 78%  |  技术 → 50%  |  宏观 ⬇️ 72%  ...  │
│                                                  │
│  【分析摘要，2-3行，颜色#4B5563】                   │
│                                                  │
│  [图表预览缩略图，如果有]                          │
│                                                  │
├─────────────────────────────────────────────────┤
│  👍 12  👎 2  💬 5  👁️ 89  |  [查看详情]  [投票]   │
└─────────────────────────────────────────────────┘
```

**组件代码**：
```tsx
// AnalysisCard.tsx
import React from 'react';
import { TrendingDown, TrendingUp, Minus } from 'lucide-react';

interface AnalysisCardProps {
  post: {
    id: string;
    title: string;
    author: { name: string; avatar: string };
    created_at: string;
    analysis_data: {
      asset: string;
      signal: 'bullish' | 'bearish' | 'neutral';
      confidence: number;
      dimensions: Record<string, { signal: string; confidence: number }>;
      reasoning: string;
    };
    upvotes: number;
    downvotes: number;
    comment_count: number;
    view_count: number;
    attachments: any[];
  };
}

export function AnalysisCard({ post }: AnalysisCardProps) {
  const { analysis_data: data } = post;

  const signalConfig = {
    bullish: { icon: TrendingUp, color: '#22c55e', label: '看多', emoji: '⬆️' },
    bearish: { icon: TrendingDown, color: '#ef4444', label: '看空', emoji: '⬇️' },
    neutral: { icon: Minus, color: '#6b7280', label: '中性', emoji: '→' }
  };

  const config = signalConfig[data.signal];
  const SignalIcon = config.icon;

  return (
    <div className="bg-white rounded-lg border border-gray-200 p-6 hover:shadow-lg hover:-translate-y-0.5 transition-all duration-200 cursor-pointer">
      {/* Header */}
      <div className="flex items-center justify-between mb-3">
        <div className="flex items-center space-x-3">
          <img src={post.author.avatar} className="w-8 h-8 rounded-full" />
          <span className="font-medium text-sm text-gray-900">{post.author.name}</span>
          <span className="text-gray-400 text-sm">·</span>
          <span className="text-gray-500 text-sm">{formatTimeAgo(post.created_at)}</span>
        </div>
        <span className="px-3 py-1 bg-blue-50 text-blue-700 rounded-full text-sm font-medium">
          {data.asset}
        </span>
      </div>

      {/* Title */}
      <h3 className="text-lg font-semibold text-gray-900 mb-3">
        {post.title}
      </h3>

      {/* Signal Indicator */}
      <div className="mb-4">
        <div className="flex items-center space-x-4 mb-2">
          <div className="flex items-center space-x-2">
            <SignalIcon className="w-5 h-5" style={{ color: config.color }} />
            <span className="font-semibold" style={{ color: config.color }}>
              {config.label} {config.emoji}
            </span>
          </div>
          <div className="text-gray-600">
            置信度 <span className="font-semibold">{(data.confidence * 100).toFixed(0)}%</span>
          </div>
        </div>

        {/* Dimension Breakdown */}
        <div className="flex items-center space-x-4 text-sm">
          {Object.entries(data.dimensions).map(([dim, dimData]) => {
            const dimConfig = signalConfig[dimData.signal];
            return (
              <div key={dim} className="flex items-center space-x-1">
                <span className="text-gray-600">{dimNameMap[dim]}</span>
                <span style={{ color: dimConfig.color }}>
                  {dimConfig.emoji} {(dimData.confidence * 100).toFixed(0)}%
                </span>
              </div>
            );
          })}
        </div>
      </div>

      {/* Reasoning Preview */}
      <p className="text-gray-600 text-sm mb-4 line-clamp-2">
        {data.reasoning}
      </p>

      {/* Chart Preview */}
      {post.attachments.length > 0 && (
        <div className="mb-4 rounded-lg overflow-hidden">
          <img
            src={post.attachments[0].thumbnail_url}
            className="w-full h-32 object-cover"
          />
        </div>
      )}

      {/* Footer */}
      <div className="flex items-center justify-between pt-4 border-t border-gray-100">
        <div className="flex items-center space-x-4 text-sm text-gray-500">
          <span className="flex items-center space-x-1">
            <span>👍</span>
            <span>{post.upvotes}</span>
          </span>
          <span className="flex items-center space-x-1">
            <span>👎</span>
            <span>{post.downvotes}</span>
          </span>
          <span className="flex items-center space-x-1">
            <span>💬</span>
            <span>{post.comment_count}</span>
          </span>
          <span className="flex items-center space-x-1">
            <span>👁️</span>
            <span>{post.view_count}</span>
          </span>
        </div>

        <div className="flex items-center space-x-2">
          <button className="px-4 py-2 text-sm text-blue-600 hover:bg-blue-50 rounded-lg transition">
            查看详情
          </button>
          <button className="px-4 py-2 text-sm bg-blue-600 text-white hover:bg-blue-700 rounded-lg transition">
            投票
          </button>
        </div>
      </div>
    </div>
  );
}

const dimNameMap = {
  on_chain: '链上',
  technical: '技术',
  macro: '宏观',
  sentiment: '情绪'
};

function formatTimeAgo(timestamp: string): string {
  const now = Date.now();
  const then = new Date(timestamp).getTime();
  const diff = now - then;

  const minutes = Math.floor(diff / 60000);
  const hours = Math.floor(diff / 3600000);
  const days = Math.floor(diff / 86400000);

  if (minutes < 60) return `${minutes}分钟前`;
  if (hours < 24) return `${hours}小时前`;
  return `${days}天前`;
}
```

#### 7.3.3 投票组件（VoteModal.tsx）

**UI规格**：
- 模态框尺寸：600px x 500px
- 背景：#FFFFFF
- 圆角：12px
- 遮罩：rgba(0, 0, 0, 0.5)
- 动画：淡入 + 缩放，0.3s ease-out

**布局**：
```
┌─────────────────────────────────────────────────┐
│  投票：BTC短期看空分析                 [X]         │
├─────────────────────────────────────────────────┤
│                                                  │
│  你的观点：                                       │
│  ○ 同意（看空）  ○ 不同意（看多）  ○ 中性          │
│                                                  │
│  你的置信度：                                     │
│  ├─────────●───────┤  65%                       │
│                                                  │
│  你的理由（可选，AI会帮你结构化）：                 │
│  ┌────────────────────────────────────────────┐ │
│  │ 我也看到了交易所流入数据...                  │ │
│  │                                             │ │
│  └────────────────────────────────────────────┘ │
│                                                  │
│  [AI辅助扩展]  [直接提交]                         │
│                                                  │
└─────────────────────────────────────────────────┘
```

**组件代码**：
```tsx
// VoteModal.tsx
import React, { useState } from 'react';
import { X } from 'lucide-react';
import { api } from '@/lib/api';

interface VoteModalProps {
  post: any;
  onClose: () => void;
  onVoteSuccess: () => void;
}

export function VoteModal({ post, onClose, onVoteSuccess }: VoteModalProps) {
  const [voteType, setVoteType] = useState<'agree' | 'disagree' | 'neutral'>('agree');
  const [confidence, setConfidence] = useState(65);
  const [reasoning, setReasoning] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async () => {
    setIsSubmitting(true);
    try {
      await api.post(`/posts/${post.id}/vote`, {
        vote_type: voteType,
        own_signal: voteType === 'agree' ? post.analysis_data.signal : (voteType === 'disagree' ? oppositeSignal(post.analysis_data.signal) : 'neutral'),
        confidence: confidence / 100,
        reasoning
      });
      onVoteSuccess();
      onClose();
    } catch (error) {
      console.error('投票失败:', error);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center">
      {/* Overlay */}
      <div className="absolute inset-0 bg-black bg-opacity-50" onClick={onClose} />

      {/* Modal */}
      <div className="relative bg-white rounded-xl shadow-2xl w-full max-w-2xl p-6 animate-modal-in">
        {/* Header */}
        <div className="flex items-center justify-between mb-6">
          <h2 className="text-xl font-semibold text-gray-900">
            投票：{post.title}
          </h2>
          <button onClick={onClose} className="text-gray-400 hover:text-gray-600">
            <X className="w-6 h-6" />
          </button>
        </div>

        {/* Vote Type */}
        <div className="mb-6">
          <label className="block text-sm font-medium text-gray-700 mb-3">
            你的观点：
          </label>
          <div className="flex space-x-4">
            {[
              { value: 'agree', label: `同意（${post.analysis_data.signal === 'bullish' ? '看多' : '看空'}）`, color: 'blue' },
              { value: 'disagree', label: `不同意（${post.analysis_data.signal === 'bullish' ? '看空' : '看多'}）`, color: 'red' },
              { value: 'neutral', label: '中性', color: 'gray' }
            ].map(option => (
              <button
                key={option.value}
                onClick={() => setVoteType(option.value as any)}
                className={`flex-1 py-3 px-4 rounded-lg border-2 transition ${
                  voteType === option.value
                    ? `border-${option.color}-500 bg-${option.color}-50 text-${option.color}-700`
                    : 'border-gray-200 text-gray-600 hover:border-gray-300'
                }`}
              >
                {option.label}
              </button>
            ))}
          </div>
        </div>

        {/* Confidence Slider */}
        <div className="mb-6">
          <label className="block text-sm font-medium text-gray-700 mb-3">
            你的置信度：<span className="text-blue-600 font-semibold">{confidence}%</span>
          </label>
          <input
            type="range"
            min="0"
            max="100"
            value={confidence}
            onChange={(e) => setConfidence(Number(e.target.value))}
            className="w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer accent-blue-600"
          />
          <div className="flex justify-between text-xs text-gray-500 mt-1">
            <span>完全不确定</span>
            <span>非常确定</span>
          </div>
        </div>

        {/* Reasoning */}
        <div className="mb-6">
          <label className="block text-sm font-medium text-gray-700 mb-3">
            你的理由（可选）：
          </label>
          <textarea
            value={reasoning}
            onChange={(e) => setReasoning(e.target.value)}
            placeholder="简单说说你的想法，AI会帮你结构化..."
            className="w-full h-32 px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent resize-none"
          />
        </div>

        {/* Actions */}
        <div className="flex justify-end space-x-3">
          <button
            onClick={onClose}
            className="px-6 py-2 text-gray-700 hover:bg-gray-100 rounded-lg transition"
          >
            取消
          </button>
          <button
            onClick={handleSubmit}
            disabled={isSubmitting}
            className="px-6 py-2 bg-blue-600 text-white hover:bg-blue-700 rounded-lg transition disabled:opacity-50"
          >
            {isSubmitting ? '提交中...' : '提交投票'}
          </button>
        </div>
      </div>
    </div>
  );
}

function oppositeSignal(signal: string): string {
  return signal === 'bullish' ? 'bearish' : 'bullish';
}
```

#### 7.3.4 共识报告页（ConsensusReportPage.tsx）

**UI规格**：
- 整体背景：#F9FAFB
- 卡片背景：#FFFFFF
- 标题字号：28px，字重700，颜色#111827
- 副标题字号：16px，字重400，颜色#6B7280
- 间距：各section之间32px

**布局**：
```
┌─────────────────────────────────────────────────┐
│  BTC/USD 共识报告                                 │
│  2026年2月7日 - 2月8日                            │
├─────────────────────────────────────────────────┤
│                                                  │
│  📊 观点概览                                      │
│  ┌──────────────────────────────────────────┐  │
│  │  小组72%成员看空BTC短期走势                 │  │
│  │                                           │  │
│  │  [饼图] 看多 12% | 看空 72% | 中性 16%      │  │
│  └──────────────────────────────────────────┘  │
│                                                  │
│  🎯 核心共识                                      │
│  ┌──────────────────────────────────────────┐  │
│  │  方向：看空 ⬇️                              │  │
│  │  置信度：68%                                │  │
│  │  关键位：支撑 $67,200 / 阻力 $69,800       │  │
│  └──────────────────────────────────────────┘  │
│                                                  │
│  🔍 分维度分析                                    │
│  ┌──────────┬──────────┬──────────┬─────────┐  │
│  │ 链上     │ 技术     │ 宏观     │ 情绪    │  │
│  │ 共识⬇️   │ 分歧     │ 共识⬇️   │ 共识⬇️  │  │
│  │ 78%      │ 50%      │ 72%      │ 65%     │  │
│  └──────────┴──────────┴──────────┴─────────┘  │
│                                                  │
│  [详细分析内容...]                                │
│                                                  │
└─────────────────────────────────────────────────┘
```

---

### 7.4 历史复盘追踪算法

#### 7.4.1 复盘触发逻辑

```javascript
// retrospective-tracker.js

/**
 * 定期检查需要复盘的分析（Cron任务，每天运行）
 */
async function checkAnalysesForRetrospective() {
  const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  
  // 查找7天前发布且未复盘的分析
  const posts = await db.group_posts.find({
    created_at: { $lte: sevenDaysAgo },
    evaluated_at: null,
    is_published: true
  });
  
  for (const post of posts) {
    await evaluatePostAccuracy(post);
  }
}

/**
 * 评估单个分析的准确性
 */
async function evaluatePostAccuracy(post) {
  const { analysis_data } = post;
  const asset = analysis_data.asset;
  const timeframe = parseTimeframe(analysis_data.timeframe); // "48h" -> 48小时
  
  // 1. 获取实际价格数据
  const predictionTime = new Date(post.created_at);
  const evaluationTime = new Date(predictionTime.getTime() + timeframe * 60 * 60 * 1000);
  
  const priceAtPrediction = await fetchHistoricalPrice(asset, predictionTime);
  const priceAtEvaluation = await fetchHistoricalPrice(asset, evaluationTime);
  
  const priceChangePct = ((priceAtEvaluation - priceAtPrediction) / priceAtPrediction) * 100;
  
  // 2. 判断实际走势
  let actualSignal = 'neutral';
  if (priceChangePct > 2) actualSignal = 'bullish';
  else if (priceChangePct < -2) actualSignal = 'bearish';
  
  // 3. 计算准确率
  const predictionCorrect = actualSignal === analysis_data.signal;
  const accuracy = predictionCorrect ? 
    (1 - Math.abs(priceChangePct - getPredictedChange(analysis_data)) / 100) : 
    0;
  
  // 4. 检查关键位是否被触及
  const hitSupport = analysis_data.key_levels.support?.some(level => 
    checkIfPriceHitLevel(asset, predictionTime, evaluationTime, level, 'below')
  );
  
  const hitResistance = analysis_data.key_levels.resistance?.some(level => 
    checkIfPriceHitLevel(asset, predictionTime, evaluationTime, level, 'above')
  );
  
  // 5. 保存复盘结果
  const outcomeData = {
    actual_signal: actualSignal,
    prediction_accuracy: accuracy,
    evaluated_at: new Date().toISOString(),
    price_change_pct: priceChangePct,
    price_at_prediction: priceAtPrediction,
    price_at_evaluation: priceAtEvaluation,
    hit_support: hitSupport,
    hit_resistance: hitResistance,
    timeframe_hours: timeframe
  };
  
  await db.group_posts.update(post.id, {
    outcome_data: outcomeData,
    evaluated_at: new Date()
  });
  
  // 6. 更新用户的个人统计（只给用户自己看）
  await updateUserAccuracyStats(post.author_id, {
    asset,
    signal: analysis_data.signal,
    correct: predictionCorrect,
    confidence: analysis_data.confidence,
    accuracy
  });
  
  // 7. 通知用户（私密，不在小组公开）
  await notifyUser(post.author_id, {
    type: 'retrospective_ready',
    post_id: post.id,
    accuracy,
    prediction_correct: predictionCorrect
  });
}

/**
 * 检查价格是否触及某个关键位
 */
async function checkIfPriceHitLevel(asset, startTime, endTime, level, direction) {
  const ohlcData = await fetchOHLCData(asset, '1h', startTime, endTime);
  
  for (const candle of ohlcData) {
    if (direction === 'below' && candle.low <= level) return true;
    if (direction === 'above' && candle.high >= level) return true;
  }
  
  return false;
}

/**
 * 更新用户个人准确率统计
 */
async function updateUserAccuracyStats(userId, predictionData) {
  const stats = await db.user_stats.findOne({ user_id: userId }) || {
    user_id: userId,
    total_predictions: 0,
    correct_predictions: 0,
    by_asset: {},
    by_signal: {},
    by_dimension: {},
    confidence_calibration: []
  };
  
  // 总体统计
  stats.total_predictions += 1;
  if (predictionData.correct) stats.correct_predictions += 1;
  
  // 按资产统计
  if (!stats.by_asset[predictionData.asset]) {
    stats.by_asset[predictionData.asset] = { total: 0, correct: 0 };
  }
  stats.by_asset[predictionData.asset].total += 1;
  if (predictionData.correct) stats.by_asset[predictionData.asset].correct += 1;
  
  // 按信号类型统计
  if (!stats.by_signal[predictionData.signal]) {
    stats.by_signal[predictionData.signal] = { total: 0, correct: 0 };
  }
  stats.by_signal[predictionData.signal].total += 1;
  if (predictionData.correct) stats.by_signal[predictionData.signal].correct += 1;
  
  // 置信度校准（检查用户的置信度是否和实际准确率匹配）
  stats.confidence_calibration.push({
    confidence: predictionData.confidence,
    accuracy: predictionData.accuracy,
    timestamp: new Date()
  });
  
  // 只保留最近100条记录
  if (stats.confidence_calibration.length > 100) {
    stats.confidence_calibration = stats.confidence_calibration.slice(-100);
  }
  
  await db.user_stats.upsert({ user_id: userId }, stats);
}
```

#### 7.4.2 个人复盘仪表板（只给用户自己看）

**UI规格**：
- 背景：渐变 from-blue-50 to-purple-50
- 卡片：白色，圆角12px，阴影 0 4px 20px rgba(0,0,0,0.08)

**布局**：
```
┌─────────────────────────────────────────────────┐
│  🎯 我的分析复盘                                  │
├─────────────────────────────────────────────────┤
│                                                  │
│  总体准确率                                       │
│  ┌──────────────────────────────────────────┐  │
│  │                                           │  │
│  │      68%                                  │  │
│  │   ━━━━━━━━━━━                            │  │
│  │   23/34 次预测正确                         │  │
│  └──────────────────────────────────────────┘  │
│                                                  │
│  按资产分类                                       │
│  BTC/USD: 72% (18/25)  ████████░░              │
│  ETH/USD: 60% (3/5)    ██████░░░░              │
│  SOL/USD: 50% (2/4)    █████░░░░░              │
│                                                  │
│  按信号类型                                       │
│  看多: 65% (11/17)                               │
│  看空: 71% (12/17)                               │
│                                                  │
│  置信度校准                                       │
│  ┌──────────────────────────────────────────┐  │
│  │  [散点图]                                  │  │
│  │  X轴：你的置信度                            │  │
│  │  Y轴：实际准确率                            │  │
│  │  理想情况：散点在对角线上                    │  │
│  └──────────────────────────────────────────┘  │
│                                                  │
│  💡 改进建议                                     │
│  · 你在BTC分析上表现最好，建议继续深耕            │
│  · 你的置信度略高于实际准确率，建议更谨慎         │
│  · 看空判断比看多判断更准，可能你更擅长风险识别   │
│                                                  │
└─────────────────────────────────────────────────┘
```

**组件代码**：
```tsx
// MyRetrospectiveDashboard.tsx
import React, { useEffect, useState } from 'react';
import { ScatterChart, Scatter, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';
import { api } from '@/lib/api';

export function MyRetrospectiveDashboard() {
  const [stats, setStats] = useState(null);

  useEffect(() => {
    loadStats();
  }, []);

  async function loadStats() {
    const data = await api.get('/me/retrospective-stats');
    setStats(data);
  }

  if (!stats) return <div>加载中...</div>;

  const overallAccuracy = (stats.correct_predictions / stats.total_predictions) * 100;

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-purple-50 p-8">
      <div className="max-w-6xl mx-auto">
        <h1 className="text-3xl font-bold text-gray-900 mb-8">🎯 我的分析复盘</h1>

        {/* Overall Accuracy */}
        <div className="bg-white rounded-xl shadow-lg p-8 mb-6">
          <h2 className="text-xl font-semibold mb-4">总体准确率</h2>
          <div className="text-center">
            <div className="text-6xl font-bold text-blue-600 mb-2">
              {overallAccuracy.toFixed(0)}%
            </div>
            <div className="text-gray-600">
              {stats.correct_predictions} / {stats.total_predictions} 次预测正确
            </div>
            <div className="w-full bg-gray-200 rounded-full h-3 mt-4">
              <div
                className="bg-blue-600 h-3 rounded-full transition-all duration-1000"
                style={{ width: `${overallAccuracy}%` }}
              />
            </div>
          </div>
        </div>

        {/* By Asset */}
        <div className="bg-white rounded-xl shadow-lg p-8 mb-6">
          <h2 className="text-xl font-semibold mb-4">按资产分类</h2>
          {Object.entries(stats.by_asset).map(([asset, assetStats]) => {
            const accuracy = (assetStats.correct / assetStats.total) * 100;
            return (
              <div key={asset} className="mb-3">
                <div className="flex justify-between mb-1">
                  <span className="font-medium">{asset}</span>
                  <span className="text-gray-600">
                    {accuracy.toFixed(0)}% ({assetStats.correct}/{assetStats.total})
                  </span>
                </div>
                <div className="w-full bg-gray-200 rounded-full h-2">
                  <div
                    className={`h-2 rounded-full ${accuracy >= 70 ? 'bg-green-500' : accuracy >= 50 ? 'bg-yellow-500' : 'bg-red-500'}`}
                    style={{ width: `${accuracy}%` }}
                  />
                </div>
              </div>
            );
          })}
        </div>

        {/* By Signal Type */}
        <div className="bg-white rounded-xl shadow-lg p-8 mb-6">
          <h2 className="text-xl font-semibold mb-4">按信号类型</h2>
          <div className="grid grid-cols-2 gap-4">
            {Object.entries(stats.by_signal).map(([signal, signalStats]) => {
              const accuracy = (signalStats.correct / signalStats.total) * 100;
              return (
                <div key={signal} className="text-center p-4 bg-gray-50 rounded-lg">
                  <div className="text-2xl font-bold text-gray-900 mb-1">
                    {accuracy.toFixed(0)}%
                  </div>
                  <div className="text-gray-600">
                    {signal === 'bullish' ? '看多' : signal === 'bearish' ? '看空' : '中性'}
                  </div>
                  <div className="text-sm text-gray-500">
                    ({signalStats.correct}/{signalStats.total})
                  </div>
                </div>
              );
            })}
          </div>
        </div>

        {/* Confidence Calibration */}
        <div className="bg-white rounded-xl shadow-lg p-8">
          <h2 className="text-xl font-semibold mb-4">置信度校准</h2>
          <ResponsiveContainer width="100%" height={300}>
            <ScatterChart margin={{ top: 20, right: 20, bottom: 20, left: 20 }}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis
                type="number"
                dataKey="confidence"
                name="你的置信度"
                unit="%"
                domain={[0, 100]}
              />
              <YAxis
                type="number"
                dataKey="accuracy"
                name="实际准确率"
                unit="%"
                domain={[0, 100]}
              />
              <Tooltip cursor={{ strokeDasharray: '3 3' }} />
              <Scatter
                name="预测记录"
                data={stats.confidence_calibration.map(c => ({
                  confidence: c.confidence * 100,
                  accuracy: c.accuracy * 100
                }))}
                fill="#3b82f6"
              />
              {/* Ideal line */}
              <Scatter
                name="理想线"
                data={[{ confidence: 0, accuracy: 0 }, { confidence: 100, accuracy: 100 }]}
                fill="#22c55e"
                line
                lineType="fitting"
              />
            </ScatterChart>
          </ResponsiveContainer>
          <p className="text-sm text-gray-600 mt-4">
            💡 理想情况下，散点应该在绿色对角线上（你的置信度 = 实际准确率）
          </p>
        </div>

        {/* Insights */}
        <div className="bg-gradient-to-r from-blue-500 to-purple-600 rounded-xl shadow-lg p-8 mt-6 text-white">
          <h2 className="text-xl font-semibold mb-4">💡 AI 改进建议</h2>
          <ul className="space-y-2">
            {generateInsights(stats).map((insight, i) => (
              <li key={i} className="flex items-start">
                <span className="mr-2">·</span>
                <span>{insight}</span>
              </li>
            ))}
          </ul>
        </div>
      </div>
    </div>
  );
}

function generateInsights(stats) {
  const insights = [];
  
  // 最佳资产
  const bestAsset = Object.entries(stats.by_asset).reduce((best, [asset, assetStats]) => {
    const accuracy = assetStats.correct / assetStats.total;
    return (!best || accuracy > best.accuracy) ? { asset, accuracy } : best;
  }, null);
  
  if (bestAsset) {
    insights.push(`你在 ${bestAsset.asset} 分析上表现最好（${(bestAsset.accuracy*100).toFixed(0)}%），建议继续深耕`);
  }
  
  // 置信度校准
  const avgConfidence = stats.confidence_calibration.reduce((sum, c) => sum + c.confidence, 0) / stats.confidence_calibration.length;
  const avgAccuracy = stats.confidence_calibration.reduce((sum, c) => sum + c.accuracy, 0) / stats.confidence_calibration.length;
  
  if (avgConfidence > avgAccuracy + 0.1) {
    insights.push('你的置信度略高于实际准确率，建议更谨慎地评估确定性');
  } else if (avgAccuracy > avgConfidence + 0.1) {
    insights.push('你的实际表现比你的置信度更好，可以更自信一些');
  }
  
  // 信号类型对比
  const bullishAcc = stats.by_signal.bullish ? stats.by_signal.bullish.correct / stats.by_signal.bullish.total : 0;
  const bearishAcc = stats.by_signal.bearish ? stats.by_signal.bearish.correct / stats.by_signal.bearish.total : 0;
  
  if (bearishAcc > bullishAcc + 0.15) {
    insights.push('看空判断比看多判断更准，可能你更擅长风险识别');
  } else if (bullishAcc > bearishAcc + 0.15) {
    insights.push('看多判断比看空判断更准，可能你更擅长捕捉机会');
  }
  
  return insights;
}
```

---

## 八、聊天软件分工

### 8.1 Telegram Bot 消息模板

#### 8.1.1 每日摘要推送

**消息格式**：
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

**ETH/USD** $3,580 → +0.5%
🟡 中性信号（置信度 55%）
震荡整理中，等待方向选择

━━━━━━━━━━━━━━━━━━━━━━

🚨 **今日关注**
· 21:30 美国CPI数据公布
· BTC链上大额转账增加
· 小组"加密短线技术派"新分析 3 条

[查看详细看板] [调整偏好]
```

**Telegram Bot代码**：
```javascript
// telegram-bot.js
const TelegramBot = require('node-telegram-bot-api');
const { formatCurrency, formatPercentage } = require('./utils');

class FinVerseTelegramBot {
  constructor(token) {
    this.bot = new TelegramBot(token, { polling: true });
    this.setupHandlers();
  }

  setupHandlers() {
    // 启动命令
    this.bot.onText(/\/start/, this.handleStart.bind(this));
    
    // 自然语言查询
    this.bot.on('message', this.handleMessage.bind(this));
  }

  async sendDailySummary(userId, summaryData) {
    const { assets, alerts, groups } = summaryData;

    let message = `📋 **今日市场摘要** · ${formatDate(new Date())}\n\n`;
    message += '━━━━━━━━━━━━━━━━━━━━━━\n\n';

    // 主要资产
    for (const asset of assets) {
      message += this.formatAssetSummary(asset);
      message += '\n\n';
    }

    message += '━━━━━━━━━━━━━━━━━━━━━━\n\n';

    // 今日关注
    if (alerts.length > 0) {
      message += '🚨 **今日关注**\n';
      alerts.forEach(alert => {
        message += `· ${alert.description}\n`;
      });
      message += '\n';
    }

    // 小组动态
    if (groups.length > 0) {
      groups.forEach(group => {
        message += `· 小组"${group.name}"新分析 ${group.new_posts} 条\n`;
      });
      message += '\n';
    }

    // Inline keyboard
    const keyboard = {
      inline_keyboard: [
        [
          { text: '查看详细看板', url: `https://finverse.app/dashboard` },
          { text: '调整偏好', callback_data: 'settings' }
        ]
      ]
    };

    await this.bot.sendMessage(userId, message, {
      parse_mode: 'Markdown',
      reply_markup: keyboard
    });
  }

  formatAssetSummary(asset) {
    const { symbol, price, change_pct, signal, confidence, dimensions, key_levels } = asset;

    const signalEmoji = {
      bullish: '🟢',
      bearish: '🔴',
      neutral: '🟡'
    };

    const signalLabel = {
      bullish: '看多',
      bearish: '看空',
      neutral: '中性'
    };

    const signalIcon = {
      bullish: '⬆️',
      bearish: '⬇️',
      neutral: '→'
    };

    let msg = `**${symbol}** ${formatCurrency(price)} ${signalIcon[signal]} ${formatPercentage(change_pct)}\n`;
    msg += `${signalEmoji[signal]} ${signalLabel[signal]}信号（置信度 ${Math.round(confidence * 100)}%）\n`;

    // 维度分解
    Object.entries(dimensions).forEach(([dim, data], index) => {
      const prefix = index === Object.keys(dimensions).length - 1 ? '└' : '├';
      msg += `${prefix} ${dimNameMap[dim]}：${data.summary} ${signalIcon[data.signal]}\n`;
    });

    // 关键位
    if (key_levels.support.length > 0 || key_levels.resistance.length > 0) {
      msg += `\n关键位：支撑 ${key_levels.support.map(formatCurrency).join(', ')} | 阻力 ${key_levels.resistance.map(formatCurrency).join(', ')}`;
    }

    return msg;
  }

  async sendAlert(userId, alertData) {
    const { type, asset, message: alertMessage, severity, url } = alertData;

    const severityEmoji = {
      high: '🚨',
      medium: '⚠️',
      low: 'ℹ️'
    };

    let message = `${severityEmoji[severity]} **${asset} 异常预警**\n\n`;
    message += alertMessage;

    const keyboard = url ? {
      inline_keyboard: [[
        { text: '查看详情', url }
      ]]
    } : null;

    await this.bot.sendMessage(userId, message, {
      parse_mode: 'Markdown',
      reply_markup: keyboard
    });
  }

  async sendGroupNotification(userId, notification) {
    const { type, group_name, post_title, author_name, url } = notification;

    let message = '';

    if (type === 'new_post') {
      message = `📊 小组"**${group_name}**"新分析\n\n`;
      message += `**${post_title}**\n`;
      message += `作者：${author_name}`;
    } else if (type === 'consensus_report_ready') {
      message = `📋 小组"**${group_name}**"共识报告已生成`;
    } else if (type === 'new_comment') {
      message = `💬 ${author_name} 评论了你的分析\n\n`;
      message += `"${post_title}"`;
    }

    const keyboard = {
      inline_keyboard: [[
        { text: '查看', url }
      ]]
    };

    await this.bot.sendMessage(userId, message, {
      parse_mode: 'Markdown',
      reply_markup: keyboard
    });
  }

  async handleMessage(msg) {
    const userId = msg.from.id;
    const text = msg.text;

    // 跳过命令
    if (text.startsWith('/')) return;

    // 发送"正在思考"指示器
    await this.bot.sendChatAction(userId, 'typing');

    // 调用用户的Agent处理自然语言查询
    try {
      const response = await this.callUserAgent(userId, text);

      if (response.type === 'text') {
        await this.bot.sendMessage(userId, response.content, {
          parse_mode: 'Markdown'
        });
      } else if (response.type === 'chart') {
        // 发送图表（先发图，再发分析）
        await this.bot.sendPhoto(userId, response.chart_url);
        await this.bot.sendMessage(userId, response.content, {
          parse_mode: 'Markdown'
        });
      } else if (response.type === 'redirect') {
        // 复杂分析，引导用户去网站查看
        const keyboard = {
          inline_keyboard: [[
            { text: '在网站查看详细分析', url: response.url }
          ]]
        };
        await this.bot.sendMessage(userId, response.content, {
          parse_mode: 'Markdown',
          reply_markup: keyboard
        });
      }
    } catch (error) {
      console.error('处理消息失败:', error);
      await this.bot.sendMessage(userId, '抱歉，处理你的请求时出错了。请稍后再试。');
    }
  }

  async callUserAgent(userId, query) {
    // 调用FinVerse API，触发用户的Agent
    const response = await fetch(`${process.env.FINVERSE_API}/agent/query`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ user_id: userId, query })
    });

    return await response.json();
  }
}

const dimNameMap = {
  on_chain: '链上',
  technical: '技术',
  macro: '宏观',
  sentiment: '情绪'
};
```

#### 8.1.2 Inline 图片卡片格式

当Agent生成图表时，在Telegram中以图片+说明的形式发送：

**图片生成**（使用Canvas或Puppeteer）：
```javascript
// chart-image-generator.js
const { createCanvas } = require('canvas');

async function generateChartImage(chartData) {
  const width = 800;
  const height = 600;
  const canvas = createCanvas(width, height);
  const ctx = canvas.getContext('2d');

  // 背景
  ctx.fillStyle = '#FFFFFF';
  ctx.fillRect(0, 0, width, height);

  // 标题
  ctx.fillStyle = '#111827';
  ctx.font = 'bold 24px Arial';
  ctx.fillText(chartData.title, 20, 40);

  // 副标题
  ctx.fillStyle = '#6B7280';
  ctx.font = '16px Arial';
  ctx.fillText(chartData.subtitle, 20, 70);

  // 绘制K线图（简化版）
  const candles = chartData.candles;
  const chartHeight = 400;
  const chartTop = 100;
  const chartWidth = width - 40;
  const candleWidth = chartWidth / candles.length;

  const prices = candles.flatMap(c => [c.high, c.low]);
  const maxPrice = Math.max(...prices);
  const minPrice = Math.min(...prices);
  const priceRange = maxPrice - minPrice;

  candles.forEach((candle, i) => {
    const x = 20 + i * candleWidth;
    const openY = chartTop + ((maxPrice - candle.open) / priceRange) * chartHeight;
    const closeY = chartTop + ((maxPrice - candle.close) / priceRange) * chartHeight;
    const highY = chartTop + ((maxPrice - candle.high) / priceRange) * chartHeight;
    const lowY = chartTop + ((maxPrice - candle.low) / priceRange) * chartHeight;

    // 蜡烛颜色
    ctx.fillStyle = candle.close >= candle.open ? '#22c55e' : '#ef4444';
    ctx.strokeStyle = ctx.fillStyle;

    // 影线
    ctx.beginPath();
    ctx.moveTo(x + candleWidth / 2, highY);
    ctx.lineTo(x + candleWidth / 2, lowY);
    ctx.stroke();

    // 实体
    const bodyTop = Math.min(openY, closeY);
    const bodyHeight = Math.abs(closeY - openY) || 1;
    ctx.fillRect(x + 2, bodyTop, candleWidth - 4, bodyHeight);
  });

  // AI标注（如果有）
  if (chartData.annotations) {
    chartData.annotations.forEach(annotation => {
      if (annotation.type === 'horizontal_line') {
        const y = chartTop + ((maxPrice - annotation.price) / priceRange) * chartHeight;
        ctx.strokeStyle = annotation.color;
        ctx.lineWidth = 2;
        ctx.setLineDash([5, 5]);
        ctx.beginPath();
        ctx.moveTo(20, y);
        ctx.lineTo(width - 20, y);
        ctx.stroke();
        ctx.setLineDash([]);

        // 标签
        ctx.fillStyle = annotation.color;
        ctx.font = 'bold 14px Arial';
        ctx.fillText(`${annotation.label} $${annotation.price}`, width - 180, y - 5);
      }
    });
  }

  // 底部信息
  ctx.fillStyle = '#6B7280';
  ctx.font = '12px Arial';
  ctx.fillText('FinVerse · AI驱动的金融分析平台', 20, height - 20);

  return canvas.toBuffer('image/png');
}
```

**发送带图表的消息**：
```javascript
async function sendChartAnalysis(userId, analysis) {
  // 1. 生成图表图片
  const chartImage = await generateChartImage(analysis.chart_data);

  // 2. 发送图片
  await bot.sendPhoto(userId, chartImage, {
    caption: `📊 ${analysis.asset} 分析图表`
  });

  // 3. 发送文字分析
  let message = `**${analysis.title}**\n\n`;
  message += `${analysis.summary}\n\n`;
  message += `[查看完整分析](${analysis.url})`;

  await bot.sendMessage(userId, message, {
    parse_mode: 'Markdown',
    disable_web_page_preview: true
  });
}
```

### 8.2 推送触发逻辑

#### 8.2.1 触发条件矩阵

| 事件 | 触发条件 | 推送时机 | 内容 |
|------|---------|---------|------|
| 每日摘要 | 用户订阅 | 用户起床时间（学习得出）| 市场总览+关注资产 |
| 异常预警 | 监控脚本检测到异常 | 立即 | 具体异常+AI分析 |
| 开盘提醒 | 用户关注的市场开盘 | 开盘前30分钟 | 开盘前景展望 |
| 收盘复盘 | 市场收盘 | 收盘后1小时 | 当日回顾+明日展望 |
| 小组新分析 | 小组成员发布 | 立即（可设置免打扰） | 分析摘要+链接 |
| 共识报告 | 报告生成完成 | 立即 | 报告摘要+链接 |
| 评论/@ | 有人评论或@用户 | 立即 | 评论内容+链接 |
| 价格穿越关键位 | 价格触及用户关注的位 | 立即 | 穿越提醒+当前分析 |

#### 8.2.2 通知优先级和频率控制

```javascript
// notification-manager.js

class NotificationManager {
  constructor() {
    this.priorityLevels = {
      critical: 1,   // 异常预警、价格穿越
      high: 2,       // 小组@、评论回复
      medium: 3,     // 小组新分析、共识报告
      low: 4         // 每日摘要、市场开闭盘
    };

    this.cooldowns = new Map(); // 用户ID -> 最后推送时间
  }

  async shouldSendNotification(userId, notification) {
    const { priority, type } = notification;

    // 1. 检查用户通知设置
    const userSettings = await db.user_settings.findOne({ user_id: userId });
    if (userSettings.notifications_disabled) return false;

    // 2. 检查免打扰时段
    const userTimezone = userSettings.timezone;
    const currentHour = new Date().toLocaleString('en-US', {
      timeZone: userTimezone,
      hour: 'numeric',
      hour12: false
    });

    if (currentHour >= 23 || currentHour < 8) {
      // 深夜/凌晨，只发critical级别
      if (priority !== 'critical') return false;
    }

    // 3. 检查频率限制（防止刷屏）
    const lastSent = this.cooldowns.get(userId);
    if (lastSent) {
      const timeSinceLastMinutes = (Date.now() - lastSent) / 1000 / 60;

      if (priority === 'critical') {
        // Critical可以连续发，但至少间隔2分钟
        if (timeSinceLastMinutes < 2) return false;
      } else if (priority === 'high') {
        // High至少间隔5分钟
        if (timeSinceLastMinutes < 5) return false;
      } else {
        // Medium/Low至少间隔30分钟
        if (timeSinceLastMinutes < 30) return false;
      }
    }

    // 4. 检查用户对此类通知的订阅
    if (userSettings.notification_types[type] === false) return false;

    return true;
  }

  async sendNotification(userId, notification) {
    if (!await this.shouldSendNotification(userId, notification)) {
      console.log(`跳过通知: ${userId} - ${notification.type}`);
      return;
    }

    // 发送通知
    const user = await db.users.findById(userId);
    const channel = user.preferred_channel; // telegram, whatsapp, discord, etc.

    if (channel === 'telegram') {
      await this.sendTelegramNotification(user.telegram_id, notification);
    } else if (channel === 'whatsapp') {
      await this.sendWhatsAppNotification(user.whatsapp_number, notification);
    }
    // ... 其他渠道

    // 更新cooldown
    this.cooldowns.set(userId, Date.now());

    // 记录日志
    await db.notification_logs.create({
      user_id: userId,
      notification_type: notification.type,
      priority: notification.priority,
      sent_at: new Date()
    });
  }

  async sendTelegramNotification(telegramId, notification) {
    switch (notification.type) {
      case 'daily_summary':
        await telegramBot.sendDailySummary(telegramId, notification.data);
        break;
      case 'alert':
        await telegramBot.sendAlert(telegramId, notification.data);
        break;
      case 'group_new_post':
      case 'group_consensus_report':
      case 'group_comment':
        await telegramBot.sendGroupNotification(telegramId, notification.data);
        break;
      default:
        console.warn(`未知通知类型: ${notification.type}`);
    }
  }
}
```

### 8.3 链接生成（跳转到网站）

所有从聊天软件推送的消息，需要深度查看时都应该附带链接，跳转到网站对应页面。

**链接格式**：
```
https://finverse.app/dashboard
https://finverse.app/groups/{groupId}
https://finverse.app/groups/{groupId}/posts/{postId}
https://finverse.app/groups/{groupId}/consensus-reports/{reportId}
https://finverse.app/assets/{asset}
https://finverse.app/retrospective
```

**链接生成工具**：
```javascript
// url-builder.js

class URLBuilder {
  constructor(baseURL = 'https://finverse.app') {
    this.baseURL = baseURL;
  }

  dashboard(userId) {
    return `${this.baseURL}/dashboard?user=${userId}`;
  }

  group(groupId) {
    return `${this.baseURL}/groups/${groupId}`;
  }

  groupPost(groupId, postId) {
    return `${this.baseURL}/groups/${groupId}/posts/${postId}`;
  }

  consensusReport(groupId, reportId) {
    return `${this.baseURL}/groups/${groupId}/consensus-reports/${reportId}`;
  }

  asset(asset) {
    return `${this.baseURL}/assets/${encodeURIComponent(asset)}`;
  }

  retrospective(userId) {
    return `${this.baseURL}/retrospective?user=${userId}`;
  }

  // 带认证token的链接（用于从聊天软件直接登录）
  withAuth(url, userId) {
    const token = generateAuthToken(userId, '1h');
    const separator = url.includes('?') ? '&' : '?';
    return `${url}${separator}auth_token=${token}`;
  }
}

function generateAuthToken(userId, expiresIn) {
  const jwt = require('jsonwebtoken');
  return jwt.sign({ user_id: userId }, process.env.JWT_SECRET, {
    expiresIn
  });
}
```

**使用示例**：
```javascript
const urlBuilder = new URLBuilder();

// 小组新分析通知
const notification = {
  type: 'group_new_post',
  data: {
    group_name: '加密短线技术派',
    post_title: 'BTC短期看空：交易所流入激增',
    author_name: 'Alice',
    url: urlBuilder.withAuth(
      urlBuilder.groupPost(groupId, postId),
      userId
    )
  }
};
```

---

## 九、可视化——三种呈现模式

### 9.1 摘要模式（Summary Mode）

**设计理念**：1秒看完，知道该关注什么。极简、信息密度低、关键信息突出。

#### 9.1.1 整体布局

```
┌─────────────────────────────────────────────────┐
│  Header: 摘要模式 | 图表模式 | 数据模式             │
├─────────────────────────────────────────────────┤
│                                                  │
│  [BTC/USD]  [ETH/USD]  [SOL/USD]  [+ 添加资产]   │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━     │
│                                                  │
│  ╔══════════════════════════════════════════╗  │
│  ║  BTC/USD                                  ║  │
│  ║                                            ║  │
│  ║  $67,450  ⬇️ -2.3%                         ║  │
│  ║  看空 🔴 置信度 72%                         ║  │
│  ║                                            ║  │
│  ║  交易所流入激增，技术面超买，关注支撑$67,200  ║  │
│  ║                                            ║  │
│  ║  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━      ║  │
│  ║  链上  ████████░░ 78% ⬇️                   ║  │
│  ║  技术  █████░░░░░ 50% →                    ║  │
│  ║  宏观  ███████░░░ 72% ⬇️                   ║  │
│  ║  情绪  ██████░░░░ 65% ⬇️                   ║  │
│  ╚══════════════════════════════════════════╝  │
│                                                  │
│  🚨 异常预警 (2)                                 │
│  ┌────────────────────────────────────────────┐│
│  │ ⚠️  BTC 链上大额转账  15 分钟前             ││
│  │ ℹ️  ETH Gas费飙升     1 小时前              ││
│  └────────────────────────────────────────────┘│
│                                                  │
│  📊 小组共识                                     │
│  ┌────────────────────────────────────────────┐│
│  │ "加密短线技术派"  72%看空  [查看报告]        ││
│  └────────────────────────────────────────────┘│
│                                                  │
└─────────────────────────────────────────────────┘
```

#### 9.1.2 UI规格详细参数

**资产卡片（AssetSummaryCard）**：
- 外框：2px solid，根据信号颜色（看多#22c55e / 看空#ef4444 / 中性#6b7280）
- 背景：渐变，from-white to-{signal-color}-50/10
- 圆角：16px
- 内边距：32px
- 宽度：100%（最小800px）
- 阴影：0 8px 32px rgba(0, 0, 0, 0.12)
- 悬停效果：阴影增强 0 12px 48px rgba(0, 0, 0, 0.18)，过渡0.3s

**价格显示**：
- 字号：48px
- 字重：700
- 颜色：#111827
- 行高：1.2

**涨跌幅**：
- 字号：24px
- 字重：600
- 颜色：涨#22c55e / 跌#ef4444
- 包含emoji箭头（⬆️ / ⬇️）

**信号标签**：
- 背景：信号颜色，50%透明度
- 文字：信号颜色，加粗
- 内边距：12px 24px
- 圆角：24px（胶囊形）
- 字号：18px
- 字重：600

**一句话结论**：
- 字号：18px
- 字重：400
- 颜色：#4B5563
- 行高：1.6
- 最大2行，超出省略

**多维信号条**：
- 高度：每条 32px
- 间距：12px
- 背景条：#E5E7EB
- 填充条：渐变，from-{signal-color}-400 to-{signal-color}-600
- 圆角：8px
- 文字：左侧维度名（14px，#6B7280），右侧百分比+emoji（14px，信号颜色，bold）
- 动画：填充条加载时从0宽度animate到目标宽度，1s ease-out

**异常预警卡片**：
- 背景：根据严重度（high: #FEF2F2 / medium: #FEF3C7 / low: #F0F9FF）
- 边框：左侧4px solid，严重度颜色（high: #EF4444 / medium: #F59E0B / low: #3B82F6）
- 内边距：16px 20px
- 圆角：8px
- emoji大小：20px
- 文字：14px，#374151
- 时间：12px，#9CA3AF，右上角

**小组共识条**：
- 背景：#F3F4F6
- 内边距：20px
- 圆角：12px
- 小组名：16px，#111827，bold
- 共识文字：16px，#4B5563
- 按钮：蓝色链接，14px

**组件代码**：
```tsx
// SummaryMode.tsx
import React from 'react';
import { TrendingUp, TrendingDown, AlertTriangle, Info } from 'lucide-react';

interface SummaryModeProps {
  assets: Array<{
    symbol: string;
    price: number;
    change_pct: number;
    signal: 'bullish' | 'bearish' | 'neutral';
    confidence: number;
    summary: string;
    dimensions: Record<string, { signal: string; confidence: number; summary: string }>;
  }>;
  alerts: Array<{
    severity: 'high' | 'medium' | 'low';
    message: string;
    timestamp: string;
  }>;
  groupConsensus: Array<{
    group_name: string;
    signal: string;
    percentage: number;
    report_url: string;
  }>;
}

export function SummaryMode({ assets, alerts, groupConsensus }: SummaryModeProps) {
  return (
    <div className="min-h-screen bg-gray-50 p-8">
      {/* Asset Tabs */}
      <div className="mb-8 flex space-x-4 border-b-2 border-gray-200">
        {assets.map(asset => (
          <button
            key={asset.symbol}
            className="pb-4 px-4 font-semibold text-gray-700 hover:text-blue-600 border-b-2 border-transparent hover:border-blue-600 transition"
          >
            {asset.symbol}
          </button>
        ))}
        <button className="pb-4 px-4 text-gray-400 hover:text-blue-600">
          + 添加资产
        </button>
      </div>

      {/* Main Asset Card */}
      {assets.map(asset => (
        <AssetSummaryCard key={asset.symbol} asset={asset} />
      ))}

      {/* Alerts */}
      {alerts.length > 0 && (
        <div className="mt-8">
          <h2 className="text-xl font-semibold text-gray-900 mb-4">
            🚨 异常预警 ({alerts.length})
          </h2>
          <div className="space-y-3">
            {alerts.map((alert, i) => (
              <AlertCard key={i} alert={alert} />
            ))}
          </div>
        </div>
      )}

      {/* Group Consensus */}
      {groupConsensus.length > 0 && (
        <div className="mt-8">
          <h2 className="text-xl font-semibold text-gray-900 mb-4">
            📊 小组共识
          </h2>
          <div className="space-y-3">
            {groupConsensus.map((consensus, i) => (
              <GroupConsensusBar key={i} consensus={consensus} />
            ))}
          </div>
        </div>
      )}
    </div>
  );
}

function AssetSummaryCard({ asset }) {
  const signalConfig = {
    bullish: { color: '#22c55e', bg: 'from-white to-green-50', border: 'border-green-500', emoji: '⬆️' },
    bearish: { color: '#ef4444', bg: 'from-white to-red-50', border: 'border-red-500', emoji: '⬇️' },
    neutral: { color: '#6b7280', bg: 'from-white to-gray-50', border: 'border-gray-500', emoji: '→' }
  };

  const config = signalConfig[asset.signal];

  return (
    <div
      className={`bg-gradient-to-br ${config.bg} border-2 ${config.border} rounded-2xl p-8 shadow-xl hover:shadow-2xl transition-all duration-300 mb-6`}
    >
      {/* Header: Symbol */}
      <h1 className="text-2xl font-bold text-gray-900 mb-4">{asset.symbol}</h1>

      {/* Price and Change */}
      <div className="flex items-baseline space-x-4 mb-4">
        <div className="text-5xl font-bold text-gray-900">
          ${asset.price.toLocaleString()}
        </div>
        <div
          className="text-2xl font-semibold flex items-center"
          style={{ color: asset.change_pct >= 0 ? '#22c55e' : '#ef4444' }}
        >
          {asset.change_pct >= 0 ? <TrendingUp className="w-6 h-6 mr-1" /> : <TrendingDown className="w-6 h-6 mr-1" />}
          {asset.change_pct >= 0 ? '+' : ''}{asset.change_pct.toFixed(2)}%
        </div>
      </div>

      {/* Signal Badge */}
      <div className="mb-6">
        <span
          className="inline-block px-6 py-3 rounded-full font-semibold text-lg"
          style={{ backgroundColor: `${config.color}20`, color: config.color }}
        >
          {asset.signal === 'bullish' ? '看多' : asset.signal === 'bearish' ? '看空' : '中性'} {config.emoji} 置信度 {Math.round(asset.confidence * 100)}%
        </span>
      </div>

      {/* Summary */}
      <p className="text-lg text-gray-600 mb-6 leading-relaxed">
        {asset.summary}
      </p>

      {/* Dimension Bars */}
      <div className="space-y-3">
        {Object.entries(asset.dimensions).map(([dim, data]) => (
          <DimensionBar key={dim} dimension={dim} data={data} />
        ))}
      </div>
    </div>
  );
}

function DimensionBar({ dimension, data }) {
  const dimNames = {
    on_chain: '链上',
    technical: '技术',
    macro: '宏观',
    sentiment: '情绪'
  };

  const signalConfig = {
    bullish: { color: '#22c55e', emoji: '⬆️' },
    bearish: { color: '#ef4444', emoji: '⬇️' },
    neutral: { color: '#6b7280', emoji: '→' }
  };

  const config = signalConfig[data.signal];
  const percentage = Math.round(data.confidence * 100);

  return (
    <div>
      <div className="flex justify-between mb-1">
        <span className="text-sm font-medium text-gray-700">{dimNames[dimension]}</span>
        <span className="text-sm font-bold" style={{ color: config.color }}>
          {percentage}% {config.emoji}
        </span>
      </div>
      <div className="w-full bg-gray-200 rounded-lg h-8 overflow-hidden">
        <div
          className="h-8 rounded-lg transition-all duration-1000 ease-out"
          style={{
            width: `${percentage}%`,
            background: `linear-gradient(to right, ${config.color}, ${config.color}dd)`
          }}
        />
      </div>
    </div>
  );
}

function AlertCard({ alert }) {
  const severityConfig = {
    high: { bg: '#FEF2F2', border: '#EF4444', icon: AlertTriangle, text: '#DC2626' },
    medium: { bg: '#FEF3C7', border: '#F59E0B', icon: Info, text: '#D97706' },
    low: { bg: '#F0F9FF', border: '#3B82F6', icon: Info, text: '#2563EB' }
  };

  const config = severityConfig[alert.severity];
  const Icon = config.icon;

  return (
    <div
      className="rounded-lg p-4 flex items-start space-x-3"
      style={{ backgroundColor: config.bg, borderLeft: `4px solid ${config.border}` }}
    >
      <Icon className="w-5 h-5 flex-shrink-0 mt-0.5" style={{ color: config.text }} />
      <div className="flex-1">
        <p className="text-sm text-gray-800">{alert.message}</p>
      </div>
      <span className="text-xs text-gray-400">{formatTimeAgo(alert.timestamp)}</span>
    </div>
  );
}

function GroupConsensusBar({ consensus }) {
  return (
    <div className="bg-gray-100 rounded-xl p-5 flex items-center justify-between">
      <div>
        <span className="font-semibold text-gray-900">"{consensus.group_name}"</span>
        <span className="ml-3 text-gray-600">
          {consensus.percentage}%{consensus.signal === 'bullish' ? '看多' : '看空'}
        </span>
      </div>
      <a
        href={consensus.report_url}
        className="text-blue-600 hover:text-blue-700 font-medium text-sm"
      >
        查看报告 →
      </a>
    </div>
  );
}

function formatTimeAgo(timestamp) {
  const now = Date.now();
  const then = new Date(timestamp).getTime();
  const diffMinutes = Math.floor((now - then) / 60000);
  
  if (diffMinutes < 60) return `${diffMinutes} 分钟前`;
  const diffHours = Math.floor(diffMinutes / 60);
  if (diffHours < 24) return `${diffHours} 小时前`;
  const diffDays = Math.floor(diffHours / 24);
  return `${diffDays} 天前`;
}
```

---

### 9.2 图表模式（Chart Mode）

**设计理念**：技术派的最爱。多图层叠加，AI标注，推理链可视化。

#### 9.2.1 整体布局

```
┌─────────────────────────────────────────────────┐
│  [BTC/USD] 1h | 4h | 1d | 1w     $67,450 ⬇️-2.3% │
├─────────────────────────────────────────────────┤
│                                                  │
│  ┌──────────────────┐  ┌─────────────────────┐ │
│  │  图层控制          │  │   主图表区            │ │
│  │                   │  │                      │ │
│  │  ☑ 价格           │  │   [K线图 + 图层]      │ │
│  │  ☑ 成交量         │  │                      │ │
│  │  ☑ AI标注         │  │   [可交互]           │ │
│  │  ☑ 支撑阻力       │  │                      │ │
│  │  ☑ 宏观事件       │  │                      │ │
│  │  ☑ 链上数据       │  │                      │ │
│  │  ☐ 异常热力图     │  │                      │ │
│  │  ☐ 历史叠影       │  │                      │ │
│  │                   │  │                      │ │
│  │  [图层样式设置]   │  └─────────────────────┘ │
│  └──────────────────┘                           │
│                                                  │
│  AI推理链（点击标注点展开）                       │
│  ┌────────────────────────────────────────────┐ │
│  │  ① 链上数据异常 → 权重 30%                  │ │
│  │     交易所流入 +12,400 BTC                  │ │
│  │  ② 宏观环境利空 → 权重 35%                  │ │
│  │     CPI超预期概率 60%                        │ │
│  │  ③ 技术面支撑尚在 → 权重 20%                │ │
│  │     $67,200 支撑未破                         │ │
│  │  ④ 历史形态相似度 → 权重 15%                │ │
│  │     与 2024-08 走势相似度 78%                │ │
│  │                                             │ │
│  │  综合评分：偏空 72%                          │ │
│  └────────────────────────────────────────────┘ │
│                                                  │
└─────────────────────────────────────────────────┘
```

#### 9.2.2 图层系统实现（Canvas/WebGL）

使用 **lightweight-charts** 库作为基础（高性能，支持大数据量）：

```bash
npm install lightweight-charts
```

**组件架构**：
```tsx
// ChartMode.tsx
import React, { useRef, useEffect, useState } from 'react';
import { createChart, CrosshairMode } from 'lightweight-charts';

export function ChartMode({ asset, timeframe }) {
  const chartContainerRef = useRef(null);
  const [chart, setChart] = useState(null);
  const [layers, setLayers] = useState({
    price: true,
    volume: true,
    ai_annotations: true,
    support_resistance: true,
    macro_events: true,
    on_chain: true,
    heatmap: false,
    historical_overlay: false
  });

  useEffect(() => {
    if (!chartContainerRef.current) return;

    // 创建图表
    const newChart = createChart(chartContainerRef.current, {
      width: chartContainerRef.current.clientWidth,
      height: 600,
      layout: {
        background: { color: '#FFFFFF' },
        textColor: '#333',
      },
      grid: {
        vertLines: { color: '#F0F0F0' },
        horzLines: { color: '#F0F0F0' },
      },
      crosshair: {
        mode: CrosshairMode.Normal,
      },
      timeScale: {
        borderColor: '#D1D4DC',
        timeVisible: true,
        secondsVisible: false,
      },
      priceScale: {
        borderColor: '#D1D4DC',
      },
    });

    setChart(newChart);

    // 响应式
    const handleResize = () => {
      newChart.applyOptions({
        width: chartContainerRef.current.clientWidth,
      });
    };
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      newChart.remove();
    };
  }, []);

  useEffect(() => {
    if (!chart) return;

    // 加载数据并绘制图层
    loadChartData(asset, timeframe).then(data => {
      renderLayers(chart, data, layers);
    });
  }, [chart, asset, timeframe, layers]);

  return (
    <div className="flex h-screen bg-gray-50">
      {/* Sidebar: Layer Controls */}
      <div className="w-64 bg-white border-r border-gray-200 p-4">
        <h2 className="text-lg font-semibold mb-4">图层控制</h2>
        <div className="space-y-2">
          {Object.entries(layers).map(([key, enabled]) => (
            <label key={key} className="flex items-center space-x-2 cursor-pointer">
              <input
                type="checkbox"
                checked={enabled}
                onChange={() => setLayers(prev => ({ ...prev, [key]: !enabled }))}
                className="w-4 h-4 text-blue-600 rounded"
              />
              <span className="text-sm text-gray-700">{layerNames[key]}</span>
            </label>
          ))}
        </div>
      </div>

      {/* Main Chart */}
      <div className="flex-1 p-6">
        <div ref={chartContainerRef} className="w-full" />
        <AIReasoningChain asset={asset} />
      </div>
    </div>
  );
}

const layerNames = {
  price: '价格',
  volume: '成交量',
  ai_annotations: 'AI标注',
  support_resistance: '支撑阻力',
  macro_events: '宏观事件',