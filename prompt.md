# FinVerse 完整开发提示词

你是一个全栈开发者，现在要构建 **FinVerse 平台** —— 一个 AI 时代的金融协作平台。用户自带 AI API key 和数据源,平台提供 Agent 运行环境、可视化界面和协作社区。

以下是完整的开发指令,按模块分阶段执行。每个阶段都可以独立完成并测试。

---

## 阶段 0: 项目基础设施搭建

### 0.1 初始化项目结构

创建以下目录结构:

```
finverse/
├── frontend/              # Next.js 前端
│   ├── app/              # App Router
│   ├── components/       # React 组件
│   ├── lib/              # 工具函数
│   ├── styles/           # 全局样式
│   └── public/           # 静态资源
├── backend/              # FastAPI 后端
│   ├── api/              # API 路由
│   ├── models/           # 数据模型
│   ├── services/         # 业务逻辑
│   ├── orchestrator/     # Agent 调度器
│   └── monitoring/       # 监控脚本
├── database/             # 数据库脚本
│   ├── migrations/       # 迁移脚本
│   └── schemas/          # Schema 定义
└── docker/               # Docker 配置
    ├── agent/            # Agent 容器
    ├── backend/          # 后端容器
    └── frontend/         # 前端容器
```

### 0.2 环境配置文件

创建 `.env.example`:

```env
# 数据库
DATABASE_URL=postgresql://user:password@localhost:5432/finverse
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=your-secret-key-here
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30

# CORS
ALLOWED_ORIGINS=http://localhost:3000,https://finverse.app

# OpenClaw
OPENCLAW_BINARY_PATH=/usr/local/bin/openclaw
OPENCLAW_WORKSPACE_BASE=/var/lib/finverse/agents

# 数据源 (平台官方 Agent 使用)
COINGECKO_API_KEY=optional
YAHOO_FINANCE_API_KEY=optional

# 消息通道 (平台通知)
TELEGRAM_BOT_TOKEN=your-bot-token
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=noreply@finverse.app
SMTP_PASSWORD=your-password

# 监控
SENTRY_DSN=optional
LOG_LEVEL=info
```

### 0.3 Docker Compose 配置

创建 `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: finverse
      POSTGRES_USER: finverse
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  backend:
    build: ./backend
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
    volumes:
      - ./backend:/app
      - agent_workspaces:/var/lib/finverse/agents
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis

  frontend:
    build: ./frontend
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8000
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"
    depends_on:
      - backend

  monitoring:
    build: ./backend
    command: python -m monitoring.daemon
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
    volumes:
      - ./backend:/app
    depends_on:
      - postgres
      - redis

volumes:
  postgres_data:
  redis_data:
  agent_workspaces:
```

---

## 阶段 1: 数据库设计与 API 基础

### 1.1 数据库 Schema 设计

创建 `database/schemas/core.sql`:

```sql
-- 用户表
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    
    -- 基本信息
    display_name VARCHAR(100),
    avatar_url TEXT,
    timezone VARCHAR(50) DEFAULT 'UTC',
    language VARCHAR(10) DEFAULT 'en',
    
    -- 订阅状态
    subscription_tier VARCHAR(20) DEFAULT 'free', -- free, pro, team
    subscription_expires_at TIMESTAMP,
    
    -- 元数据
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active_at TIMESTAMP,
    is_active BOOLEAN DEFAULT true,
    
    -- 索引
    INDEX idx_email (email),
    INDEX idx_username (username),
    INDEX idx_subscription (subscription_tier, subscription_expires_at)
);

-- API Key 管理表 (用户的外部 API keys)
CREATE TABLE user_api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    
    -- Key 信息
    service_type VARCHAR(50) NOT NULL, -- openai, anthropic, deepseek, coingecko, etc.
    encrypted_key TEXT NOT NULL, -- AES-256 加密
    key_name VARCHAR(100), -- 用户自定义名称
    
    -- 元数据
    is_active BOOLEAN DEFAULT true,
    last_validated_at TIMESTAMP,
    last_error TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 索引
    INDEX idx_user_service (user_id, service_type),
    UNIQUE(user_id, service_type, key_name)
);

-- 用户偏好与标签
CREATE TABLE user_preferences (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    
    -- 偏好数据 (JSONB)
    trading_styles JSONB DEFAULT '[]'::jsonb, -- ["day_trading", "swing"]
    analysis_preferences JSONB DEFAULT '[]'::jsonb, -- ["technical", "on_chain"]
    asset_preferences JSONB DEFAULT '[]'::jsonb, -- ["crypto", "forex"]
    risk_profile VARCHAR(20), -- conservative, balanced, aggressive
    
    -- 可视化偏好
    default_view_mode VARCHAR(20) DEFAULT 'summary', -- summary, chart, data
    chart_settings JSONB DEFAULT '{}'::jsonb,
    
    -- 通知偏好
    notification_channels JSONB DEFAULT '[]'::jsonb, -- ["telegram", "email"]
    notification_frequency VARCHAR(20) DEFAULT 'realtime', -- realtime, digest, quiet
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    
    -- Agent 行为偏好
    agent_personality JSONB DEFAULT '{}'::jsonb,
    agent_custom_prompts TEXT,
    
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Agent 实例表
CREATE TABLE agent_instances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    
    -- 容器信息
    container_id VARCHAR(100) UNIQUE,
    container_status VARCHAR(20) DEFAULT 'creating', -- creating, running, paused, stopped, error
    
    -- OpenClaw 配置
    workspace_path TEXT NOT NULL,
    openclaw_version VARCHAR(20),
    
    -- 资源状态
    resource_tier VARCHAR(20) DEFAULT 'active', -- active, standby, hibernated
    last_heartbeat_at TIMESTAMP,
    cpu_usage_percent DECIMAL(5,2),
    memory_usage_mb INTEGER,
    
    -- 元数据
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 索引
    INDEX idx_user (user_id),
    INDEX idx_container (container_id),
    INDEX idx_status (container_status)
);

-- 聊天通道连接表
CREATE TABLE chat_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    
    -- 通道信息
    platform VARCHAR(20) NOT NULL, -- telegram, whatsapp, discord, signal
    platform_user_id VARCHAR(255) NOT NULL, -- 平台侧的用户 ID
    platform_username VARCHAR(100),
    
    -- 连接状态
    is_active BOOLEAN DEFAULT true,
    is_primary BOOLEAN DEFAULT false, -- 主要通道
    
    -- 认证信息
    access_token TEXT,
    refresh_token TEXT,
    token_expires_at TIMESTAMP,
    
    -- 元数据
    connected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_message_at TIMESTAMP,
    
    -- 索引
    UNIQUE(platform, platform_user_id),
    INDEX idx_user_platform (user_id, platform)
);

-- 公域信号表
CREATE TABLE public_signals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID REFERENCES agent_instances(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    
    -- 信号内容
    asset VARCHAR(50) NOT NULL, -- BTC/USD, AAPL, EUR/USD, etc.
    signal VARCHAR(20) NOT NULL, -- bullish, bearish, neutral
    confidence DECIMAL(4,3) NOT NULL CHECK (confidence >= 0 AND confidence <= 1),
    
    -- 多维分析
    dimensions JSONB NOT NULL, -- {on_chain: {signal, confidence, summary}, ...}
    key_levels JSONB, -- {support: [67200, 64800], resistance: [69800]}
    reasoning TEXT,
    
    -- 时间信息
    timeframe VARCHAR(20), -- 1h, 4h, 1d, 1w
    valid_until TIMESTAMP,
    
    -- 信誉相关
    reputation_score DECIMAL(4,3) DEFAULT 0.5,
    view_count INTEGER DEFAULT 0,
    reference_count INTEGER DEFAULT 0, -- 被其他 Agent 引用次数
    
    -- 元数据
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 索引
    INDEX idx_asset_time (asset, created_at DESC),
    INDEX idx_agent_time (agent_id, created_at DESC),
    INDEX idx_signal_confidence (asset, signal, confidence DESC)
);

-- 信号订阅表
CREATE TABLE signal_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    
    -- 订阅条件
    assets JSONB DEFAULT '[]'::jsonb, -- ["BTC/USD", "ETH/USD"] 或 [] 表示全部
    min_confidence DECIMAL(4,3) DEFAULT 0.6,
    signals JSONB DEFAULT '[]'::jsonb, -- ["bullish", "bearish"] 或 [] 表示全部
    dimensions JSONB DEFAULT '[]'::jsonb, -- ["on_chain", "macro"] 或 [] 表示全部
    
    -- 订阅选项
    is_active BOOLEAN DEFAULT true,
    notification_enabled BOOLEAN DEFAULT true,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user (user_id)
);

-- 小组表
CREATE TABLE groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- 基本信息
    name VARCHAR(100) NOT NULL,
    description TEXT,
    avatar_url TEXT,
    
    -- 小组特性
    focus_assets JSONB DEFAULT '[]'::jsonb, -- 关注的品种
    focus_styles JSONB DEFAULT '[]'::jsonb, -- 交易风格
    
    -- 权限设置
    is_public BOOLEAN DEFAULT false, -- 是否公开可见
    join_approval_required BOOLEAN DEFAULT true, -- 是否需要审批
    max_members INTEGER DEFAULT 50,
    
    -- 统计
    member_count INTEGER DEFAULT 0,
    analysis_count INTEGER DEFAULT 0,
    
    -- 元数据
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_public (is_public, created_at DESC),
    INDEX idx_focus (focus_assets, focus_styles)
);

-- 小组成员表
CREATE TABLE group_members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id UUID REFERENCES groups(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    
    -- 角色
    role VARCHAR(20) DEFAULT 'member', -- owner, admin, member
    
    -- 状态
    status VARCHAR(20) DEFAULT 'pending', -- pending, active, left
    
    -- 贡献统计
    analysis_count INTEGER DEFAULT 0,
    comment_count INTEGER DEFAULT 0,
    reputation_in_group DECIMAL(4,3) DEFAULT 0.5,
    
    -- 元数据
    joined_at TIMESTAMP,
    left_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(group_id, user_id),
    INDEX idx_group_status (group_id, status),
    INDEX idx_user_groups (user_id, status)
);

-- 小组分析表
CREATE TABLE group_analyses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id UUID REFERENCES groups(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    
    -- 分析内容
    title VARCHAR(200) NOT NULL,
    asset VARCHAR(50) NOT NULL,
    signal VARCHAR(20) NOT NULL, -- bullish, bearish, neutral
    confidence DECIMAL(4,3),
    
    -- 详细内容
    content_markdown TEXT, -- AI 生成的结构化内容
    data_snapshot JSONB, -- 分析时刻的数据快照
    charts JSONB DEFAULT '[]'::jsonb, -- 图表配置和数据
    
    -- 讨论统计
    view_count INTEGER DEFAULT 0,
    upvote_count INTEGER DEFAULT 0,
    downvote_count INTEGER DEFAULT 0,
    comment_count INTEGER DEFAULT 0,
    
    -- 元数据
    published_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_pinned BOOLEAN DEFAULT false,
    
    INDEX idx_group_time (group_id, published_at DESC),
    INDEX idx_asset (asset, published_at DESC)
);

-- 分析评论表
CREATE TABLE analysis_comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id UUID REFERENCES group_analyses(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    parent_comment_id UUID REFERENCES analysis_comments(id) ON DELETE CASCADE,
    
    -- 评论内容
    content TEXT NOT NULL,
    
    -- 统计
    upvote_count INTEGER DEFAULT 0,
    
    -- 元数据
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_deleted BOOLEAN DEFAULT false,
    
    INDEX idx_analysis_time (analysis_id, created_at),
    INDEX idx_parent (parent_comment_id)
);

-- 分析投票表
CREATE TABLE analysis_votes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id UUID REFERENCES group_analyses(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    
    vote_type VARCHAR(10) NOT NULL, -- upvote, downvote
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(analysis_id, user_id),
    INDEX idx_analysis (analysis_id)
);

-- 共识报告表
CREATE TABLE consensus_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id UUID REFERENCES groups(id) ON DELETE CASCADE,
    
    -- 报告范围
    report_type VARCHAR(20) NOT NULL, -- daily, weekly, topic
    topic_asset VARCHAR(50), -- 如果是 topic 类型
    time_range_start TIMESTAMP NOT NULL,
    time_range_end TIMESTAMP NOT NULL,
    
    -- 报告内容
    summary TEXT, -- AI 生成的摘要
    consensus_signal VARCHAR(20), -- 整体共识
    consensus_confidence DECIMAL(4,3),
    
    -- 统计数据
    total_analyses INTEGER,
    bullish_count INTEGER,
    bearish_count INTEGER,
    neutral_count INTEGER,
    
    -- 分歧分析
    divergence_dimensions JSONB, -- 哪些维度分歧大
    key_debates JSONB, -- 主要争议点
    
    -- 与公域对比
    public_consensus_comparison JSONB,
    
    -- 元数据
    generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_group_time (group_id, time_range_end DESC),
    INDEX idx_type_asset (report_type, topic_asset, generated_at DESC)
);

-- 历史复盘表
CREATE TABLE historical_reviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_id UUID REFERENCES group_analyses(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    
    -- 原始预测
    original_signal VARCHAR(20),
    original_confidence DECIMAL(4,3),
    predicted_at TIMESTAMP,
    
    -- 实际结果
    actual_outcome VARCHAR(20), -- correct, incorrect, partial
    price_at_prediction DECIMAL(20,8),
    price_at_review DECIMAL(20,8),
    price_change_percent DECIMAL(8,4),
    
    -- 分析
    accuracy_score DECIMAL(4,3), -- 0-1
    what_went_right TEXT, -- AI 生成
    what_went_wrong TEXT, -- AI 生成
    lessons_learned TEXT, -- AI 生成
    
    -- 元数据
    reviewed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    review_timeframe_days INTEGER, -- 预测后多久复盘
    
    INDEX idx_user_time (user_id, reviewed_at DESC),
    INDEX idx_analysis (analysis_id)
);

-- 用户信誉表 (聚合统计)
CREATE TABLE user_reputation (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    
    -- 公域信号信誉
    public_signal_count INTEGER DEFAULT 0,
    public_signal_accuracy DECIMAL(4,3) DEFAULT 0.5,
    public_reputation_score DECIMAL(4,3) DEFAULT 0.5,
    
    -- 小组贡献信誉
    total_analyses INTEGER DEFAULT 0,
    total_comments INTEGER DEFAULT 0,
    total_upvotes INTEGER DEFAULT 0,
    
    -- 历史准确率 (按时间段)
    accuracy_7d DECIMAL(4,3),
    accuracy_30d DECIMAL(4,3),
    accuracy_90d DECIMAL(4,3),
    
    -- 按维度准确率
    accuracy_by_dimension JSONB DEFAULT '{}'::jsonb,
    
    -- 按品种准确率
    accuracy_by_asset JSONB DEFAULT '{}'::jsonb,
    
    -- 更新时间
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_reputation (public_reputation_score DESC)
);

-- 异常事件表 (监控脚本产出)
CREATE TABLE anomaly_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- 异常信息
    asset VARCHAR(50) NOT NULL,
    anomaly_type VARCHAR(50) NOT NULL, -- price_spike, volume_spike, on_chain_flow, etc.
    severity VARCHAR(20) NOT NULL, -- low, medium, high, critical
    
    -- 数据
    data_snapshot JSONB NOT NULL, -- 异常时刻的完整数据
    description TEXT,
    
    -- 通知状态
    notified_agents INTEGER DEFAULT 0, -- 已通知的 Agent 数量
    
    -- 元数据
    detected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP,
    
    INDEX idx_asset_severity_time (asset, severity, detected_at DESC),
    INDEX idx_unresolved (resolved_at) WHERE resolved_at IS NULL
);

-- 数据源配置表 (用户的数据源)
CREATE TABLE data_source_configs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    
    -- 数据源信息
    source_type VARCHAR(50) NOT NULL, -- glassnode, polygon, twelve_data, etc.
    encrypted_api_key TEXT, -- 如果需要 key
    
    -- 配置
    is_enabled BOOLEAN DEFAULT true,
    config_options JSONB DEFAULT '{}'::jsonb,
    
    -- 状态
    last_sync_at TIMESTAMP,
    last_error TEXT,
    
    -- 元数据
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(user_id, source_type),
    INDEX idx_user (user_id)
);

-- 创建更新时间自动触发器函数
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- 为所有有 updated_at 的表添加触发器
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_user_api_keys_updated_at BEFORE UPDATE ON user_api_keys FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_user_preferences_updated_at BEFORE UPDATE ON user_preferences FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_agent_instances_updated_at BEFORE UPDATE ON agent_instances FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_public_signals_updated_at BEFORE UPDATE ON public_signals FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_signal_subscriptions_updated_at BEFORE UPDATE ON signal_subscriptions FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_groups_updated_at BEFORE UPDATE ON groups FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_group_analyses_updated_at BEFORE UPDATE ON group_analyses FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_analysis_comments_updated_at BEFORE UPDATE ON analysis_comments FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_data_source_configs_updated_at BEFORE UPDATE ON data_source_configs FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 1.2 Redis 数据结构设计

Redis 用于缓存、实时数据和 WebSocket 通信。

**缓存 Keys:**
```
# 用户会话
session:{user_id}  -> {user_data_json}  (TTL: 1 hour)

# 市场数据缓存
market:price:{asset}  -> {price, volume, timestamp}  (TTL: 10 seconds)
market:signals:{asset}  -> [signal_ids]  (TTL: 5 minutes)

# Agent 状态
agent:{agent_id}:status  -> {status, last_heartbeat}  (TTL: 1 minute)
agent:{agent_id}:tasks  -> [task_queue]  (List)

# 实时通知队列
notifications:{user_id}  -> [notification_objects]  (List, max 100)

# 信号聚合缓存
consensus:{asset}:{timeframe}  -> {bullish_pct, bearish_pct, ...}  (TTL: 5 minutes)

# Rate limiting
ratelimit:{user_id}:{endpoint}  -> count  (TTL: 1 minute)

# WebSocket 会话
ws:session:{connection_id}  -> {user_id, agent_id, channels}  (TTL: 1 hour)

# Pub/Sub 频道
channel:signals  -> 新信号发布
channel:anomalies  -> 异常事件
channel:group:{group_id}  -> 小组更新
```

### 1.3 后端 API 结构

创建 `backend/main.py`:

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import uvicorn

from api import auth, users, agents, signals, groups, market
from database import engine
from config import settings

app = FastAPI(
    title="FinVerse API",
    description="AI-powered financial collaboration platform",
    version="1.0.0"
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 路由注册
app.include_router(auth.router, prefix="/api/auth", tags=["Authentication"])
app.include_router(users.router, prefix="/api/users", tags=["Users"])
app.include_router(agents.router, prefix="/api/agents", tags=["Agents"])
app.include_router(signals.router, prefix="/api/signals", tags=["Signals"])
app.include_router(groups.router, prefix="/api/groups", tags=["Groups"])
app.include_router(market.router, prefix="/api/market", tags=["Market Data"])

@app.get("/")
async def root():
    return {"message": "FinVerse API", "version": "1.0.0"}

@app.get("/health")
async def health_check():
    return {"status": "healthy"}

if __name__ == "__main__":
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

### 1.4 API Endpoints 设计

创建 `backend/api/auth.py`:

```python
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel, EmailStr
from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext

router = APIRouter()
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="api/auth/login")

# === Models ===
class UserRegister(BaseModel):
    email: EmailStr
    username: str
    password: str
    display_name: str | None = None
    timezone: str = "UTC"
    language: str = "en"

class Token(BaseModel):
    access_token: str
    token_type: str
    expires_in: int

class UserProfile(BaseModel):
    id: str
    email: str
    username: str
    display_name: str | None
    avatar_url: str | None
    timezone: str
    language: str
    subscription_tier: str
    created_at: datetime

# === Endpoints ===

@router.post("/register", response_model=UserProfile, status_code=status.HTTP_201_CREATED)
async def register(data: UserRegister):
    """
    注册新用户
    
    流程:
    1. 验证邮箱和用户名唯一性
    2. 哈希密码
    3. 创建用户记录
    4. 创建用户偏好记录
    5. 返回用户信息 (不返回敏感信息)
    """
    # 实现略 (连接数据库, 插入记录)
    pass

@router.post("/login", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """
    用户登录
    
    流程:
    1. 验证用户名/邮箱 + 密码
    2. 生成 JWT token
    3. 返回 access_token
    """
    # 实现略
    pass

@router.get("/me", response_model=UserProfile)
async def get_current_user(token: str = Depends(oauth2_scheme)):
    """
    获取当前登录用户信息
    
    从 JWT token 解析用户 ID, 查询数据库返回完整信息
    """
    # 实现略
    pass

@router.post("/logout")
async def logout(token: str = Depends(oauth2_scheme)):
    """
    登出 (可选: 将 token 加入黑名单)
    """
    return {"message": "Logged out successfully"}
```

创建 `backend/api/users.py`:

```python
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from typing import List

router = APIRouter()

# === Models ===
class APIKeyCreate(BaseModel):
    service_type: str  # openai, anthropic, deepseek, coingecko, etc.
    api_key: str
    key_name: str | None = None

class APIKeyResponse(BaseModel):
    id: str
    service_type: str
    key_name: str | None
    is_active: bool
    last_validated_at: datetime | None
    masked_key: str  # 只显示后4位: "sk-...abc123"

class PreferencesUpdate(BaseModel):
    trading_styles: List[str] | None
    analysis_preferences: List[str] | None
    asset_preferences: List[str] | None
    risk_profile: str | None
    default_view_mode: str | None
    notification_channels: List[str] | None
    quiet_hours_start: str | None  # "22:00"
    quiet_hours_end: str | None    # "08:00"

# === Endpoints ===

@router.get("/preferences")
async def get_preferences(current_user = Depends(get_current_user)):
    """获取用户偏好设置"""
    pass

@router.patch("/preferences")
async def update_preferences(
    data: PreferencesUpdate,
    current_user = Depends(get_current_user)
):
    """更新用户偏好"""
    pass

@router.get("/api-keys", response_model=List[APIKeyResponse])
async def list_api_keys(current_user = Depends(get_current_user)):
    """列出用户的所有 API keys (脱敏)"""
    pass

@router.post("/api-keys", response_model=APIKeyResponse, status_code=201)
async def add_api_key(
    data: APIKeyCreate,
    current_user = Depends(get_current_user)
):
    """
    添加 API key
    
    流程:
    1. 验证 key 格式
    2. 测试 key 是否有效 (调用对应服务的简单 API)
    3. AES-256 加密存储
    4. 通知用户的 Agent 重新加载配置
    """
    pass

@router.delete("/api-keys/{key_id}")
async def delete_api_key(
    key_id: str,
    current_user = Depends(get_current_user)
):
    """删除 API key"""
    pass

@router.post("/api-keys/{key_id}/validate")
async def validate_api_key(
    key_id: str,
    current_user = Depends(get_current_user)
):
    """
    验证 API key 是否仍然有效
    
    调用对应服务测试, 更新 last_validated_at 和 last_error
    """
    pass
```

创建 `backend/api/agents.py`:

```python
from fastapi import APIRouter, Depends
from pydantic import BaseModel
from typing import List, Dict

router = APIRouter()

# === Models ===
class AgentStatus(BaseModel):
    agent_id: str
    container_id: str | None
    status: str  # creating, running, paused, stopped, error
    resource_tier: str
    last_heartbeat_at: datetime | None
    uptime_seconds: int | None
    
class AgentQuery(BaseModel):
    query: str  # 用户的自然语言查询

class AgentResponse(BaseModel):
    response: str
    data: Dict | None  # 如果有结构化数据
    charts: List[Dict] | None  # 图表配置
    
class ChatConnectionCreate(BaseModel):
    platform: str  # telegram, whatsapp, discord
    auth_code: str  # OAuth code 或 bot token

# === Endpoints ===

@router.get("/status", response_model=AgentStatus)
async def get_agent_status(current_user = Depends(get_current_user)):
    """
    获取用户 Agent 的状态
    
    如果 Agent 不存在, 返回 404
    如果 Agent 在创建中, 返回 creating 状态
    """
    pass

@router.post("/start")
async def start_agent(current_user = Depends(get_current_user)):
    """
    启动/创建 Agent
    
    流程:
    1. 检查是否已有 Agent
    2. 如果没有, 调用 orchestrator 创建新容器
    3. 初始化 OpenClaw workspace
    4. 生成 SOUL.md (基于用户偏好)
    5. 设置 Cron 任务
    6. 返回 agent_id
    """
    pass

@router.post("/stop")
async def stop_agent(current_user = Depends(get_current_user)):
    """暂停 Agent (容器停止, 但保留数据)"""
    pass

@router.delete("/")
async def delete_agent(current_user = Depends(get_current_user)):
    """
    删除 Agent
    
    警告: 删除容器和所有数据, 不可恢复
    """
    pass

@router.post("/query", response_model=AgentResponse)
async def query_agent(
    data: AgentQuery,
    current_user = Depends(get_current_user)
):
    """
    向 Agent 发送查询 (从网页端)
    
    流程:
    1. 将查询放入 Agent 的任务队列 (Redis)
    2. Agent 处理后将结果写回 Redis
    3. 通过 WebSocket 推送结果到前端
    4. 同时返回结果 (如果超时则返回 processing 状态)
    """
    pass

@router.get("/chat-connections", response_model=List[Dict])
async def list_chat_connections(current_user = Depends(get_current_user)):
    """列出已连接的聊天通道"""
    pass

@router.post("/chat-connections", status_code=201)
async def add_chat_connection(
    data: ChatConnectionCreate,
    current_user = Depends(get_current_user)
):
    """
    添加聊天通道连接
    
    流程 (以 Telegram 为例):
    1. 用户在前端点击 "连接 Telegram"
    2. 前端打开 Telegram bot 链接, 用户点击 Start
    3. Bot 生成一个临时 auth_code, 让用户复制
    4. 用户在网页粘贴 auth_code
    5. 后端验证 auth_code, 关联 telegram_user_id 和 finverse_user_id
    6. Agent 开始监听 Telegram 消息
    """
    pass

@router.delete("/chat-connections/{connection_id}")
async def remove_chat_connection(
    connection_id: str,
    current_user = Depends(get_current_user)
):
    """断开聊天通道"""
    pass
```

创建 `backend/api/signals.py`:

```python
from fastapi import APIRouter, Depends, Query
from pydantic import BaseModel
from typing import List, Dict
from datetime import datetime

router = APIRouter()

# === Models ===
class SignalDimension(BaseModel):
    signal: str  # bullish, bearish, neutral
    confidence: float
    summary: str

class SignalCreate(BaseModel):
    asset: str
    signal: str
    confidence: float
    dimensions: Dict[str, SignalDimension]  # {on_chain: {...}, macro: {...}, ...}
    key_levels: Dict | None
    reasoning: str | None
    timeframe: str
    valid_until: datetime | None

class SignalResponse(BaseModel):
    id: str
    agent_id: str
    user_id: str
    asset: str
    signal: str
    confidence: float
    dimensions: Dict[str, SignalDimension]
    key_levels: Dict | None
    timeframe: str
    reputation_score: float
    created_at: datetime

class ConsensusResponse(BaseModel):
    asset: str
    timeframe: str
    total_signals: int
    bullish_pct: float
    bearish_pct: float
    neutral_pct: float
    avg_confidence: float
    top_dimensions: List[Dict]  # 被引用最多的维度
    divergence_score: float  # 0-1, 分歧程度
    
# === Endpoints ===

@router.post("/", response_model=SignalResponse, status_code=201)
async def publish_signal(
    data: SignalCreate,
    current_user = Depends(get_current_user)
):
    """
    发布公域信号
    
    流程:
    1. 验证用户有活跃的 Agent
    2. 插入 public_signals 表
    3. 发布到 Redis pub/sub (channel:signals)
    4. 触发订阅者的 Agent 更新
    """
    pass

@router.get("/", response_model=List[SignalResponse])
async def list_signals(
    asset: str | None = None,
    timeframe: str | None = None,
    signal: str | None = None,
    min_confidence: float = 0.0,
    limit: int = Query(50, le=200),
    offset: int = 0,
    current_user = Depends(get_current_user)
):
    """
    查询公域信号
    
    支持按资产、时间框架、信号类型、最小置信度筛选
    """
    pass

@router.get("/consensus/{asset}", response_model=ConsensusResponse)
async def get_consensus(
    asset: str,
    timeframe: str = "24h",
    current_user = Depends(get_current_user)
):
    """
    获取某个资产的共识数据
    
    流程:
    1. 查询最近 N 小时的所有信号
    2. 计算看多/看空/中性比例
    3. 分析分歧维度
    4. 返回聚合结果
    
    结果缓存 5 分钟 (Redis)
    """
    pass

@router.get("/subscriptions")
async def get_subscriptions(current_user = Depends(get_current_user)):
    """获取用户的信号订阅配置"""
    pass

@router.post("/subscriptions")
async def update_subscriptions(
    assets: List[str],
    min_confidence: float,
    dimensions: List[str],
    current_user = Depends(get_current_user)
):
    """更新信号订阅配置"""
    pass
```

创建 `backend/api/groups.py`:

```python
from fastapi import APIRouter, Depends, Query, UploadFile, File
from pydantic import BaseModel
from typing import List, Dict

router = APIRouter()

# === Models ===
class GroupCreate(BaseModel):
    name: str
    description: str | None
    focus_assets: List[str]
    focus_styles: List[str]
    is_public: bool = False
    join_approval_required: bool = True

class GroupResponse(BaseModel):
    id: str
    name: str
    description: str | None
    avatar_url: str | None
    focus_assets: List[str]
    focus_styles: List[str]
    is_public: bool
    member_count: int
    analysis_count: int
    created_by: str
    created_at: datetime
    user_role: str | None  # 当前用户在小组的角色 (如果已加入)
    user_status: str | None  # 当前用户在小组的状态

class AnalysisCreate(BaseModel):
    title: str
    asset: str
    signal: str
    confidence: float | None
    user_input: str  # 用户的自然语言输入, AI 会扩展

class AnalysisResponse(BaseModel):
    id: str
    group_id: str
    user_id: str
    user_display_name: str
    user_avatar_url: str | None
    title: str
    asset: str
    signal: str
    confidence: float | None
    content_markdown: str
    data_snapshot: Dict | None
    charts: List[Dict] | None
    view_count: int
    upvote_count: int
    downvote_count: int
    comment_count: int
    published_at: datetime
    is_pinned: bool

# === Endpoints ===

@router.post("/", response_model=GroupResponse, status_code=201)
async def create_group(
    data: GroupCreate,
    current_user = Depends(get_current_user)
):
    """
    创建小组
    
    流程:
    1. 验证用户订阅等级 (free 用户不能创建小组)
    2. 创建 groups 记录
    3. 添加创建者为 owner
    4. 返回小组信息
    """
    pass

@router.get("/", response_model=List[GroupResponse])
async def list_groups(
    is_public: bool | None = None,
    focus_asset: str | None = None,
    limit: int = Query(20, le=100),
    offset: int = 0,
    current_user = Depends(get_current_user)
):
    """
    列出小组
    
    返回:
    - 用户已加入的小组
    - 公开可见的小组 (如果 is_public=True)
    """
    pass

@router.get("/{group_id}", response_model=GroupResponse)
async def get_group(
    group_id: str,
    current_user = Depends(get_current_user)
):
    """获取小组详情"""
    pass

@router.post("/{group_id}/join")
async def join_group(
    group_id: str,
    current_user = Depends(get_current_user)
):
    """
    加入小组
    
    流程:
    1. 检查小组是否存在, 是否已满
    2. 如果需要审批, 创建 pending 状态记录, 通知 owner/admin
    3. 如果不需要审批, 直接创建 active 记录
    4. 返回状态
    """
    pass

@router.post("/{group_id}/leave")
async def leave_group(
    group_id: str,
    current_user = Depends(get_current_user)
):
    """退出小组"""
    pass

@router.post("/{group_id}/analyses", response_model=AnalysisResponse, status_code=201)
async def create_analysis(
    group_id: str,
    data: AnalysisCreate,
    current_user = Depends(get_current_user)
):
    """
    发布分析
    
    流程 (核心差异化功能):
    1. 验证用户是小组成员
    2. 调用用户的 Agent: "请根据用户输入生成结构化分析"
    3. Agent 返回: Markdown 内容 + 数据快照 + 图表配置
    4. 插入 group_analyses 表
    5. 通知小组其他成员 (通过聊天软件)
    6. 发布到 Redis pub/sub
    7. 返回分析内容
    
    用户只需要说一句话, Agent 帮他生成完整的分析报告!
    """
    pass

@router.get("/{group_id}/analyses", response_model=List[AnalysisResponse])
async def list_analyses(
    group_id: str,
    asset: str | None = None,
    limit: int = Query(20, le=100),
    offset: int = 0,
    current_user = Depends(get_current_user)
):
    """列出小组的分析"""
    pass

@router.get("/{group_id}/analyses/{analysis_id}", response_model=AnalysisResponse)
async def get_analysis(
    group_id: str,
    analysis_id: str,
    current_user = Depends(get_current_user)
):
    """
    获取分析详情
    
    自动增加 view_count
    """
    pass

@router.post("/{group_id}/analyses/{analysis_id}/vote")
async def vote_analysis(
    group_id: str,
    analysis_id: str,
    vote_type: str,  # upvote, downvote
    current_user = Depends(get_current_user)
):
    """对分析投票"""
    pass

@router.get("/{group_id}/consensus")
async def get_group_consensus(
    group_id: str,
    timeframe: str = "24h",
    asset: str | None = None,
    current_user = Depends(get_current_user)
):
    """
    获取小组共识
    
    调用 AI 生成共识报告 (缓存结果)
    """
    pass
```

创建 `backend/api/market.py`:

```python
from fastapi import APIRouter, Depends, Query
from pydantic import BaseModel
from typing import List, Dict

router = APIRouter()

# === Models ===
class PriceData(BaseModel):
    asset: str
    price: float
    change_24h: float
    volume_24h: float
    timestamp: datetime

class AnomalyEvent(BaseModel):
    id: str
    asset: str
    anomaly_type: str
    severity: str
    description: str
    data_snapshot: Dict
    detected_at: datetime

# === Endpoints ===

@router.get("/price/{asset}", response_model=PriceData)
async def get_price(
    asset: str,
    current_user = Depends(get_current_user)
):
    """
    获取实时价格
    
    优先从 Redis 缓存读取 (10秒TTL)
    缓存未命中则调用数据源 API
    """
    pass

@router.get("/anomalies", response_model=List[AnomalyEvent])
async def list_anomalies(
    asset: str | None = None,
    severity: str | None = None,
    limit: int = Query(20, le=100),
    current_user = Depends(get_current_user)
):
    """
    列出最近的异常事件
    
    由监控脚本产出, 存储在 anomaly_events 表
    """
    pass
```

---

## 阶段 2: Agent 调度器 (Orchestrator)

### 2.1 Orchestrator 核心逻辑

创建 `backend/orchestrator/manager.py`:

```python
import docker
import json
import os
from pathlib import Path
from typing import Dict
from cryptography.fernet import Fernet
import uuid

class AgentOrchestrator:
    def __init__(self):
        self.docker_client = docker.from_env()
        self.workspace_base = Path(os.getenv("OPENCLAW_WORKSPACE_BASE", "/var/lib/finverse/agents"))
        self.encryption_key = Fernet.generate_key()  # 实际应从环境变量或密钥管理服务读取
        self.cipher = Fernet(self.encryption_key)
        
    async def create_agent(self, user_id: str, user_data: Dict) -> str:
        """
        创建新的 Agent 实例
        
        Args:
            user_id: 用户 ID
            user_data: 包含 API keys, 偏好等信息
            
        Returns:
            agent_id
        """
        agent_id = str(uuid.uuid4())
        workspace_path = self.workspace_base / user_id / agent_id
        workspace_path.mkdir(parents=True, exist_ok=True)
        
        # 1. 生成 Agent 配置文件
        await self._generate_soul_md(workspace_path, user_data)
        await self._generate_tools_md(workspace_path, user_data)
        await self._setup_api_keys(workspace_path, user_data["api_keys"])
        
        # 2. 创建 Docker 容器
        container = self.docker_client.containers.run(
            image="finverse/agent:latest",
            name=f"agent-{user_id}-{agent_id}",
            detach=True,
            environment={
                "AGENT_ID": agent_id,
                "USER_ID": user_id,
                "WORKSPACE_PATH": str(workspace_path),
            },
            volumes={
                str(workspace_path): {"bind": "/root/.openclaw/workspace", "mode": "rw"}
            },
            network_mode="finverse_network",
            restart_policy={"Name": "unless-stopped"},
            mem_limit="512m",  # 限制内存
            cpu_quota=50000,   # 限制 CPU (50%)
        )
        
        # 3. 记录到数据库
        # (插入 agent_instances 表)
        
        # 4. 设置 Cron 任务
        await self._setup_cron_jobs(agent_id, user_data)
        
        return agent_id
        
    async def _generate_soul_md(self, workspace_path: Path, user_data: Dict):
        """
        根据用户偏好生成 SOUL.md
        """
        preferences = user_data.get("preferences", {})
        trading_styles = ", ".join(preferences.get("trading_styles", []))
        analysis_prefs = ", ".join(preferences.get("analysis_preferences", []))
        asset_prefs = ", ".join(preferences.get("asset_preferences", []))
        
        soul_content = f"""# 你是谁

你是 {user_data['display_name']} 的个人金融 Agent。你的任务是帮助他在金融市场做出更好的决策。

## 交易风格

{trading_styles}

## 分析偏好

用户更关注: {analysis_prefs}

## 资产偏好

用户主要关注: {asset_prefs}

## 风险偏好

{preferences.get('risk_profile', 'balanced')}

## 你的职责

1. **每日摘要**: 每天开盘前推送市场摘要
2. **异常预警**: 监控异常事件, 及时通知
3. **查询响应**: 回答用户的市场问题
4. **信号发布**: 定期分析并发布结构化信号到公域
5. **小组协作**: 辅助用户在小组中发布和讨论分析

## 重要规则

- 所有分析必须基于数据, 附上数据来源
- 不做投资建议, 只提供信息和分析
- 保持客观, 呈现多方观点
- 记住用户的历史判断和教训 (写入 MEMORY.md)

## 语言

所有输出使用: {user_data.get('language', 'en')}

## 时区

用户时区: {user_data.get('timezone', 'UTC')}
"""
        (workspace_path / "SOUL.md").write_text(soul_content)
        
    async def _generate_tools_md(self, workspace_path: Path, user_data: Dict):
        """
        生成 TOOLS.md (用户的数据源配置)
        """
        api_keys = user_data.get("api_keys", [])
        tools_content = "# TOOLS.md - 数据源配置\n\n"
        
        for key_info in api_keys:
            tools_content += f"## {key_info['service_type']}\n"
            tools_content += f"- 状态: {'✓ 已配置' if key_info['is_active'] else '✗ 未激活'}\n"
            tools_content += f"- Key: {key_info['masked_key']}\n\n"
            
        (workspace_path / "TOOLS.md").write_text(tools_content)
        
    async def _setup_api_keys(self, workspace_path: Path, api_keys: List[Dict]):
        """
        加密存储 API keys
        
        存储在 workspace/.secrets/keys.enc
        """
        secrets_path = workspace_path / ".secrets"
        secrets_path.mkdir(exist_ok=True)
        
        keys_data = {}
        for key_info in api_keys:
            encrypted_key = self.cipher.encrypt(key_info["api_key"].encode())
            keys_data[key_info["service_type"]] = encrypted_key.decode()
            
        (secrets_path / "keys.enc").write_text(json.dumps(keys_data))
        
    async def _setup_cron_jobs(self, agent_id: str, user_data: Dict):
        """
        设置 Agent 的定时任务
        
        通过 OpenClaw 的 cron 系统或直接调用容器内脚本
        """
        timezone = user_data.get("timezone", "UTC")
        
        # 任务列表
        jobs = [
            {
                "name": "daily_morning_summary",
                "schedule": "0 8 * * *",  # 每天 8:00 (用户时区)
                "command": "openclaw exec 'python /app/scripts/morning_summary.py'"
            },
            {
                "name": "market_scan_4h",
                "schedule": "0 */4 * * *",  # 每 4 小时
                "command": "openclaw exec 'python /app/scripts/market_scan.py'"
            },
            {
                "name": "daily_evening_recap",
                "schedule": "0 20 * * *",  # 每天 20:00
                "command": "openclaw exec 'python /app/scripts/evening_recap.py'"
            }
        ]
        
        # 实际实现: 将任务写入容器的 crontab 或使用调度系统
        pass
        
    async def pause_agent(self, agent_id: str):
        """暂停 Agent (容器停止)"""
        container = self.docker_client.containers.get(f"agent-{agent_id}")
        container.pause()
        
    async def resume_agent(self, agent_id: str):
        """恢复 Agent"""
        container = self.docker_client.containers.get(f"agent-{agent_id}")
        container.unpause()
        
    async def delete_agent(self, agent_id: str, user_id: str):
        """删除 Agent (容器 + workspace)"""
        # 1. 停止并删除容器
        container = self.docker_client.containers.get(f"agent-{agent_id}")
        container.stop()
        container.remove()
        
        # 2. 删除 workspace
        workspace_path = self.workspace_base / user_id / agent_id
        shutil.rmtree(workspace_path)
        
        # 3. 删除数据库记录
        pass
        
    async def get_agent_status(self, agent_id: str) -> Dict:
        """获取 Agent 状态"""
        try:
            container = self.docker_client.containers.get(f"agent-{agent_id}")
            stats = container.stats(stream=False)
            
            return {
                "status": container.status,
                "cpu_usage": self._calculate_cpu_percent(stats),
                "memory_usage_mb": stats["memory_stats"]["usage"] / 1024 / 1024,
                "uptime_seconds": (datetime.now() - container.attrs["State"]["StartedAt"]).total_seconds()
            }
        except docker.errors.NotFound:
            return {"status": "not_found"}
            
    def _calculate_cpu_percent(self, stats: Dict) -> float:
        """计算 CPU 使用率"""
        cpu_delta = stats["cpu_stats"]["cpu_usage"]["total_usage"] - stats["precpu_stats"]["cpu_usage"]["total_usage"]
        system_delta = stats["cpu_stats"]["system_cpu_usage"] - stats["precpu_stats"]["system_cpu_usage"]
        cpu_count = stats["cpu_stats"]["online_cpus"]
        
        if system_delta > 0 and cpu_delta > 0:
            return (cpu_delta / system_delta) * cpu_count * 100.0
        return 0.0
```

### 2.2 健康检查与自动恢复

创建 `backend/orchestrator/health_monitor.py`:

```python
import asyncio
from datetime import datetime, timedelta
from typing import List

class HealthMonitor:
    def __init__(self, orchestrator: AgentOrchestrator):
        self.orchestrator = orchestrator
        self.check_interval = 60  # 每分钟检查一次
        
    async def run(self):
        """持续运行的健康检查守护进程"""
        while True:
            await self.check_all_agents()
            await asyncio.sleep(self.check_interval)
            
    async def check_all_agents(self):
        """检查所有 Agent 的健康状态"""
        # 从数据库获取所有活跃的 Agent
        agents = await self._get_active_agents()
        
        for agent in agents:
            try:
                status = await self.orchestrator.get_agent_status(agent["id"])
                
                if status["status"] == "not_found":
                    # 容器丢失, 尝试重建
                    await self._recover_agent(agent)
                elif status["status"] == "exited":
                    # 容器退出, 重启
                    await self.orchestrator.resume_agent(agent["id"])
                elif agent["last_heartbeat_at"] and (datetime.now() - agent["last_heartbeat_at"]) > timedelta(minutes=5):
                    # 超过 5 分钟无心跳, 可能僵死
                    await self._restart_agent(agent)
                    
                # 更新数据库状态
                await self._update_agent_status(agent["id"], status)
                
            except Exception as e:
                # 记录错误日志
                print(f"Health check failed for agent {agent['id']}: {e}")
                
    async def _recover_agent(self, agent: Dict):
        """恢复丢失的 Agent"""
        # 从数据库读取用户配置, 重新创建容器
        user_data = await self._get_user_data(agent["user_id"])
        
        # 重新创建容器, 挂载原 workspace
        # (实现略, 类似 create_agent 但不重新生成 workspace)
        pass
        
    async def _restart_agent(self, agent: Dict):
        """重启僵死的 Agent"""
        container = self.orchestrator.docker_client.containers.get(f"agent-{agent['id']}")
        container.restart()
```

---

## 阶段 3: 监控脚本基础设施

### 3.1 监控脚本设计

创建 `backend/monitoring/daemon.py`:

```python
import asyncio
import aiohttp
from datetime import datetime
from typing import List, Dict

class MarketMonitor:
    """
    市场监控守护进程
    
    职责:
    - 持续采集市场数据 (免费数据源)
    - 检测异常 (价格突变, 成交量突变, 链上异常等)
    - 异常发生时: 记录到数据库 + 唤醒相关 Agent
    
    不使用 AI (成本太高), 只做简单阈值判断
    """
    
    def __init__(self):
        self.watched_assets = []  # 从数据库读取所有用户关注的资产
        self.thresholds = {
            "price_change_1m": 0.02,  # 1分钟涨跌超过 2%
            "volume_spike": 3.0,      # 成交量是 1h 均值的 3 倍
        }
        
    async def run(self):
        """主循环"""
        while True:
            await self.update_watched_assets()
            await self.check_all_assets()
            await asyncio.sleep(10)  # 每 10 秒检查一次
            
    async def update_watched_assets(self):
        """从数据库更新需要监控的资产列表"""
        # 查询所有用户的 asset_preferences
        # 去重得到唯一资产列表
        pass
        
    async def check_all_assets(self):
        """检查所有资产"""
        tasks = [self.check_asset(asset) for asset in self.watched_assets]
        await asyncio.gather(*tasks)
        
    async def check_asset(self, asset: str):
        """检查单个资产"""
        try:
            # 1. 获取当前数据
            current_data = await self.fetch_market_data(asset)
            
            # 2. 获取历史数据 (从 Redis 缓存)
            historical_data = await self.get_historical_data(asset)
            
            # 3. 检测异常
            anomalies = self.detect_anomalies(asset, current_data, historical_data)
            
            # 4. 如果有异常, 记录并通知
            if anomalies:
                await self.handle_anomalies(asset, anomalies, current_data)
                
            # 5. 更新缓存
            await self.update_cache(asset, current_data)
            
        except Exception as e:
            print(f"Error checking {asset}: {e}")
            
    async def fetch_market_data(self, asset: str) -> Dict:
        """从数据源获取实时数据"""
        # 示例: 调用 CoinGecko API
        async with aiohttp.ClientSession() as session:
            async with session.get(f"https://api.coingecko.com/api/v3/simple/price?ids={asset}&vs_currencies=usd&include_24hr_vol=true") as resp:
                data = await resp.json()
                return {
                    "price": data[asset]["usd"],
                    "volume_24h": data[asset]["usd_24h_vol"],
                    "timestamp": datetime.now()
                }
                
    def detect_anomalies(self, asset: str, current: Dict, historical: List[Dict]) -> List[Dict]:
        """检测异常"""
        anomalies = []
        
        # 价格突变检测
        if len(historical) > 0:
            last_price = historical[-1]["price"]
            price_change = abs(current["price"] - last_price) / last_price
            
            if price_change > self.thresholds["price_change_1m"]:
                anomalies.append({
                    "type": "price_spike",
                    "severity": "high" if price_change > 0.05 else "medium",
                    "description": f"{asset} 价格 1 分钟变动 {price_change*100:.2f}%"
                })
                
        # 成交量突变检测
        if len(historical) >= 60:  # 至少 1 小时数据
            avg_volume = sum(h["volume_24h"] for h in historical[-60:]) / 60
            volume_ratio = current["volume_24h"] / avg_volume if avg_volume > 0 else 0
            
            if volume_ratio > self.thresholds["volume_spike"]:
                anomalies.append({
                    "type": "volume_spike",
                    "severity": "medium",
                    "description": f"{asset} 成交量是均值的 {volume_ratio:.1f} 倍"
                })
                
        return anomalies
        
    async def handle_anomalies(self, asset: str, anomalies: List[Dict], data: Dict):
        """处理检测到的异常"""
        for anomaly in anomalies:
            # 1. 记录到数据库
            anomaly_id = await self.save_anomaly(asset, anomaly, data)
            
            # 2. 发布到 Redis pub/sub
            await redis.publish("channel:anomalies", json.dumps({
                "asset": asset,
                "anomaly": anomaly,
                "data": data
            }))
            
            # 3. 唤醒关注该资产的 Agent
            await self.notify_agents(asset, anomaly_id)
            
    async def notify_agents(self, asset: str, anomaly_id: str):
        """通知关注该资产的 Agent"""
        # 查询数据库: 哪些用户关注这个资产?
        # 将任务放入各 Agent 的任务队列 (Redis)
        # Agent 醒来后处理任务, 决定是否推送给用户
        pass
```

### 3.2 Agent 异常处理脚本

创建 Agent 容器内的 `scripts/handle_anomaly.py`:

```python
import sys
import json
from datetime import datetime

def handle_anomaly(anomaly_id: str):
    """
    Agent 处理异常事件
    
    流程:
    1. 从 API 获取异常详情
    2. AI 分析: 这个异常对用户重要吗? 严重程度?
    3. 如果重要, 推送给用户 (聊天软件)
    4. 发布信号到公域 (可选)
    """
    
    # 1. 获取异常详情
    anomaly = fetch_anomaly(anomaly_id)
    
    # 2. AI 分析
    analysis = analyze_anomaly_with_ai(anomaly)
    
    # 3. 决定是否推送
    if analysis["should_notify"]:
        send_notification(analysis["message"])
        
    # 4. 更新 MEMORY.md
    update_memory(f"异常检测: {anomaly['asset']} - {anomaly['description']}")
    
def analyze_anomaly_with_ai(anomaly: Dict) -> Dict:
    """
    调用 AI 分析异常
    
    Prompt:
    - 用户关注的资产: [从 SOUL.md 读取]
    - 当前异常: {anomaly}
    - 问题: 这个异常对用户重要吗? 如果重要, 应该怎么通知?
    
    返回:
    {
        "should_notify": true/false,
        "severity": "low/medium/high",
        "message": "推送给用户的消息"
    }
    """
    # 实现略 (调用 OpenAI/Anthropic API)
    pass

if __name__ == "__main__":
    anomaly_id = sys.argv[1]
    handle_anomaly(anomaly_id)
```

---

## 阶段 4: 前端基础架构

### 4.1 Next.js 项目初始化

```bash
cd frontend
npx create-next-app@latest . --typescript --tailwind --app
npm install @tanstack/react-query axios zustand lightweight-charts socket.io-client
npm install @radix-ui/react-dropdown-menu @radix-ui/react-dialog @radix-ui/react-tabs
npm install recharts framer-motion lucide-react
```

### 4.2 项目结构

```
frontend/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/
│   │   ├── layout.tsx          # Dashboard 布局
│   │   ├── page.tsx             # 主看板
│   │   ├── signals/             # 公域信号
│   │   ├── groups/              # 小组
│   │   └── settings/            # 设置
│   ├── layout.tsx               # 全局布局
│   └── providers.tsx            # Context Providers
├── components/
│   ├── ui/                      # 基础 UI 组件
│   ├── charts/                  # 图表组件
│   ├── dashboard/               # 看板组件
│   ├── signals/                 # 信号组件
│   └── groups/                  # 小组组件
├── lib/
│   ├── api.ts                   # API 客户端
│   ├── websocket.ts             # WebSocket 管理
│   ├── store.ts                 # Zustand store
│   └── utils.ts                 # 工具函数
└── styles/
    └── globals.css
```

### 4.3 全局样式 (FinVerse 暗色主题)

创建 `frontend/styles/globals.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 210 40% 5%;        /* #0a0e17 */
    --foreground: 210 40% 98%;       /* #f8f9fb */
    
    --card: 210 30% 8%;              /* #0f141f */
    --card-foreground: 210 40% 98%;
    
    --primary: 200 100% 60%;         /* #33a3ff - 亮蓝色 */
    --primary-foreground: 210 40% 98%;
    
    --secondary: 270 60% 65%;        /* #9d7cf5 - 紫色 */
    --secondary-foreground: 210 40% 98%;
    
    --accent: 160 70% 50%;           /* #26d9a6 - 青绿色 */
    --accent-foreground: 210 40% 5%;
    
    --success: 120 60% 50%;          /* #4caf50 - 绿色 (看涨) */
    --danger: 0 70% 60%;             /* #f44336 - 红色 (看跌) */
    --warning: 45 100% 55%;          /* #ffa726 - 橙色 (中性/警告) */
    
    --muted: 210 30% 15%;
    --muted-foreground: 210 20% 60%;
    
    --border: 210 30% 18%;
    --input: 210 30% 18%;
    --ring: 200 100% 60%;
    
    --radius: 0.5rem;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
    font-feature-settings: "rlig" 1, "calt" 1;
  }
}

@layer utilities {
  .text-balance {
    text-wrap: balance;
  }
  
  /* 自定义滚动条 */
  .custom-scrollbar::-webkit-scrollbar {
    width: 6px;
    height: 6px;
  }
  
  .custom-scrollbar::-webkit-scrollbar-track {
    @apply bg-transparent;
  }
  
  .custom-scrollbar::-webkit-scrollbar-thumb {
    @apply bg-muted rounded-full;
  }
  
  .custom-scrollbar::-webkit-scrollbar-thumb:hover {
    @apply bg-muted-foreground/50;
  }
  
  /* 毛玻璃效果 */
  .glass {
    @apply bg-card/60 backdrop-blur-md border border-border;
  }
  
  /* 信号颜色 */
  .signal-bullish {
    @apply text-green-500;
  }
  
  .signal-bearish {
    @apply text-red-500;
  }
  
  .signal-neutral {
    @apply text-yellow-500;
  }
}
```

### 4.4 API 客户端

创建 `frontend/lib/api.ts`:

```typescript
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';

class APIClient {
  private client: AxiosInstance;
  
  constructor() {
    this.client = axios.create({
      baseURL: process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000/api',
      headers: {
        'Content-Type': 'application/json',
      },
    });
    
    // 请求拦截器: 添加 token
    this.client.interceptors.request.use((config) => {
      const token = localStorage.getItem('access_token');
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;
      }
      return config;
    });
    
    // 响应拦截器: 处理错误
    this.client.interceptors.response.use(
      (response) => response,
      (error) => {
        if (error.response?.status === 401) {
          // Token 过期, 跳转登录
          localStorage.removeItem('access_token');
          window.location.href = '/login';
        }
        return Promise.reject(error);
      }
    );
  }
  
  // === Auth ===
  async register(data: { email: string; username: string; password: string }) {
    const res = await this.client.post('/auth/register', data);
    return res.data;
  }
  
  async login(username: string, password: string) {
    const formData = new FormData();
    formData.append('username', username);
    formData.append('password', password);
    
    const res = await this.client.post('/auth/login', formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
    });
    
    localStorage.setItem('access_token', res.data.access_token);
    return res.data;
  }
  
  async getCurrentUser() {
    const res = await this.client.get('/auth/me');
    return res.data;
  }
  
  // === Users ===
  async getPreferences() {
    const res = await this.client.get('/users/preferences');
    return res.data;
  }
  
  async updatePreferences(data: any) {
    const res = await this.client.patch('/users/preferences', data);
    return res.data;
  }
  
  async listAPIKeys() {
    const res = await this.client.get('/users/api-keys');
    return res.data;
  }
  
  async addAPIKey(data: { service_type: string; api_key: string; key_name?: string }) {
    const res = await this.client.post('/users/api-keys', data);
    return res.data;
  }
  
  async deleteAPIKey(keyId: string) {
    await this.client.delete(`/users/api-keys/${keyId}`);
  }
  
  // === Agents ===
  async getAgentStatus() {
    const res = await this.client.get('/agents/status');
    return res.data;
  }
  
  async startAgent() {
    const res = await this.client.post('/agents/start');
    return res.data;
  }
  
  async queryAgent(query: string) {
    const res = await this.client.post('/agents/query', { query });
    return res.data;
  }
  
  // === Signals ===
  async listSignals(params?: { asset?: string; timeframe?: string; limit?: number }) {
    const res = await this.client.get('/signals', { params });
    return res.data;
  }
  
  async getConsensus(asset: string, timeframe: string = '24h') {
    const res = await this.client.get(`/signals/consensus/${asset}`, { params: { timeframe } });
    return res.data;
  }
  
  // === Groups ===
  async listGroups(params?: { is_public?: boolean; limit?: number }) {
    const res = await this.client.get('/groups', { params });
    return res.data;
  }
  
  async getGroup(groupId: string) {
    const res = await this.client.get(`/groups/${groupId}`);
    return res.data;
  }
  
  async createGroup(data: any) {
    const res = await this.client.post('/groups', data);
    return res.data;
  }
  
  async joinGroup(groupId: string) {
    const res = await this.client.post(`/groups/${groupId}/join`);
    return res.data;
  }
  
  async listGroupAnalyses(groupId: string, params?: { limit?: number }) {
    const res = await this.client.get(`/groups/${groupId}/analyses`, { params });
    return res.data;
  }
  
  async createAnalysis(groupId: string, data: any) {
    const res = await this.client.post(`/groups/${groupId}/analyses`, data);
    return res.data;
  }
  
  // === Market ===
  async getPrice(asset: string) {
    const res = await this.client.get(`/market/price/${asset}`);
    return res.data;
  }
  
  async listAnomalies(params?: { asset?: string; severity?: string }) {
    const res = await this.client.get('/market/anomalies', { params });
    return res.data;
  }
}

export const api = new APIClient();
```

### 4.5 WebSocket 管理

创建 `frontend/lib/websocket.ts`:

```typescript
import { io, Socket } from 'socket.io-client';

class WebSocketManager {
  private socket: Socket | null = null;
  private listeners: Map<string, Set<Function>> = new Map();
  
  connect(token: string) {
    if (this.socket?.connected) return;
    
    this.socket = io(process.env.NEXT_PUBLIC_WS_URL || 'ws://localhost:8000', {
      auth: { token },
      transports: ['websocket'],
    });
    
    this.socket.on('connect', () => {
      console.log('WebSocket connected');
    });
    
    this.socket.on('disconnect', () => {
      console.log('WebSocket disconnected');
    });
    
    // 订阅所有事件并分发给监听器
    this.socket.onAny((event, data) => {
      this.emit(event, data);
    });
  }
  
  disconnect() {
    this.socket?.disconnect();
    this.socket = null;
  }
  
  on(event: string, callback: Function) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(callback);
  }
  
  off(event: string, callback: Function) {
    this.listeners.get(event)?.delete(callback);
  }
  
  private emit(event: string, data: any) {
    this.listeners.get(event)?.forEach(callback => callback(data));
  }
  
  send(event: string, data: any) {
    this.socket?.emit(event, data);
  }
}

export const ws = new WebSocketManager();
```

---

## 阶段 5: 可视化组件开发

### 5.1 摘要模式组件

创建 `frontend/components/dashboard/SummaryView.tsx`:

```typescript
'use client';

import { useState, useEffect } from 'react';
import { api } from '@/lib/api';
import { ArrowUp, ArrowDown, Minus, AlertTriangle } from 'lucide-react';

interface Signal {
  asset: string;
  signal: 'bullish' | 'bearish' | 'neutral';
  confidence: number;
  dimensions: {
    [key: string]: {
      signal: string;
      confidence: number;
      summary: string;
    };
  };
}

export function SummaryView({ assets }: { assets: string[] }) {
  const [signals, setSignals] = useState<Signal[]>([]);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    loadSignals();
  }, [assets]);
  
  async function loadSignals() {
    setLoading(true);
    try {
      // 并行获取所有资产的共识
      const results = await Promise.all(
        assets.map(asset => api.getConsensus(asset, '24h'))
      );
      setSignals(results);
    } catch (error) {
      console.error('Failed to load signals:', error);
    } finally {
      setLoading(false);
    }
  }
  
  if (loading) {
    return <div className="animate-pulse">加载中...</div>;
  }
  
  return (
    <div className="space-y-4">
      {/* 标题 */}
      <div className="flex items-center justify-between">
        <h2 className="text-2xl font-bold">市场摘要</h2>
        <button onClick={loadSignals} className="text-sm text-muted-foreground hover:text-foreground">
          刷新
        </button>
      </div>
      
      {/* 资产卡片 */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {signals.map((signal) => (
          <AssetSummaryCard key={signal.asset} signal={signal} />
        ))}
      </div>
    </div>
  );
}

function AssetSummaryCard({ signal }: { signal: Signal }) {
  const SignalIcon = {
    bullish: ArrowUp,
    bearish: ArrowDown,
    neutral: Minus,
  }[signal.signal];
  
  const signalColor = {
    bullish: 'text-green-500',
    bearish: 'text-red-500',
    neutral: 'text-yellow-500',
  }[signal.signal];
  
  return (
    <div className="glass rounded-lg p-6 hover:border-primary/50 transition-colors">
      {/* 资产名称 + 信号 */}
      <div className="flex items-center justify-between mb-4">
        <h3 className="text-lg font-semibold">{signal.asset}</h3>
        <div className={`flex items-center gap-2 ${signalColor}`}>
          <SignalIcon className="w-5 h-5" />
          <span className="text-sm font-medium">
            {signal.confidence > 0.7 ? '强' : ''}
            {signal.signal === 'bullish' ? '看涨' : signal.signal === 'bearish' ? '看跌' : '中性'}
          </span>
        </div>
      </div>
      
      {/* 多维信号条 */}
      <div className="space-y-2">
        {Object.entries(signal.dimensions).map(([dimension, data]) => (
          <DimensionBar key={dimension} dimension={dimension} data={data} />
        ))}
      </div>
      
      {/* 置信度 */}
      <div className="mt-4 pt-4 border-t border-border">
        <div className="flex items-center justify-between text-sm">
          <span className="text-muted-foreground">共识置信度</span>
          <span className="font-medium">{(signal.confidence * 100).toFixed(0)}%</span>
        </div>
        <div className="mt-2 h-1 bg-muted rounded-full overflow-hidden">
          <div 
            className={`h-full ${signalColor.replace('text-', 'bg-')}`}
            style={{ width: `${signal.confidence * 100}%` }}
          />
        </div>
      </div>
    </div>
  );
}

function DimensionBar({ dimension, data }: { dimension: string; data: any }) {
  const dimensionLabels: Record<string, string> = {
    on_chain: '链上',
    macro: '宏观',
    technical: '技术面',
    sentiment: '情绪',
  };
  
  const signalColor = {
    bullish: 'bg-green-500',
    bearish: 'bg-red-500',
    neutral: 'bg-yellow-500',
  }[data.signal as string] || 'bg-gray-500';
  
  return (
    <div className="group relative">
      <div className="flex items-center gap-2">
        <span className="text-xs text-muted-foreground w-12">
          {dimensionLabels[dimension] || dimension}
        </span>
        <div className="flex-1 h-1.5 bg-muted rounded-full overflow-hidden">
          <div 
            className={`h-full ${signalColor}`}
            style={{ width: `${data.confidence * 100}%` }}
          />
        </div>
        <span className="text-xs font-medium w-8 text-right">
          {(data.confidence * 100).toFixed(0)}%
        </span>
      </div>
      
      {/* 悬停显示摘要 */}
      <div className="absolute left-0 top-full mt-1 p-2 bg-card border border-border rounded shadow-lg text-xs opacity-0 group-hover:opacity-100 transition-opacity pointer-events-none z-10 w-64">
        {data.summary}
      </div>
    </div>
  );
}
```

### 5.2 图表模式组件 (多图层系统)

创建 `frontend/components/charts/LayeredChart.tsx`:

```typescript
'use client';

import { useEffect, useRef, useState } from 'react';
import { createChart, IChartApi, ISeriesApi, Time } from 'lightweight-charts';
import { Eye, EyeOff } from 'lucide-react';

interface ChartLayer {
  id: string;
  name: string;
  enabled: boolean;
  type: 'candlestick' | 'line' | 'area' | 'histogram' | 'markers';
  data: any[];
  color?: string;
}

interface AIAnnotation {
  time: Time;
  position: 'aboveBar' | 'belowBar';
  shape: 'circle' | 'square' | 'arrowUp' | 'arrowDown';
  color: string;
  text: string;
  reasoning: string;  // AI 推理过程
  confidence: number;
}

export function LayeredChart({ asset, timeframe }: { asset: string; timeframe: string }) {
  const chartContainerRef = useRef<HTMLDivElement>(null);
  const chartRef = useRef<IChartApi | null>(null);
  const [layers, setLayers] = useState<ChartLayer[]>([
    { id: 'price', name: 'K线图', enabled: true, type: 'candlestick', data: [] },
    { id: 'ai_annotations', name: 'AI 标注', enabled: true, type: 'markers', data: [] },
    { id: 'macro_events', name: '宏观事件', enabled: true, type: 'markers', data: [] },
    { id: 'on_chain', name: '链上数据', enabled: false, type: 'line', data: [], color: '#26d9a6' },
    { id: 'anomaly_heatmap', name: '异常热力图', enabled: false, type: 'area', data: [], color: 'rgba(255, 87, 51, 0.3)' },
  ]);
  
  useEffect(() => {
    if (!chartContainerRef.current) return;
    
    // 创建图表
    const chart = createChart(chartContainerRef.current, {
      width: chartContainerRef.current.clientWidth,
      height: 500,
      layout: {
        background: { color: '#0a0e17' },
        textColor: '#d1d4dc',
      },
      grid: {
        vertLines: { color: 'rgba(255, 255, 255, 0.05)' },
        horzLines: { color: 'rgba(255, 255, 255, 0.05)' },
      },
      timeScale: {
        borderColor: 'rgba(255, 255, 255, 0.1)',
      },
    });
    
    chartRef.current = chart;
    
    // 加载数据
    loadChartData();
    
    // 响应式
    const handleResize = () => {
      chart.applyOptions({ width: chartContainerRef.current!.clientWidth });
    };
    window.addEventListener('resize', handleResize);
    
    return () => {
      window.removeEventListener('resize', handleResize);
      chart.remove();
    };
  }, [asset, timeframe]);
  
  // 图层变化时重新渲染
  useEffect(() => {
    renderLayers();
  }, [layers]);
  
  async function loadChartData() {
    // 从 API 获取图表数据
    // 包括: 价格数据, AI 标注, 宏观事件, 链上数据等
    // 实现略
  }
  
  function renderLayers() {
    if (!chartRef.current) return;
    
    // 清除所有系列
    // 然后根据 enabled 状态重新添加
    
    layers.forEach(layer => {
      if (!layer.enabled) return;
      
      switch (layer.type) {
        case 'candlestick':
          const candlestickSeries = chartRef.current!.addCandlestickSeries();
          candlestickSeries.setData(layer.data);
          break;
          
        case 'markers':
          // AI 标注或事件标记
          if (layer.id === 'ai_annotations') {
            renderAIAnnotations(layer.data);
          } else if (layer.id === 'macro_events') {
            renderMacroEvents(layer.data);
          }
          break;
          
        case 'line':
          const lineSeries = chartRef.current!.addLineSeries({
            color: layer.color,
            lineWidth: 2,
          });
          lineSeries.setData(layer.data);
          break;
          
        case 'area':
          const areaSeries = chartRef.current!.addAreaSeries({
            topColor: layer.color,
            bottomColor: 'rgba(0, 0, 0, 0)',
            lineColor: layer.color,
            lineWidth: 1,
          });
          areaSeries.setData(layer.data);
          break;
      }
    });
  }
  
  function renderAIAnnotations(annotations: AIAnnotation[]) {
    // 在图表上标注 AI 分析点
    // 悬停时显示推理过程
  }
  
  function renderMacroEvents(events: any[]) {
    // 在图表上标注宏观事件 (CPI, FOMC 等)
  }
  
  function toggleLayer(layerId: string) {
    setLayers(prev => prev.map(layer => 
      layer.id === layerId ? { ...layer, enabled: !layer.enabled } : layer
    ));
  }
  
  return (
    <div className="space-y-4">
      {/* 图层控制 */}
      <div className="flex gap-2 flex-wrap">
        {layers.map(layer => (
          <button
            key={layer.id}
            onClick={() => toggleLayer(layer.id)}
            className={`px-3 py-1.5 rounded text-sm flex items-center gap-2 transition-colors ${
              layer.enabled 
                ? 'bg-primary/20 text-primary border border-primary/50' 
                : 'bg-muted text-muted-foreground border border-border'
            }`}
          >
            {layer.enabled ? <Eye className="w-4 h-4" /> : <EyeOff className="w-4 h-4" />}
            {layer.name}
          </button>
        ))}
      </div>
      
      {/* 图表容器 */}
      <div ref={chartContainerRef} className="rounded-lg overflow-hidden border border-border" />
    </div>
  );
}
```

### 5.3 数据模式组件

创建 `frontend/components/dashboard/DataView.tsx`:

```typescript
'use client';

import { useState, useEffect } from 'react';
import { api } from '@/lib/api';

interface MarketData {
  asset: string;
  price: number;
  change_24h: number;
  volume_24h: number;
  // ... 更多字段
}

export function DataView({ assets }: { assets: string[] }) {
  const [data, setData] = useState<MarketData[]>([]);
  const [anomalies, setAnomalies] = useState<any[]>([]);
  
  useEffect(() => {
    loadData();
    const interval = setInterval(loadData, 5000); // 每 5 秒刷新
    return () => clearInterval(interval);
  }, [assets]);
  
  async function loadData() {
    const prices = await Promise.all(assets.map(asset => api.getPrice(asset)));
    setData(prices);
    
    const anomalyData = await api.listAnomalies({ limit: 10 });
    setAnomalies(anomalyData);
  }
  
  return (
    <div className="space-y-6">
      {/* 数据表格 */}
      <div className="glass rounded-lg overflow-hidden">
        <table className="w-full text-sm">
          <thead className="bg-muted/50 border-b border-border">
            <tr>
              <th className="text-left p-3 font-medium">资产</th>
              <th className="text-right p-3 font-medium">价格</th>
              <th className="text-right p-3 font-medium">24h 变动</th>
              <th className="text-right p-3 font-medium">24h 成交量</th>
              <th className="text-right p-3 font-medium">状态</th>
            </tr>
          </thead>
          <tbody>
            {data.map((item) => (
              <tr 
                key={item.asset}
                className="border-b border-border hover:bg-muted/30 transition-colors"
              >
                <td className="p-3 font-medium">{item.asset}</td>
                <td className="p-3 text-right font-mono">${item.price.toLocaleString()}</td>
                <td className={`p-3 text-right font-mono ${item.change_24h >= 0 ? 'text-green-500' : 'text-red-500'}`}>
                  {item.change_24h >= 0 ? '+' : ''}{item.change_24h.toFixed(2)}%
                </td>
                <td className="p-3 text-right font-mono">${(item.volume_24h / 1e6).toFixed(2)}M</td>
                <td className="p-3 text-right">
                  {/* 状态指示器 */}
                  <StatusIndicator asset={item.asset} />
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
      
      {/* 异常面板 */}
      <div className="glass rounded-lg p-6">
        <h3 className="text-lg font-semibold mb-4">实时异常</h3>
        <div className="space-y-2">
          {anomalies.map((anomaly) => (
            <div 
              key={anomaly.id}
              className={`p-3 rounded border ${
                anomaly.severity === 'high' ? 'border-red-500/50 bg-red-500/10' :
                anomaly.severity === 'medium' ? 'border-yellow-500/50 bg-yellow-500/10' :
                'border-blue-500/50 bg-blue-500/10'
              } animate-pulse`}
            >
              <div className="flex items-center justify-between">
                <div>
                  <span className="font-medium">{anomaly.asset}</span>
                  <span className="text-muted-foreground text-sm ml-2">{anomaly.anomaly_type}</span>
                </div>
                <span className="text-xs text-muted-foreground">
                  {new Date(anomaly.detected_at).toLocaleTimeString()}
                </span>
              </div>
              <p className="text-sm mt-1">{anomaly.description}</p>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}

function StatusIndicator({ asset }: { asset: string }) {
  // 从 WebSocket 实时接收状态
  return (
    <div className="flex items-center gap-1">
      <div className="w-2 h-2 rounded-full bg-green-500 animate-pulse" />
      <span className="text-xs text-muted-foreground">正常</span>
    </div>
  );
}
```

---

## 阶段 6: Agent 信号 JSON Schema 与集成

### 6.1 Agent 信号标准 JSON Schema

创建 `docs/agent-signal-schema.json`:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "FinVerse Agent Signal",
  "description": "FinVerse 平台的标准化 Agent 信号格式",
  "type": "object",
  "required": ["agent_id", "asset", "signal", "confidence", "dimensions", "timeframe", "timestamp"],
  "properties": {
    "agent_id": {
      "type": "string",
      "format": "uuid",
      "description": "发布信号的 Agent ID"
    },
    "asset": {
      "type": "string",
      "description": "资产标识符 (e.g., BTC/USD, AAPL, EUR/USD)",
      "examples": ["BTC/USD", "AAPL", "EUR/USD", "XAU/USD"]
    },
    "signal": {
      "type": "string",
      "enum": ["bullish", "bearish", "neutral"],
      "description": "整体信号方向"
    },
    "confidence": {
      "type": "number",
      "minimum": 0,
      "maximum": 1,
      "description": "整体置信度 (0-1)"
    },
    "dimensions": {
      "type": "object",
      "description": "多维度分析",
      "properties": {
        "on_chain": {
          "$ref": "#/definitions/dimension"
        },
        "macro": {
          "$ref": "#/definitions/dimension"
        },
        "technical": {
          "$ref": "#/definitions/dimension"
        },
        "sentiment": {
          "$ref": "#/definitions/dimension"
        }
      },
      "additionalProperties": {
        "$ref": "#/definitions/dimension"
      }
    },
    "key_levels": {
      "type": "object",
      "description": "关键价格位",
      "properties": {
        "support": {
          "type": "array",
          "items": { "type": "number" },
          "description": "支撑位数组 (从强到弱)"
        },
        "resistance": {
          "type": "array",
          "items": { "type": "number" },
          "description": "阻力位数组 (从强到弱)"
        }
      }
    },
    "reasoning": {
      "type": "string",
      "description": "AI 推理过程 (可选, 用于展示)"
    },
    "timeframe": {
      "type": "string",
      "description": "信号时间框架",
      "examples": ["1h", "4h", "1d", "1w", "48h"]
    },
    "valid_until": {
      "type": "string",
      "format": "date-time",
      "description": "信号有效期 (可选)"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "信号生成时间 (ISO 8601)"
    },
    "metadata": {
      "type": "object",
      "description": "额外元数据 (可选)",
      "properties": {
        "data_sources": {
          "type": "array",
          "items": { "type": "string" },
          "description": "使用的数据源"
        },
        "model_version": {
          "type": "string",
          "description": "使用的 AI 模型版本"
        }
      }
    }
  },
  "definitions": {
    "dimension": {
      "type": "object",
      "required": ["signal", "confidence", "summary"],
      "properties": {
        "signal": {
          "type": "string",
          "enum": ["bullish", "bearish", "neutral"]
        },
        "confidence": {
          "type": "number",
          "minimum": 0,
          "maximum": 1
        },
        "summary": {
          "type": "string",
          "description": "该维度的简短摘要"
        },
        "data_points": {
          "type": "object",
          "description": "支撑该维度的数据点 (可选)",
          "additionalProperties": true
        }
      }
    }
  }
}
```

### 6.2 Agent 信号发布脚本

创建 Agent 容器内的 `scripts/publish_signal.py`:

```python
import json
import sys
from datetime import datetime, timedelta
import requests

def publish_signal(asset: str, timeframe: str = "24h"):
    """
    生成并发布公域信号
    
    流程:
    1. 调用 AI 分析资产
    2. 生成符合 schema 的 JSON
    3. 调用 API 发布
    """
    
    # 1. 收集数据
    data = collect_market_data(asset)
    
    # 2. AI 分析
    analysis = analyze_with_ai(asset, data, timeframe)
    
    # 3. 构建信号 JSON
    signal = {
        "agent_id": get_agent_id(),
        "asset": asset,
        "signal": analysis["overall_signal"],
        "confidence": analysis["overall_confidence"],
        "dimensions": {
            "on_chain": {
                "signal": analysis["on_chain"]["signal"],
                "confidence": analysis["on_chain"]["confidence"],
                "summary": analysis["on_chain"]["summary"],
            },
            "macro": {
                "signal": analysis["macro"]["signal"],
                "confidence": analysis["macro"]["confidence"],
                "summary": analysis["macro"]["summary"],
            },
            "technical": {
                "signal": analysis["technical"]["signal"],
                "confidence": analysis["technical"]["confidence"],
                "summary": analysis["technical"]["summary"],
            },
            "sentiment": {
                "signal": analysis["sentiment"]["signal"],
                "confidence": analysis["sentiment"]["confidence"],
                "summary": analysis["sentiment"]["summary"],
            },
        },
        "key_levels": analysis.get("key_levels"),
        "reasoning": analysis.get("reasoning"),
        "timeframe": timeframe,
        "valid_until": (datetime.now() + timedelta(hours=24)).isoformat(),
        "timestamp": datetime.now().isoformat(),
    }
    
    # 4. 验证 schema (可选)
    validate_signal_schema(signal)
    
    # 5. 发布到 API
    response = requests.post(
        f"{get_api_url()}/signals",
        json=signal,
        headers={"Authorization": f"Bearer {get_user_token()}"}
    )
    
    if response.status_code == 201:
        print(f"✓ 信号已发布: {asset} - {signal['signal']} ({signal['confidence']:.2f})")
        return response.json()
    else:
        print(f"✗ 发布失败: {response.text}")
        return None

def analyze_with_ai(asset: str, data: dict, timeframe: str) -> dict:
    """
    调用 AI 分析
    
    Prompt 模板:
    ```
    你是一个金融分析 AI。请分析以下数据并输出结构化的信号。
    
    资产: {asset}
    时间框架: {timeframe}
    
    数据:
    - 价格: {data['price']}
    - 24h 变动: {data['change_24h']}%
    - 成交量: {data['volume']}
    - 链上数据: {data['on_chain']}
    - 宏观环境: {data['macro']}
    - 技术指标: {data['technical']}
    - 市场情绪: {data['sentiment']}
    
    请输出 JSON 格式:
    {
      "overall_signal": "bullish/bearish/neutral",
      "overall_confidence": 0.0-1.0,
      "on_chain": {"signal": "...", "confidence": 0.0-1.0, "summary": "..."},
      "macro": {"signal": "...", "confidence": 0.0-1.0, "summary": "..."},
      "technical": {"signal": "...", "confidence": 0.0-1.0, "summary": "..."},
      "sentiment": {"signal": "...", "confidence": 0.0-1.0, "summary": "..."},
      "key_levels": {"support": [67200, 64800], "resistance": [69800]},
      "reasoning": "多步推理过程..."
    }
    ```
    """
    
    # 从 .secrets/keys.enc 读取用户的 AI API key
    api_key = load_encrypted_key("openai")  # 或 anthropic, deepseek
    
    # 调用 AI API (实现略)
    # response = openai.ChatCompletion.create(...)
    
    # 解析 JSON 返回
    return parsed_response

if __name__ == "__main__":
    asset = sys.argv[1] if len(sys.argv) > 1 else "BTC/USD"
    timeframe = sys.argv[2] if len(sys.argv) > 2 else "24h"
    
    publish_signal(asset, timeframe)
```

### 6.3 Agent 初始化配置

在 Agent 创建时,添加定时发布信号的 Cron 任务:

```bash
# 每 4 小时发布一次信号 (用户关注的资产)
0 */4 * * * python /app/scripts/publish_signal.py BTC/USD 4h
0 */4 * * * python /app/scripts/publish_signal.py ETH/USD 4h
```

---

## 阶段 7: OpenClaw 集成配置

### 7.1 Agent Docker 镜像

创建 `docker/agent/Dockerfile`:

```dockerfile
FROM ubuntu:22.04

# 安装基础工具
RUN apt-get update && apt-get install -y \
    curl \
    git \
    python3 \
    python3-pip \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# 安装 OpenClaw
RUN curl -fsSL https://openclaw.com/install.sh | bash

# 安装 Python 依赖
COPY requirements.txt /app/requirements.txt
RUN pip3 install -r /app/requirements.txt

# 复制 Agent 脚本
COPY scripts/ /app/scripts/
COPY config/ /app/config/

# 设置工作目录
WORKDIR /root/.openclaw/workspace

# 启动 OpenClaw
CMD ["openclaw", "gateway", "start", "--foreground"]
```

`requirements.txt`:
```
requests
aiohttp
cryptography
pydantic
jsonschema
python-dotenv
```

### 7.2 Agent 启动脚本

创建 `docker/agent/entrypoint.sh`:

```bash
#!/bin/bash
set -e

# 等待 workspace 挂载
until [ -d /root/.openclaw/workspace ]; do
  echo "Waiting for workspace..."
  sleep 1
done

# 解密 API keys 并设置环境变量
python3 /app/scripts/setup_env.py

# 启动 OpenClaw
openclaw gateway start --foreground
```

### 7.3 Agent 配置文件模板

创建 `docker/agent/config/agent_config.yaml`:

```yaml
# FinVerse Agent 配置模板

agent:
  name: "FinVerse Agent"
  version: "1.0.0"
  
# OpenClaw 设置
openclaw:
  workspace: "/root/.openclaw/workspace"
  model: "{{ user_ai_provider }}"  # 用户选择的 AI 服务商
  
# 数据源
data_sources:
  # 免费数据源 (平台提供)
  - name: coingecko
    type: crypto
    enabled: true
  - name: yahoo_finance
    type: stocks
    enabled: true
    
  # 付费数据源 (用户自己的 key)
  - name: glassnode
    type: on_chain
    enabled: "{{ user_has_glassnode }}"
  - name: polygon
    type: stocks_premium
    enabled: "{{ user_has_polygon }}"

# Cron 任务
cron_jobs:
  - name: morning_summary
    schedule: "0 {{ user_wake_hour }} * * *"
    command: "python /app/scripts/morning_summary.py"
    
  - name: market_scan
    schedule: "0 */4 * * *"
    command: "python /app/scripts/market_scan.py"
    
  - name: evening_recap
    schedule: "0 {{ user_sleep_hour }} * * *"
    command: "python /app/scripts/evening_recap.py"
    
  - name: publish_signals
    schedule: "0 */4 * * *"
    command: |
      {% for asset in user_favorite_assets %}
      python /app/scripts/publish_signal.py {{ asset }} 4h
      {% endfor %}

# 通知设置
notifications:
  channels:
    {% for channel in user_notification_channels %}
    - type: {{ channel.type }}
      config: {{ channel.config }}
    {% endfor %}
  
  quiet_hours:
    enabled: {{ user_quiet_hours_enabled }}
    start: "{{ user_quiet_hours_start }}"
    end: "{{ user_quiet_hours_end }}"

# 记忆设置
memory:
  max_daily_files: 90  # 保留 90 天的日志
  auto_cleanup: true
  backup_interval: "0 2 * * *"  # 每天 2:00 备份
```

---

## 阶段 8: 用户注册流程与引导

### 8.1 注册页面

创建 `frontend/app/(auth)/register/page.tsx`:

```typescript
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { api } from '@/lib/api';

export default function RegisterPage() {
  const router = useRouter();
  const [step, setStep] = useState(1);
  const [formData, setFormData] = useState({
    email: '',
    username: '',
    password: '',
    display_name: '',
    timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
    language: 'zh',
  });
  
  const [apiKeys, setApiKeys] = useState({
    ai_provider: '',
    ai_key: '',
  });
  
  const [preferences, setPreferences] = useState({
    trading_styles: [] as string[],
    analysis_preferences: [] as string[],
    asset_preferences: [] as string[],
    risk_profile: '',
  });
  
  async function handleSubmit() {
    try {
      // 1. 注册用户
      await api.register(formData);
      
      // 2. 登录
      await api.login(formData.username, formData.password);
      
      // 3. 添加 API key
      await api.addAPIKey({
        service_type: apiKeys.ai_provider,
        api_key: apiKeys.ai_key,
      });
      
      // 4. 保存偏好
      await api.updatePreferences(preferences);
      
      // 5. 启动 Agent
      await api.startAgent();
      
      // 6. 跳转到主页
      router.push('/');
      
    } catch (error) {
      console.error('注册失败:', error);
      alert('注册失败,请重试');
    }
  }
  
  return (
    <div className="min-h-screen flex items-center justify-center bg-background p-4">
      <div className="max-w-2xl w-full">
        {/* 进度指示器 */}
        <StepIndicator currentStep={step} totalSteps={4} />
        
        {/* 步骤内容 */}
        {step === 1 && <Step1BasicInfo data={formData} onChange={setFormData} onNext={() => setStep(2)} />}
        {step === 2 && <Step2AIProvider data={apiKeys} onChange={setApiKeys} onNext={() => setStep(3)} onBack={() => setStep(1)} />}
        {step === 3 && <Step3Preferences data={preferences} onChange={setPreferences} onNext={() => setStep(4)} onBack={() => setStep(2)} />}
        {step === 4 && <Step4ChatConnection onFinish={handleSubmit} onBack={() => setStep(3)} />}
      </div>
    </div>
  );
}

function Step2AIProvider({ data, onChange, onNext, onBack }: any) {
  const providers = [
    { id: 'openai', name: 'OpenAI', icon: '/icons/openai.svg', guide: 'https://platform.openai.com/api-keys' },
    { id: 'anthropic', name: 'Anthropic', icon: '/icons/anthropic.svg', guide: 'https://console.anthropic.com/settings/keys' },
    { id: 'deepseek', name: 'DeepSeek', icon: '/icons/deepseek.svg', guide: 'https://platform.deepseek.com/api_keys' },
  ];
  
  return (
    <div className="glass rounded-lg p-8">
      <h2 className="text-2xl font-bold mb-2">选择 AI 服务商</h2>
      <p className="text-muted-foreground mb-6">
        你的 Agent 需要 AI 能力。选择一个服务商并提供 API key。
      </p>
      
      {/* 服务商选择 */}
      <div className="grid grid-cols-3 gap-4 mb-6">
        {providers.map(provider => (
          <button
            key={provider.id}
            onClick={() => onChange({ ...data, ai_provider: provider.id })}
            className={`p-4 rounded-lg border-2 transition-colors ${
              data.ai_provider === provider.id 
                ? 'border-primary bg-primary/10' 
                : 'border-border hover:border-primary/50'
            }`}
          >
            <img src={provider.icon} alt={provider.name} className="w-12 h-12 mx-auto mb-2" />
            <div className="text-sm font-medium">{provider.name}</div>
          </button>
        ))}
      </div>
      
      {/* API key 输入 */}
      {data.ai_provider && (
        <div className="space-y-4">
          <div>
            <label className="block text-sm font-medium mb-2">API Key</label>
            <input
              type="password"
              value={data.ai_key}
              onChange={(e) => onChange({ ...data, ai_key: e.target.value })}
              placeholder="sk-..."
              className="w-full px-4 py-2 bg-input border border-border rounded-lg focus:outline-none focus:ring-2 focus:ring-primary"
            />
          </div>
          
          <div className="bg-muted/50 rounded p-4 text-sm">
            <p className="font-medium mb-2">如何获取 API key?</p>
            <ol className="list-decimal list-inside space-y-1 text-muted-foreground">
              <li>
                访问{' '}
                <a href={providers.find(p => p.id === data.ai_provider)?.guide} target="_blank" className="text-primary hover:underline">
                  {providers.find(p => p.id === data.ai_provider)?.name} 控制台
                </a>
              </li>
              <li>创建新的 API key</li>
              <li>复制并粘贴到上方</li>
            </ol>
          </div>
        </div>
      )}
      
      {/* 按钮 */}
      <div className="flex gap-4 mt-6">
        <button onClick={onBack} className="px-6 py-2 border border-border rounded-lg hover:bg-muted transition-colors">
          上一步
        </button>
        <button 
          onClick={onNext}
          disabled={!data.ai_provider || !data.ai_key}
          className="flex-1 px-6 py-2 bg-primary text-primary-foreground rounded-lg hover:bg-primary/90 transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
        >
          下一步
        </button>
      </div>
    </div>
  );
}

function Step3Preferences({ data, onChange, onNext, onBack }: any) {
  // 游戏化偏好测试
  // 示例: "你在盯盘时,最关注什么?"
  //   A. K线形态和技术指标
  //   B. 链上数据和大户动向
  //   C. 新闻和宏观经济
  //   D. 社交媒体情绪
  
  return (
    <div className="glass rounded-lg p-8">
      <h2 className="text-2xl font-bold mb-2">了解你的偏好</h2>
      <p className="text-muted-foreground mb-6">
        回答几个问题,帮助 Agent 更了解你 (2 分钟)
      </p>
      
      {/* 问题 1: 交易风格 */}
      <QuizQuestion
        question="你的交易周期是?"
        options={[
          { value: 'day_trading', label: '日内交易 (当天买卖)' },
          { value: 'swing', label: '短线 (1-5天)' },
          { value: 'position', label: '波段 (周-月)' },
          { value: 'long_term', label: '长线投资 (月-年)' },
        ]}
        selected={data.trading_styles}
        onChange={(values) => onChange({ ...data, trading_styles: values })}
        multiple
      />
      
      {/* 问题 2: 分析偏好 */}
      <QuizQuestion
        question="你最关注哪些分析维度?"
        options={[
          { value: 'technical', label: '技术面 (K线,指标)' },
          { value: 'on_chain', label: '链上数据 (仅加密货币)' },
          { value: 'macro', label: '宏观经济 (利率,CPI)' },
          { value: 'sentiment', label: '市场情绪' },
        ]}
        selected={data.analysis_preferences}
        onChange={(values) => onChange({ ...data, analysis_preferences: values })}
        multiple
      />
      
      {/* 问题 3: 品种偏好 */}
      <QuizQuestion
        question="你主要交易什么?"
        options={[
          { value: 'crypto', label: '加密货币' },
          { value: 'stocks', label: '美股' },
          { value: 'forex', label: '外汇' },
          { value: 'commodities', label: '贵金属/大宗商品' },
        ]}
        selected={data.asset_preferences}
        onChange={(values) => onChange({ ...data, asset_preferences: values })}
        multiple
      />
      
      {/* 问题 4: 风险偏好 */}
      <QuizQuestion
        question="你的风险偏好是?"
        options={[
          { value: 'conservative', label: '保守 (稳字当头)' },
          { value: 'balanced', label: '稳健 (风险收益平衡)' },
          { value: 'aggressive', label: '激进 (高风险高收益)' },
        ]}
        selected={data.risk_profile ? [data.risk_profile] : []}
        onChange={(values) => onChange({ ...data, risk_profile: values[0] })}
      />
      
      {/* 按钮 */}
      <div className="flex gap-4 mt-6">
        <button onClick={onBack} className="px-6 py-2 border border-border rounded-lg hover:bg-muted transition-colors">
          上一步
        </button>
        <button 
          onClick={onNext}
          className="flex-1 px-6 py-2 bg-primary text-primary-foreground rounded-lg hover:bg-primary/90 transition-colors"
        >
          下一步
        </button>
      </div>
    </div>
  );
}
```

---

## 阶段 9: WebSocket 实时通信

### 9.1 后端 WebSocket 服务器

创建 `backend/websocket/server.py`:

```python
from fastapi import WebSocket, WebSocketDisconnect, Depends
from typing import Dict, Set
import json
import asyncio

class ConnectionManager:
    def __init__(self):
        self.active_connections: Dict[str, Set[WebSocket]] = {}  # user_id -> {websockets}
        
    async def connect(self, websocket: WebSocket, user_id: str):
        await websocket.accept()
        if user_id not in self.active_connections:
            self.active_connections[user_id] = set()
        self.active_connections[user_id].add(websocket)
        
    def disconnect(self, websocket: WebSocket, user_id: str):
        if user_id in self.active_connections:
            self.active_connections[user_id].discard(websocket)
            if not self.active_connections[user_id]:
                del self.active_connections[user_id]
                
    async def send_to_user(self, user_id: str, message: dict):
        """向特定用户的所有连接发送消息"""
        if user_id in self.active_connections:
            dead_connections = set()
            for connection in self.active_connections[user_id]:
                try:
                    await connection.send_json(message)
                except:
                    dead_connections.add(connection)
            
            # 清理断开的连接
            for conn in dead_connections:
                self.disconnect(conn, user_id)
                
    async def broadcast(self, message: dict):
        """广播给所有用户"""
        for user_id in self.active_connections:
            await self.send_to_user(user_id, message)

manager = ConnectionManager()

@app.websocket("/ws")
async def websocket_endpoint(
    websocket: WebSocket,
    token: str,  # 从查询参数获取
):
    # 验证 token
    user = verify_token(token)
    if not user:
        await websocket.close(code=4001)
        return
        
    await manager.connect(websocket, user["id"])
    
    try:
        # 发送欢迎消息
        await websocket.send_json({
            "type": "connected",
            "message": "WebSocket connected"
        })
        
        # 订阅 Redis pub/sub 频道
        pubsub = redis_client.pubsub()
        pubsub.subscribe(
            "channel:signals",
            "channel:anomalies",
            f"channel:user:{user['id']}"
        )
        
        # 监听消息
        async for message in pubsub.listen():
            if message["type"] == "message":
                data = json.loads(message["data"])
                await websocket.send_json(data)
                
    except WebSocketDisconnect:
        manager.disconnect(websocket, user["id"])
```

### 9.2 前端 WebSocket Hook

创建 `frontend/hooks/useWebSocket.ts`:

```typescript
import { useEffect, useCallback } from 'react';
import { ws }