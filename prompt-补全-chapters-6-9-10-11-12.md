# FinVerse 开发提示词 - 遗漏章节补全

> 补全 Part A/B/C 中遗漏或不完整的章节
> 生成时间：2026-02-08

---

## 章节六：公域（信号系统）- 补全部分

### 6.1 信号池数据库完整设计

#### 6.1.1 信号表（扩展版）

```sql
-- 公域信号表（扩展 Part C 的基础版本）
CREATE TABLE public_signals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    asset VARCHAR(20) NOT NULL,
    signal VARCHAR(10) NOT NULL CHECK (signal IN ('bullish', 'bearish', 'neutral')),
    confidence NUMERIC(3, 2) NOT NULL CHECK (confidence >= 0 AND confidence <= 1),
    
    -- 多维分析
    dimensions JSONB NOT NULL,
    /* 格式:
    {
        "on_chain": {"signal": "bearish", "confidence": 0.78, "summary": "...", "key_metrics": [...]},
        "technical": {...},
        "macro": {...},
        "sentiment": {...}
    }
    */
    
    -- 关键位
    key_levels JSONB,
    /* {"support": [67200, 64800], "resistance": [69800]} */
    
    -- 时间框架
    timeframe VARCHAR(10) NOT NULL, -- '1h', '4h', '24h', '48h', '1w'
    
    -- 推理过程
    reasoning TEXT NOT NULL,
    
    -- 数据来源
    data_sources JSONB,
    /* ["CoinGecko", "Glassnode", "TradingView"] */
    
    -- 验证状态（事后验证）
    verified BOOLEAN DEFAULT FALSE,
    actual_outcome VARCHAR(10), -- 实际走势：'bullish'/'bearish'/'neutral'
    accuracy_score NUMERIC(3, 2), -- 0-1，准确度评分
    verified_at TIMESTAMP,
    
    -- 订阅相关
    view_count INTEGER DEFAULT 0,
    subscriber_count INTEGER DEFAULT 0, -- 有多少 Agent 订阅了这个信号
    
    -- 时间戳
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP, -- 信号过期时间（基于 timeframe 计算）
    
    -- 索引
    CONSTRAINT valid_timeframe CHECK (timeframe IN ('1h', '4h', '24h', '48h', '1w', '1m'))
);

CREATE INDEX idx_public_signals_asset_time ON public_signals(asset, created_at DESC);
CREATE INDEX idx_public_signals_agent ON public_signals(agent_id);
CREATE INDEX idx_public_signals_expires ON public_signals(expires_at) WHERE expires_at IS NOT NULL;
CREATE INDEX idx_public_signals_verified ON public_signals(verified, asset);

-- 信号订阅表
CREATE TABLE signal_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_agent_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- 订阅条件
    assets TEXT[] DEFAULT '{}', -- ['BTC/USD', 'ETH/USD'] 或 空数组 = 订阅所有
    min_confidence NUMERIC(3, 2) DEFAULT 0.6, -- 最低置信度阈值
    timeframes TEXT[] DEFAULT '{"24h", "48h"}', -- 关注的时间框架
    
    -- 订阅的 Agent（可选，空 = 订阅所有）
    publisher_agent_ids UUID[] DEFAULT NULL,
    
    -- 订阅的信誉等级（可选）
    min_reputation_tier VARCHAR(20), -- 'reliable', 'expert', 'master'
    
    -- 通知设置
    notify_on_publish BOOLEAN DEFAULT TRUE,
    notify_on_consensus_change BOOLEAN DEFAULT TRUE, -- 共识变化时通知
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(subscriber_agent_id) -- 每个 Agent 只有一个订阅配置
);

CREATE INDEX idx_subscriptions_assets ON signal_subscriptions USING GIN(assets);

-- 信号交互表（Agent 对信号的反馈）
CREATE TABLE signal_interactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    signal_id UUID NOT NULL REFERENCES public_signals(id) ON DELETE CASCADE,
    agent_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    interaction_type VARCHAR(20) NOT NULL, -- 'view', 'agree', 'disagree', 'use_in_analysis'
    
    -- 如果是 agree/disagree，可以附加自己的观点
    own_signal VARCHAR(10), -- 'bullish', 'bearish', 'neutral'
    own_confidence NUMERIC(3, 2),
    comment TEXT,
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT valid_interaction CHECK (interaction_type IN ('view', 'agree', 'disagree', 'use_in_analysis'))
);

CREATE INDEX idx_interactions_signal ON signal_interactions(signal_id);
CREATE INDEX idx_interactions_agent ON signal_interactions(agent_id);

-- 共识快照表（每小时聚合一次）
CREATE TABLE signal_consensus_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset VARCHAR(20) NOT NULL,
    timeframe VARCHAR(10) NOT NULL,
    snapshot_time TIMESTAMP NOT NULL,
    
    -- 统计数据
    total_signals INTEGER NOT NULL,
    bullish_count INTEGER NOT NULL,
    bearish_count INTEGER NOT NULL,
    neutral_count INTEGER NOT NULL,
    
    -- 加权共识（按信誉评分加权）
    weighted_consensus VARCHAR(10) NOT NULL, -- 'bullish', 'bearish', 'neutral'
    weighted_confidence NUMERIC(3, 2) NOT NULL,
    
    -- 分位数数据
    confidence_percentiles JSONB, -- {"p25": 0.65, "p50": 0.72, "p75": 0.85}
    
    -- 分歧度（标准差）
    divergence_score NUMERIC(4, 3), -- 0-1，越高越分歧
    
    -- 参与 Agent 信誉分布
    reputation_distribution JSONB,
    /* {"novice": 5, "reliable": 12, "expert": 8, "master": 3} */
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(asset, timeframe, snapshot_time)
);

CREATE INDEX idx_consensus_snapshots_asset_time ON signal_consensus_snapshots(asset, snapshot_time DESC);
```

---

### 6.2 信号聚合算法实现

**创建文件：`packages/signal-aggregator/src/aggregator.ts`**

```typescript
import { db } from '@finverse/database';

/**
 * 信号聚合器
 * 
 * 职责：
 * - 从公域信号池中聚合多个 Agent 的信号
 * - 按资产和时间框架分组
 * - 计算加权共识（基于信誉评分）
 * - 生成共识热力图数据
 */
export class SignalAggregator {
  
  /**
   * 聚合指定资产的信号
   */
  async aggregateSignals(
    asset: string,
    timeframe: string,
    since: Date = new Date(Date.now() - 3600000) // 默认最近 1 小时
  ): Promise<AggregatedSignal> {
    // 1. 获取时间范围内的所有信号
    const signals = await db.query<Signal>(
      `SELECT s.*, r.total_score as agent_reputation
       FROM public_signals s
       LEFT JOIN reputation_scores r ON s.agent_id = r.agent_id
       WHERE s.asset = $1 
         AND s.timeframe = $2
         AND s.created_at >= $3
         AND s.expires_at > NOW()
       ORDER BY s.created_at DESC`,
      [asset, timeframe, since]
    );

    if (signals.length === 0) {
      return {
        asset,
        timeframe,
        total_signals: 0,
        consensus: 'neutral',
        confidence: 0.5,
        divergence: 0,
        signals: []
      };
    }

    // 2. 统计基础分布
    const distribution = {
      bullish: signals.filter(s => s.signal === 'bullish').length,
      bearish: signals.filter(s => s.signal === 'bearish').length,
      neutral: signals.filter(s => s.signal === 'neutral').length
    };

    // 3. 计算加权共识（按信誉评分加权）
    let weightedBullish = 0;
    let weightedBearish = 0;
    let weightedNeutral = 0;
    let totalWeight = 0;

    signals.forEach(signal => {
      // 信誉评分作为权重（0-100 -> 0-1）
      const reputationWeight = (signal.agent_reputation || 50) / 100;
      
      // 置信度也作为权重
      const confidenceWeight = signal.confidence;
      
      // 综合权重
      const weight = reputationWeight * confidenceWeight;
      totalWeight += weight;

      if (signal.signal === 'bullish') {
        weightedBullish += weight;
      } else if (signal.signal === 'bearish') {
        weightedBearish += weight;
      } else {
        weightedNeutral += weight;
      }
    });

    // 归一化
    weightedBullish /= totalWeight;
    weightedBearish /= totalWeight;
    weightedNeutral /= totalWeight;

    // 4. 确定加权共识
    let consensus: 'bullish' | 'bearish' | 'neutral';
    let confidence: number;

    if (weightedBullish > 0.6) {
      consensus = 'bullish';
      confidence = weightedBullish;
    } else if (weightedBearish > 0.6) {
      consensus = 'bearish';
      confidence = weightedBearish;
    } else {
      consensus = 'neutral';
      confidence = 1 - Math.abs(weightedBullish - weightedBearish); // 越接近说明越中性
    }

    // 5. 计算分歧度（使用标准差）
    const confidences = signals.map(s => s.confidence);
    const mean = confidences.reduce((a, b) => a + b, 0) / confidences.length;
    const variance = confidences.reduce((sum, c) => sum + Math.pow(c - mean, 2), 0) / confidences.length;
    const divergence = Math.sqrt(variance);

    // 6. 计算信誉分布
    const reputationDistribution = signals.reduce((acc, signal) => {
      const tier = this.getReputationTier(signal.agent_reputation || 50);
      acc[tier] = (acc[tier] || 0) + 1;
      return acc;
    }, {} as Record<string, number>);

    return {
      asset,
      timeframe,
      total_signals: signals.length,
      distribution,
      consensus,
      confidence,
      divergence,
      reputation_distribution: reputationDistribution,
      signals: signals.slice(0, 20), // 返回前 20 个最新信号
      aggregated_at: new Date()
    };
  }

  /**
   * 生成共识热力图数据
   */
  async generateConsensusHeatmap(
    assets: string[],
    timeframe: string = '24h'
  ): Promise<HeatmapData> {
    const heatmapData: HeatmapData = {
      assets: [],
      timeframe,
      generated_at: new Date()
    };

    for (const asset of assets) {
      const aggregated = await this.aggregateSignals(asset, timeframe);
      
      heatmapData.assets.push({
        asset,
        consensus: aggregated.consensus,
        confidence: aggregated.confidence,
        signal_count: aggregated.total_signals,
        divergence: aggregated.divergence,
        color: this.getHeatmapColor(aggregated.consensus, aggregated.confidence)
      });
    }

    return heatmapData;
  }

  /**
   * 保存共识快照（定时任务，每小时执行）
   */
  async saveConsensusSnapshot(): Promise<void> {
    const assets = ['BTC/USD', 'ETH/USD', 'SOL/USD', 'AAPL', 'TSLA']; // 主要资产
    const timeframes = ['24h', '48h', '1w'];

    for (const asset of assets) {
      for (const timeframe of timeframes) {
        const aggregated = await this.aggregateSignals(asset, timeframe);

        await db.query(
          `INSERT INTO signal_consensus_snapshots 
           (asset, timeframe, snapshot_time, total_signals, bullish_count, bearish_count, neutral_count,
            weighted_consensus, weighted_confidence, divergence_score, reputation_distribution)
           VALUES ($1, $2, NOW(), $3, $4, $5, $6, $7, $8, $9, $10)
           ON CONFLICT (asset, timeframe, snapshot_time) DO UPDATE SET
             total_signals = EXCLUDED.total_signals,
             weighted_consensus = EXCLUDED.weighted_consensus,
             weighted_confidence = EXCLUDED.weighted_confidence`,
          [
            asset,
            timeframe,
            aggregated.total_signals,
            aggregated.distribution.bullish,
            aggregated.distribution.bearish,
            aggregated.distribution.neutral,
            aggregated.consensus,
            aggregated.confidence,
            aggregated.divergence,
            JSON.stringify(aggregated.reputation_distribution)
          ]
        );
      }
    }
  }

  /**
   * Agent 订阅信号推送
   */
  async pushSignalsToSubscribers(signal: Signal): Promise<void> {
    // 查找符合条件的订阅者
    const subscribers = await db.query<Subscription>(
      `SELECT * FROM signal_subscriptions
       WHERE (assets = '{}' OR $1 = ANY(assets))
         AND (timeframes = '{}' OR $2 = ANY(timeframes))
         AND min_confidence <= $3
         AND (publisher_agent_ids IS NULL OR $4 = ANY(publisher_agent_ids))
         AND notify_on_publish = true`,
      [signal.asset, signal.timeframe, signal.confidence, signal.agent_id]
    );

    // 推送给每个订阅者的 Agent
    for (const subscriber of subscribers) {
      await this.notifyAgent(subscriber.subscriber_agent_id, {
        type: 'new_signal',
        signal_id: signal.id,
        signal: {
          asset: signal.asset,
          consensus: signal.signal,
          confidence: signal.confidence,
          timeframe: signal.timeframe,
          summary: signal.reasoning.substring(0, 200)
        }
      });
    }
  }

  private getReputationTier(score: number): string {
    if (score >= 80) return 'master';
    if (score >= 65) return 'expert';
    if (score >= 50) return 'reliable';
    return 'novice';
  }

  private getHeatmapColor(consensus: string, confidence: number): string {
    if (consensus === 'bullish') {
      return confidence > 0.8 ? '#16a34a' : confidence > 0.6 ? '#22c55e' : '#86efac';
    } else if (consensus === 'bearish') {
      return confidence > 0.8 ? '#dc2626' : confidence > 0.6 ? '#ef4444' : '#fca5a5';
    } else {
      return '#9ca3af';
    }
  }

  private async notifyAgent(agentId: string, notification: any): Promise<void> {
    // 调用 Agent 通知 API
    // 实现省略
  }
}

interface Signal {
  id: string;
  agent_id: string;
  asset: string;
  signal: 'bullish' | 'bearish' | 'neutral';
  confidence: number;
  dimensions: any;
  key_levels: any;
  timeframe: string;
  reasoning: string;
  agent_reputation: number;
  created_at: Date;
}

interface AggregatedSignal {
  asset: string;
  timeframe: string;
  total_signals: number;
  distribution?: {
    bullish: number;
    bearish: number;
    neutral: number;
  };
  consensus: 'bullish' | 'bearish' | 'neutral';
  confidence: number;
  divergence: number;
  reputation_distribution?: Record<string, number>;
  signals: Signal[];
  aggregated_at: Date;
}

interface HeatmapData {
  assets: Array<{
    asset: string;
    consensus: string;
    confidence: number;
    signal_count: number;
    divergence: number;
    color: string;
  }>;
  timeframe: string;
  generated_at: Date;
}

interface Subscription {
  subscriber_agent_id: string;
  assets: string[];
  timeframes: string[];
  min_confidence: number;
  publisher_agent_ids: string[] | null;
}
```

---

### 6.3 共识热力图前端组件

**创建文件：`components/signals/ConsensusHeatmap.tsx`**

```typescript
import React, { useEffect, useState } from 'react';
import { motion } from 'framer-motion';

interface HeatmapData {
  assets: Array<{
    asset: string;
    consensus: 'bullish' | 'bearish' | 'neutral';
    confidence: number;
    signal_count: number;
    divergence: number;
    color: string;
  }>;
  timeframe: string;
}

export const ConsensusHeatmap: React.FC<{
  timeframe?: string;
}> = ({ timeframe = '24h' }) => {
  const [heatmapData, setHeatmapData] = useState<HeatmapData | null>(null);

  useEffect(() => {
    loadHeatmapData();
  }, [timeframe]);

  async function loadHeatmapData() {
    const response = await fetch(`/api/signals/consensus-heatmap?timeframe=${timeframe}`);
    const data = await response.json();
    setHeatmapData(data);
  }

  if (!heatmapData) return <div>加载中...</div>;

  return (
    <div className="bg-white rounded-xl p-6 shadow-lg">
      <div className="flex justify-between items-center mb-6">
        <h2 className="text-2xl font-bold">社区共识热力图</h2>
        <select
          value={timeframe}
          onChange={(e) => setTimeframe(e.target.value)}
          className="px-4 py-2 border rounded-lg"
        >
          <option value="1h">1小时</option>
          <option value="4h">4小时</option>
          <option value="24h">24小时</option>
          <option value="48h">48小时</option>
          <option value="1w">1周</option>
        </select>
      </div>

      <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-5 gap-4">
        {heatmapData.assets.map((assetData, index) => (
          <motion.div
            key={assetData.asset}
            initial={{ opacity: 0, scale: 0.9 }}
            animate={{ opacity: 1, scale: 1 }}
            transition={{ delay: index * 0.05 }}
            whileHover={{ scale: 1.05 }}
            className="rounded-lg p-4 cursor-pointer transition-all"
            style={{
              backgroundColor: assetData.color,
              color: assetData.consensus === 'neutral' ? '#1f2937' : 'white'
            }}
          >
            {/* 资产名称 */}
            <div className="font-bold text-lg mb-2">{assetData.asset}</div>

            {/* 共识 */}
            <div className="text-sm mb-1">
              {assetData.consensus === 'bullish' ? '看多' : assetData.consensus === 'bearish' ? '看空' : '中性'}
            </div>

            {/* 置信度 */}
            <div className="text-2xl font-bold mb-1">
              {Math.round(assetData.confidence * 100)}%
            </div>

            {/* 信号数量 */}
            <div className="text-xs opacity-80">
              {assetData.signal_count} 个信号
            </div>

            {/* 分歧度指示器 */}
            {assetData.divergence > 0.3 && (
              <div className="mt-2 text-xs flex items-center gap-1">
                <span>⚠️</span>
                <span>分歧较大</span>
              </div>
            )}
          </motion.div>
        ))}
      </div>

      {/* 图例 */}
      <div className="mt-6 flex items-center justify-center gap-6 text-sm">
        <div className="flex items-center gap-2">
          <div className="w-4 h-4 rounded" style={{ backgroundColor: '#16a34a' }} />
          <span>强烈看多</span>
        </div>
        <div className="flex items-center gap-2">
          <div className="w-4 h-4 rounded" style={{ backgroundColor: '#9ca3af' }} />
          <span>中性</span>
        </div>
        <div className="flex items-center gap-2">
          <div className="w-4 h-4 rounded" style={{ backgroundColor: '#dc2626' }} />
          <span>强烈看空</span>
        </div>
      </div>
    </div>
  );
};
```

---

## 章节九：可视化 - 补全部分

### 9.2 图表模式 - 图层系统完整实现

**继续 Part B 被截断的部分，补全图层系统**

```typescript
// ChartMode.tsx (续)

const layerNames = {
  price: '价格',
  volume: '成交量',
  ai_annotations: 'AI标注',
  support_resistance: '支撑阻力',
  macro_events: '宏观事件',
  on_chain: '链上数据',
  heatmap: '异常热力图',
  historical_overlay: '历史叠影'
};

/**
 * 渲染所有图层
 */
async function renderLayers(
  chart: IChartApi,
  data: ChartData,
  enabledLayers: Record<string, boolean>
) {
  // 清除所有现有系列
  chart.timeScale().fitContent();

  // 1. 价格（K线图）
  if (enabledLayers.price) {
    const candlestickSeries = chart.addCandlestickSeries({
      upColor: '#22c55e',
      downColor: '#ef4444',
      borderVisible: false,
      wickUpColor: '#22c55e',
      wickDownColor: '#ef4444'
    });

    candlestickSeries.setData(data.ohlc);
  }

  // 2. 成交量
  if (enabledLayers.volume) {
    const volumeSeries = chart.addHistogramSeries({
      color: '#60a5fa',
      priceFormat: {
        type: 'volume',
      },
      priceScaleId: 'volume',
      scaleMargins: {
        top: 0.8,
        bottom: 0,
      },
    });

    volumeSeries.setData(data.volume);
  }

  // 3. AI 标注
  if (enabledLayers.ai_annotations && data.ai_annotations) {
    data.ai_annotations.forEach(annotation => {
      if (annotation.type === 'horizontal_line') {
        // 支撑/阻力线
        const priceLine = candlestickSeries.createPriceLine({
          price: annotation.price,
          color: annotation.color,
          lineWidth: 2,
          lineStyle: 2, // 虚线
          axisLabelVisible: true,
          title: annotation.label
        });
      } else if (annotation.type === 'marker') {
        // 点标注
        candlestickSeries.setMarkers([
          {
            time: annotation.time,
            position: annotation.position, // 'aboveBar' | 'belowBar'
            color: annotation.color,
            shape: 'circle',
            text: annotation.text,
            size: 1
          }
        ]);
      }
    });
  }

  // 4. 宏观事件标记
  if (enabledLayers.macro_events && data.macro_events) {
    const markers = data.macro_events.map(event => ({
      time: event.timestamp,
      position: 'aboveBar' as const,
      color: '#f59e0b',
      shape: 'circle' as const,
      text: event.name,
      size: 1
    }));

    candlestickSeries.setMarkers(markers);
  }

  // 5. 链上数据（叠加在主图上）
  if (enabledLayers.on_chain && data.on_chain) {
    const onChainSeries = chart.addLineSeries({
      color: '#8b5cf6',
      lineWidth: 2,
      priceScaleId: 'onchain',
      scaleMargins: {
        top: 0.1,
        bottom: 0.7,
      },
    });

    onChainSeries.setData(data.on_chain);
  }

  // 6. 异常热力图（背景渐变）
  if (enabledLayers.heatmap && data.anomalies) {
    // 使用 Canvas overlay 实现热力图
    renderHeatmapOverlay(chart, data.anomalies);
  }

  // 7. 历史叠影
  if (enabledLayers.historical_overlay && data.historical_pattern) {
    const historicalSeries = chart.addLineSeries({
      color: '#9ca3af',
      lineWidth: 1,
      lineStyle: 2,
      priceScaleId: 'right',
      lastValueVisible: false,
      priceLineVisible: false,
      opacity: 0.5
    });

    historicalSeries.setData(data.historical_pattern);
  }
}

/**
 * 热力图叠加层
 */
function renderHeatmapOverlay(
  chart: IChartApi,
  anomalies: Array<{ time: number; severity: number }>
) {
  const container = chart.chartElement();
  const canvas = document.createElement('canvas');
  canvas.style.position = 'absolute';
  canvas.style.top = '0';
  canvas.style.left = '0';
  canvas.style.pointerEvents = 'none';
  canvas.width = container.clientWidth;
  canvas.height = container.clientHeight;
  
  container.appendChild(canvas);

  const ctx = canvas.getContext('2d')!;

  anomalies.forEach(anomaly => {
    const x = chart.timeScale().timeToCoordinate(anomaly.time);
    if (!x) return;

    const gradient = ctx.createRadialGradient(x, canvas.height / 2, 0, x, canvas.height / 2, 50);
    gradient.addColorStop(0, `rgba(239, 68, 68, ${anomaly.severity})`);
    gradient.addColorStop(1, 'rgba(239, 68, 68, 0)');

    ctx.fillStyle = gradient;
    ctx.fillRect(x - 50, 0, 100, canvas.height);
  });
}
```

---

### 9.3 数据模式（完整实现）

**创建文件：`components/visualization/DataMode.tsx`**

```typescript
import React, { useEffect, useState } from 'react';
import { motion } from 'framer-motion';

interface DataModeProps {
  asset: string;
}

interface MarketData {
  price: {
    current: number;
    open: number;
    high: number;
    low: number;
    change_pct: number;
    change_24h: number;
  };
  volume: {
    current: number;
    avg_24h: number;
    change_pct: number;
  };
  onchain?: {
    exchange_inflow: number;
    exchange_outflow: number;
    active_addresses: number;
    whale_transactions: number;
  };
  derivatives?: {
    open_interest: number;
    funding_rate: number;
    long_short_ratio: number;
  };
  indicators: {
    rsi: number;
    macd: { value: number; signal: number; histogram: number };
    bollinger: { upper: number; middle: number; lower: number };
    ema_20: number;
    ema_50: number;
    ema_200: number;
  };
  sentiment: {
    fear_greed_index: number;
    social_volume: number;
    social_sentiment: number;
  };
}

export const DataMode: React.FC<DataModeProps> = ({ asset }) => {
  const [data, setData] = useState<MarketData | null>(null);
  const [blinkingFields, setBlinkingFields] = useState<Set<string>>(new Set());

  useEffect(() => {
    loadData();
    
    // WebSocket 连接实时更新
    const ws = new WebSocket(`wss://api.finverse.ai/ws/realtime?asset=${asset}`);
    
    ws.onmessage = (event) => {
      const newData = JSON.parse(event.data);
      
      // 检测变化的字段并闪烁
      if (data) {
        const changedFields = detectChangedFields(data, newData);
        setBlinkingFields(changedFields);
        setTimeout(() => setBlinkingFields(new Set()), 500); // 闪烁 0.5 秒
      }
      
      setData(newData);
    };

    return () => ws.close();
  }, [asset]);

  async function loadData() {
    const response = await fetch(`/api/market-data/${asset}/full`);
    const data = await response.json();
    setData(data);
  }

  if (!data) return <div>加载中...</div>;

  return (
    <div className="min-h-screen bg-gray-900 text-gray-100 p-6">
      <div className="max-w-7xl mx-auto">
        {/* Header */}
        <div className="mb-6 flex items-center justify-between">
          <h1 className="text-3xl font-bold">{asset} 数据面板</h1>
          <div className="text-sm text-gray-400">
            实时更新 · 最后更新: {new Date().toLocaleTimeString()}
          </div>
        </div>

        {/* Price Section */}
        <DataSection title="价格" color="#6366f1">
          <DataGrid>
            <DataCell
              label="当前价格"
              value={formatCurrency(data.price.current)}
              isBlinking={blinkingFields.has('price.current')}
              size="large"
            />
            <DataCell label="开盘价" value={formatCurrency(data.price.open)} />
            <DataCell label="最高价" value={formatCurrency(data.price.high)} color="#22c55e" />
            <DataCell label="最低价" value={formatCurrency(data.price.low)} color="#ef4444" />
            <DataCell
              label="24h 涨跌"
              value={formatPercentage(data.price.change_24h)}
              color={data.price.change_24h >= 0 ? '#22c55e' : '#ef4444'}
              isBlinking={blinkingFields.has('price.change_24h')}
            />
          </DataGrid>
        </DataSection>

        {/* Volume Section */}
        <DataSection title="成交量" color="#8b5cf6">
          <DataGrid>
            <DataCell
              label="当前成交量"
              value={formatVolume(data.volume.current)}
              isBlinking={blinkingFields.has('volume.current')}
            />
            <DataCell label="24h 平均" value={formatVolume(data.volume.avg_24h)} />
            <DataCell
              label="相对平均"
              value={formatPercentage(data.volume.change_pct)}
              color={data.volume.change_pct > 50 ? '#f59e0b' : '#6b7280'}
            />
          </DataGrid>
        </DataSection>

        {/* On-Chain Section (仅加密货币) */}
        {data.onchain && (
          <DataSection title="链上数据" color="#10b981">
            <DataGrid>
              <DataCell
                label="交易所流入"
                value={`${data.onchain.exchange_inflow.toLocaleString()} BTC`}
                isBlinking={blinkingFields.has('onchain.exchange_inflow')}
              />
              <DataCell
                label="交易所流出"
                value={`${data.onchain.exchange_outflow.toLocaleString()} BTC`}
              />
              <DataCell
                label="净流入"
                value={`${(data.onchain.exchange_inflow - data.onchain.exchange_outflow).toLocaleString()} BTC`}
                color={
                  data.onchain.exchange_inflow > data.onchain.exchange_outflow
                    ? '#ef4444' // 流入多 = 抛压
                    : '#22c55e' // 流出多 = 看好
                }
              />
              <DataCell
                label="活跃地址"
                value={data.onchain.active_addresses.toLocaleString()}
              />
              <DataCell
                label="大额转账"
                value={data.onchain.whale_transactions.toString()}
                color={data.onchain.whale_transactions > 10 ? '#f59e0b' : '#6b7280'}
              />
            </DataGrid>
          </DataSection>
        )}

        {/* Derivatives Section */}
        {data.derivatives && (
          <DataSection title="衍生品数据" color="#f59e0b">
            <DataGrid>
              <DataCell
                label="持仓量"
                value={formatVolume(data.derivatives.open_interest)}
              />
              <DataCell
                label="资金费率"
                value={formatPercentage(data.derivatives.funding_rate * 100)}
                color={data.derivatives.funding_rate > 0.01 ? '#22c55e' : data.derivatives.funding_rate < -0.01 ? '#ef4444' : '#6b7280'}
              />
              <DataCell
                label="多空比"
                value={data.derivatives.long_short_ratio.toFixed(2)}
                color={data.derivatives.long_short_ratio > 1 ? '#22c55e' : '#ef4444'}
              />
            </DataGrid>
          </DataSection>
        )}

        {/* Technical Indicators */}
        <DataSection title="技术指标" color="#ec4899">
          <DataGrid>
            <DataCell
              label="RSI (14)"
              value={data.indicators.rsi.toFixed(2)}
              color={
                data.indicators.rsi > 70 ? '#ef4444' : 
                data.indicators.rsi < 30 ? '#22c55e' : 
                '#6b7280'
              }
            />
            <DataCell
              label="MACD"
              value={data.indicators.macd.value.toFixed(2)}
              color={data.indicators.macd.histogram > 0 ? '#22c55e' : '#ef4444'}
            />
            <DataCell
              label="MACD Signal"
              value={data.indicators.macd.signal.toFixed(2)}
            />
            <DataCell
              label="布林带上轨"
              value={formatCurrency(data.indicators.bollinger.upper)}
            />
            <DataCell
              label="布林带中轨"
              value={formatCurrency(data.indicators.bollinger.middle)}
            />
            <DataCell
              label="布林带下轨"
              value={formatCurrency(data.indicators.bollinger.lower)}
            />
            <DataCell
              label="EMA 20"
              value={formatCurrency(data.indicators.ema_20)}
            />
            <DataCell
              label="EMA 50"
              value={formatCurrency(data.indicators.ema_50)}
            />
            <DataCell
              label="EMA 200"
              value={formatCurrency(data.indicators.ema_200)}
            />
          </DataGrid>
        </DataSection>

        {/* Sentiment */}
        <DataSection title="市场情绪" color="#06b6d4">
          <DataGrid>
            <DataCell
              label="恐惧贪婪指数"
              value={data.sentiment.fear_greed_index.toString()}
              color={
                data.sentiment.fear_greed_index > 75 ? '#ef4444' :
                data.sentiment.fear_greed_index < 25 ? '#22c55e' :
                '#6b7280'
              }
            />
            <DataCell
              label="社交声量"
              value={data.sentiment.social_volume.toLocaleString()}
            />
            <DataCell
              label="社交情绪"
              value={formatPercentage(data.sentiment.social_sentiment)}
              color={data.sentiment.social_sentiment > 0 ? '#22c55e' : '#ef4444'}
            />
          </DataGrid>
        </DataSection>
      </div>
    </div>
  );
};

// ============================================
// 子组件
// ============================================

const DataSection: React.FC<{
  title: string;
  color: string;
  children: React.ReactNode;
}> = ({ title, color, children }) => (
  <div className="mb-6">
    <h2
      className="text-xl font-bold mb-4 pb-2 border-b-2"
      style={{ borderColor: color, color }}
    >
      {title}
    </h2>
    {children}
  </div>
);

const DataGrid: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <div className="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-5 gap-4">
    {children}
  </div>
);

const DataCell: React.FC<{
  label: string;
  value: string;
  color?: string;
  size?: 'normal' | 'large';
  isBlinking?: boolean;
}> = ({ label, value, color = '#d1d5db', size = 'normal', isBlinking = false }) => (
  <motion.div
    animate={{
      backgroundColor: isBlinking ? 'rgba(99, 102, 241, 0.2)' : 'rgba(0, 0, 0, 0)'
    }}
    transition={{ duration: 0.5 }}
    className="bg-gray-800 rounded-lg p-4"
  >
    <div className="text-xs text-gray-400 mb-1">{label}</div>
    <div
      className={`font-mono font-bold ${size === 'large' ? 'text-2xl' : 'text-lg'}`}
      style={{ color }}
    >
      {value}
    </div>
  </motion.div>
);

// ============================================
// 工具函数
// ============================================

function formatCurrency(value: number): string {
  return `$${value.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`;
}

function formatPercentage(value: number): string {
  return `${value >= 0 ? '+' : ''}${value.toFixed(2)}%`;
}

function formatVolume(value: number): string {
  if (value > 1e9) return `${(value / 1e9).toFixed(2)}B`;
  if (value > 1e6) return `${(value / 1e6).toFixed(2)}M`;
  if (value > 1e3) return `${(value / 1e3).toFixed(2)}K`;
  return value.toFixed(2);
}

function detectChangedFields(oldData: MarketData, newData: MarketData): Set<string> {
  const changed = new Set<string>();
  
  // 递归比较对象
  function compare(oldObj: any, newObj: any, path: string = '') {
    for (const key in newObj) {
      const currentPath = path ? `${path}.${key}` : key;
      
      if (typeof newObj[key] === 'object' && newObj[key] !== null) {
        compare(oldObj[key], newObj[key], currentPath);
      } else if (oldObj[key] !== newObj[key]) {
        changed.add(currentPath);
      }
    }
  }
  
  compare(oldData, newData);
  return changed;
}
```

---

### 9.4 AI 推理链可视化组件（完整版）

**创建文件：`components/visualization/AIReasoningChain.tsx`**

```typescript
import React, { useState } from 'react';
import { motion, AnimatePresence } from 'framer-motion';

interface ReasoningStep {
  step: number;
  dimension: string;
  signal: 'bullish' | 'bearish' | 'neutral';
  confidence: number;
  weight: number;
  summary: string;
  keyMetrics: Array<{
    name: string;
    value: string | number;
    change?: string;
  }>;
  supporting_data: string[];
}

interface AIReasoningChainProps {
  asset: string;
  analysisId?: string;
}

export const AIReasoningChain: React.FC<AIReasoningChainProps> = ({ asset, analysisId }) => {
  const [expanded, setExpanded] = useState(false);
  const [steps, setSteps] = useState<ReasoningStep[]>([]);

  useEffect(() => {
    loadReasoningSteps();
  }, [asset, analysisId]);

  async function loadReasoningSteps() {
    const response = await fetch(`/api/analysis/${analysisId}/reasoning-chain`);
    const data = await response.json();
    setSteps(data.steps);
  }

  if (steps.length === 0) return null;

  const dimensionNames = {
    on_chain: '链上数据',
    technical: '技术分析',
    macro: '宏观因素',
    sentiment: '市场情绪'
  };

  const signalIcons = {
    bullish: '⬆️',
    bearish: '⬇️',
    neutral: '→'
  };

  const signalColors = {
    bullish: '#22c55e',
    bearish: '#ef4444',
    neutral: '#6b7280'
  };

  return (
    <div className="mt-8 bg-white rounded-xl shadow-lg p-6">
      <button
        onClick={() => setExpanded(!expanded)}
        className="w-full flex items-center justify-between text-left"
      >
        <h3 className="text-xl font-bold text-gray-900">
          🤖 AI 推理链
        </h3>
        <motion.div
          animate={{ rotate: expanded ? 180 : 0 }}
          transition={{ duration: 0.3 }}
        >
          ▼
        </motion.div>
      </button>

      <AnimatePresence>
        {expanded && (
          <motion.div
            initial={{ height: 0, opacity: 0 }}
            animate={{ height: 'auto', opacity: 1 }}
            exit={{ height: 0, opacity: 0 }}
            transition={{ duration: 0.4 }}
            className="mt-6 space-y-4"
          >
            {steps.map((step, index) => (
              <ReasoningStepCard
                key={index}
                step={step}
                dimensionName={dimensionNames[step.dimension]}
                signalIcon={signalIcons[step.signal]}
                signalColor={signalColors[step.signal]}
                isLast={index === steps.length - 1}
              />
            ))}

            {/* Final Score */}
            <div className="mt-6 p-6 bg-gradient-to-r from-blue-50 to-purple-50 rounded-lg border-2 border-blue-300">
              <h4 className="text-lg font-bold text-blue-900 mb-2">综合评分</h4>
              <div className="flex items-center gap-4">
                <div className="text-4xl font-bold text-blue-600">
                  {calculateFinalScore(steps)}
                </div>
                <div className="flex-1">
                  <div className="h-3 bg-gray-200 rounded-full overflow-hidden">
                    <motion.div
                      initial={{ width: 0 }}
                      animate={{ width: `${calculateFinalScore(steps)}%` }}
                      transition={{ duration: 1, delay: 0.5 }}
                      className="h-full bg-gradient-to-r from-blue-500 to-purple-600"
                    />
                  </div>
                </div>
              </div>
              <p className="mt-2 text-sm text-gray-600">
                {getScoreInterpretation(calculateFinalScore(steps))}
              </p>
            </div>

            {/* Disclaimer */}
            <div className="mt-4 p-4 bg-yellow-50 border-l-4 border-yellow-400 text-sm text-yellow-800">
              ⚠️ 此为 AI 分析推理过程，仅供参考，不构成投资建议。
            </div>
          </motion.div>
        )}
      </AnimatePresence>
    </div>
  );
};

const ReasoningStepCard: React.FC<{
  step: ReasoningStep;
  dimensionName: string;
  signalIcon: string;
  signalColor: string;
  isLast: boolean;
}> = ({ step, dimensionName, signalIcon, signalColor, isLast }) => {
  const [detailsExpanded, setDetailsExpanded] = useState(false);

  return (
    <motion.div
      initial={{ x: -20, opacity: 0 }}
      animate={{ x: 0, opacity: 1 }}
      transition={{ delay: step.step * 0.1 }}
      className="relative"
    >
      {/* Connection line */}
      {!isLast && (
        <div className="absolute left-6 top-16 w-0.5 h-full bg-gray-300" />
      )}

      <div className="bg-gray-50 rounded-lg p-5 border border-gray-200 hover:shadow-md transition">
        <div className="flex items-start gap-4">
          {/* Step number */}
          <div
            className="flex-shrink-0 w-12 h-12 rounded-full flex items-center justify-center text-white font-bold text-lg z-10"
            style={{ backgroundColor: signalColor }}
          >
            {step.step}
          </div>

          <div className="flex-1">
            {/* Header */}
            <div className="flex items-center justify-between mb-2">
              <h4 className="font-bold text-gray-900">{dimensionName}</h4>
              <div className="flex items-center gap-2">
                <span className="text-2xl">{signalIcon}</span>
                <span className="font-semibold" style={{ color: signalColor }}>
                  {step.signal === 'bullish' ? '看多' : step.signal === 'bearish' ? '看空' : '中性'}
                </span>
              </div>
            </div>

            {/* Summary */}
            <p className="text-gray-700 mb-3">{step.summary}</p>

            {/* Metrics */}
            <div className="flex flex-wrap gap-3 mb-3">
              <MetricBadge label="置信度" value={`${Math.round(step.confidence * 100)}%`} />
              <MetricBadge label="权重" value={`${Math.round(step.weight * 100)}%`} color={signalColor} />
            </div>

            {/* Key Metrics */}
            {step.keyMetrics.length > 0 && (
              <div className="mb-3">
                <div className="text-xs font-semibold text-gray-500 mb-2">关键指标：</div>
                <div className="flex flex-wrap gap-2">
                  {step.keyMetrics.map((metric, i) => (
                    <div key={i} className="px-3 py-1 bg-white rounded-full text-xs border border-gray-200">
                      <span className="font-medium">{metric.name}:</span>{' '}
                      <span className="font-bold">{metric.value}</span>
                      {metric.change && (
                        <span className={metric.change.startsWith('+') ? 'text-green-600' : 'text-red-600'}>
                          {' '}({metric.change})
                        </span>
                      )}
                    </div>
                  ))}
                </div>
              </div>
            )}

            {/* Toggle details */}
            <button
              onClick={() => setDetailsExpanded(!detailsExpanded)}
              className="text-sm text-blue-600 hover:text-blue-700 font-medium"
            >
              {detailsExpanded ? '收起详情 ▲' : '查看详情 ▼'}
            </button>

            {/* Expanded details */}
            {detailsExpanded && (
              <motion.div
                initial={{ height: 0, opacity: 0 }}
                animate={{ height: 'auto', opacity: 1 }}
                className="mt-3 p-4 bg-white rounded-lg border border-gray-200"
              >
                <div className="text-sm text-gray-600 space-y-2">
                  {step.supporting_data.map((data, i) => (
                    <div key={i} className="flex items-start gap-2">
                      <span className="text-blue-500">•</span>
                      <span>{data}</span>
                    </div>
                  ))}
                </div>
              </motion.div>
            )}
          </div>
        </div>
      </div>
    </motion.div>
  );
};

const MetricBadge: React.FC<{
  label: string;
  value: string;
  color?: string;
}> = ({ label, value, color = '#6b7280' }) => (
  <div className="flex items-center gap-2 px-3 py-1 bg-white rounded-full border border-gray-200">
    <span className="text-xs text-gray-500">{label}</span>
    <span className="font-bold text-sm" style={{ color }}>
      {value}
    </span>
  </div>
);

function calculateFinalScore(steps: ReasoningStep[]): number {
  // 加权评分
  let score = 0;
  steps.forEach(step => {
    const signalValue = step.signal === 'bullish' ? 1 : step.signal === 'bearish' ? -1 : 0;
    score += signalValue * step.confidence * step.weight;
  });

  // 转换到 0-100
  return Math.round(((score + 1) / 2) * 100);
}

function getScoreInterpretation(score: number): string {
  if (score >= 75) return '强烈看多信号';
  if (score >= 60) return '偏多信号';
  if (score >= 40) return '中性偏多';
  if (score >= 25) return '中性偏空';
  return '强烈看空信号';
}
```

---

## 章节十：用户分类与匹配系统 - 完整实现

### 10.1 多维标签数据库设计

```sql
-- 标签定义表
CREATE TABLE tags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category VARCHAR(50) NOT NULL, -- 'trading_style', 'analysis_preference', 'asset_preference', 'risk_preference'
    tag_key VARCHAR(50) NOT NULL, -- 'day_trading', 'swing', 'technical', 'onchain', 'btc', 'aggressive'
    display_name VARCHAR(100) NOT NULL,
    description TEXT,
    icon VARCHAR(50), -- emoji 或 icon name
    weight NUMERIC(3, 2) DEFAULT 1.0, -- 匹配权重
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(category, tag_key)
);

-- 初始化标签数据
INSERT INTO tags (category, tag_key, display_name, description, icon, weight) VALUES
-- Trading Style
('trading_style', 'day_trading', '日内交易', '关注短期价格波动，频繁交易', '⚡', 1.5),
('trading_style', 'swing', '波段交易', '持仓数天到数周，关注中期趋势', '📈', 1.2),
('trading_style', 'long_term', '长线投资', '持仓数月到数年，关注基本面', '🎯', 1.0),
('trading_style', 'scalping', '超短线', '持仓数分钟到数小时', '⚡⚡', 1.8),
('trading_style', 'quantitative', '量化交易', '程序化交易，数据驱动', '🤖', 1.3),

-- Analysis Preference
('analysis_preference', 'technical', '技术分析', '图表形态、指标、趋势线', '📊', 1.2),
('analysis_preference', 'fundamental', '基本面分析', '财务数据、行业趋势、估值', '📚', 1.0),
('analysis_preference', 'onchain', '链上分析', '区块链数据、地址行为、流动性', '⛓️', 1.5),
('analysis_preference', 'macro', '宏观分析', '经济指标、货币政策、地缘政治', '🌍', 1.1),
('analysis_preference', 'sentiment', '情绪分析', '社交媒体、新闻、市场情绪', '💭', 1.0),

-- Asset Preference
('asset_preference', 'crypto', '加密货币', 'BTC, ETH, DeFi, NFT', '₿', 1.5),
('asset_preference', 'stocks', '美股', '科技股、价值股、成长股', '📈', 1.2),
('asset_preference', 'forex', '外汇', '主要货币对、交叉盘', '💱', 1.0),
('asset_preference', 'commodities', '贵金属', '黄金、白银、原油', '🥇', 1.0),
('asset_preference', 'options', '期权', '衍生品交易', '📊', 1.3),

-- Risk Preference
('risk_preference', 'conservative', '保守型', '低风险，稳定收益', '🛡️', 1.0),
('risk_preference', 'balanced', '稳健型', '平衡风险与收益', '⚖️', 1.0),
('risk_preference', 'aggressive', '激进型', '追求高收益，承受高风险', '🚀', 1.5);

-- 用户标签关联表
CREATE TABLE user_tags (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    source VARCHAR(50) NOT NULL, -- 'onboarding', 'inferred', 'manual'
    confidence NUMERIC(3, 2) DEFAULT 1.0, -- 0-1，标签的置信度
    assigned_at TIMESTAMP DEFAULT NOW(),
    
    PRIMARY KEY (user_id, tag_id)
);

CREATE INDEX idx_user_tags_user ON user_tags(user_id);
CREATE INDEX idx_user_tags_tag ON user_tags(tag_id);

-- 标签相似度矩阵（预计算）
CREATE TABLE tag_similarity_matrix (
    tag_a_id UUID NOT NULL REFERENCES tags(id),
    tag_b_id UUID NOT NULL REFERENCES tags(id),
    similarity_score NUMERIC(3, 2) NOT NULL, -- 0-1
    
    PRIMARY KEY (tag_a_id, tag_b_id),
    CHECK (tag_a_id < tag_b_id) -- 避免重复
);

-- 初始化相似度数据（示例）
INSERT INTO tag_similarity_matrix (tag_a_id, tag_b_id, similarity_score)
SELECT 
    t1.id, 
    t2.id, 
    CASE 
        WHEN t1.category = t2.category THEN 0.6
        WHEN (t1.tag_key = 'day_trading' AND t2.tag_key = 'technical') THEN 0.8
        WHEN (t1.tag_key = 'long_term' AND t2.tag_key = 'fundamental') THEN 0.9
        WHEN (t1.tag_key = 'crypto' AND t2.tag_key = 'onchain') THEN 0.95
        ELSE 0.3
    END
FROM tags t1
CROSS JOIN tags t2
WHERE t1.id < t2.id;
```

---

### 10.2 匹配算法实现

**创建文件：`packages/matching/src/matcher.ts`**

```typescript
import { db } from '@finverse/database';

/**
 * 用户匹配器
 * 
 * 基于多维标签相似度计算用户之间的匹配分数
 */
export class UserMatcher {
  
  /**
   * 计算两个用户之间的相似度
   */
  async calculateSimilarity(userA: string, userB: string): Promise<number> {
    // 1. 获取两个用户的标签
    const [tagsA, tagsB] = await Promise.all([
      this.getUserTags(userA),
      this.getUserTags(userB)
    ]);

    if (tagsA.length === 0 || tagsB.length === 0) {
      return 0;
    }

    // 2. 计算 Jaccard 相似度（基础）
    const tagsASet = new Set(tagsA.map(t => t.tag_id));
    const tagsBSet = new Set(tagsB.map(t => t.tag_id));
    
    const intersection = new Set([...tagsASet].filter(x => tagsBSet.has(x)));
    const union = new Set([...tagsASet, ...tagsBSet]);
    
    const jaccardSimilarity = intersection.size / union.size;

    // 3. 加权相似度（考虑标签权重和相似度矩阵）
    let weightedSimilarity = 0;
    let totalWeight = 0;

    for (const tagA of tagsA) {
      for (const tagB of tagsB) {
        // 直接匹配
        if (tagA.tag_id === tagB.tag_id) {
          const weight = tagA.weight * tagA.confidence * tagB.confidence;
          weightedSimilarity += weight;
          totalWeight += weight;
        } else {
          // 间接匹配（通过相似度矩阵）
          const crossSimilarity = await this.getTagSimilarity(tagA.tag_id, tagB.tag_id);
          if (crossSimilarity > 0.5) {
            const weight = tagA.weight * tagA.confidence * tagB.confidence * crossSimilarity;
            weightedSimilarity += weight * crossSimilarity;
            totalWeight += weight;
          }
        }
      }
    }

    const finalSimilarity = totalWeight > 0 
      ? (weightedSimilarity / totalWeight) * 0.7 + jaccardSimilarity * 0.3
      : jaccardSimilarity;

    return Math.min(finalSimilarity, 1.0);
  }

  /**
   * 为用户推荐相似用户（用于小组推荐）
   */
  async recommendUsers(userId: string, limit: number = 10): Promise<UserRecommendation[]> {
    // 1. 获取所有候选用户
    const candidates = await db.query<{id: string, email: string}>(
      'SELECT id, email FROM users WHERE id != $1 AND id IN (SELECT DISTINCT user_id FROM user_tags)',
      [userId]
    );

    // 2. 计算相似度
    const similarities: UserRecommendation[] = [];

    for (const candidate of candidates) {
      const similarity = await this.calculateSimilarity(userId, candidate.id);
      
      if (similarity > 0.3) { // 阈值过滤
        similarities.push({
          user_id: candidate.id,
          similarity_score: similarity,
          common_tags: await this.getCommonTags(userId, candidate.id),
          reason: await this.generateRecommendationReason(userId, candidate.id)
        });
      }
    }

    // 3. 排序并返回 top N
    return similarities
      .sort((a, b) => b.similarity_score - a.similarity_score)
      .slice(0, limit);
  }

  /**
   * 推荐小组
   */
  async recommendGroups(userId: string, limit: number = 5): Promise<GroupRecommendation[]> {
    const userTags = await this.getUserTags(userId);
    const userTagIds = userTags.map(t => t.tag_id);

    // 查找包含相似标签的小组
    const groups = await db.query<any>(
      `SELECT g.*, 
              ARRAY_AGG(DISTINCT gt.tag_id) as group_tags,
              COUNT(DISTINCT gm.user_id) as member_count
       FROM groups g
       LEFT JOIN group_tags gt ON g.id = gt.group_id
       LEFT JOIN group_members gm ON g.id = gm.group_id
       WHERE g.is_public = true
         AND g.is_active = true
         AND g.id NOT IN (
           SELECT group_id FROM group_members WHERE user_id = $1
         )
       GROUP BY g.id
       HAVING ARRAY_LENGTH(ARRAY_AGG(DISTINCT gt.tag_id), 1) > 0`,
      [userId]
    );

    // 计算匹配分数
    const recommendations: GroupRecommendation[] = groups.map(group => {
      const groupTagIds = group.group_tags;
      const intersection = userTagIds.filter(t => groupTagIds.includes(t));
      const matchScore = intersection.length / Math.max(userTagIds.length, groupTagIds.length);

      return {
        group_id: group.id,
        group_name: group.name,
        match_score: matchScore,
        member_count: group.member_count,
        common_tags: intersection,
        reason: this.generateGroupRecommendationReason(intersection.length, group.member_count)
      };
    });

    return recommendations
      .filter(r => r.match_score > 0.2)
      .sort((a, b) => b.match_score - a.match_score)
      .slice(0, limit);
  }

  /**
   * 获取用户的所有标签
   */
  private async getUserTags(userId: string): Promise<UserTag[]> {
    return await db.query<UserTag>(
      `SELECT ut.*, t.weight, t.category 
       FROM user_tags ut
       JOIN tags t ON ut.tag_id = t.id
       WHERE ut.user_id = $1`,
      [userId]
    );
  }

  /**
   * 获取两个标签之间的相似度
   */
  private async getTagSimilarity(tagA: string, tagB: string): Promise<number> {
    if (tagA === tagB) return 1.0;

    const result = await db.queryOne<{similarity_score: number}>(
      `SELECT similarity_score FROM tag_similarity_matrix 
       WHERE (tag_a_id = $1 AND tag_b_id = $2) 
          OR (tag_a_id = $2 AND tag_b_id = $1)`,
      [tagA, tagB]
    );

    return result?.similarity_score || 0;
  }

  /**
   * 获取两个用户的共同标签
   */
  private async getCommonTags(userA: string, userB: string): Promise<string[]> {
    const result = await db.query<{tag_key: string}>(
      `SELECT DISTINCT t.tag_key 
       FROM user_tags uta
       JOIN user_tags utb ON uta.tag_id = utb.tag_id
       JOIN tags t ON uta.tag_id = t.id
       WHERE uta.user_id = $1 AND utb.user_id = $2`,
      [userA, userB]
    );

    return result.map(r => r.tag_key);
  }

  /**
   * 生成推荐理由
   */
  private async generateRecommendationReason(userA: string, userB: string): Promise<string> {
    const commonTags = await this.getCommonTags(userA, userB);
    
    if (commonTags.length === 0) return '可能有相似的交易风格';
    
    const tagNames = await db.query<{display_name: string}>(
      'SELECT display_name FROM tags WHERE tag_key = ANY($1)',
      [commonTags]
    );

    return `你们都关注：${tagNames.map(t => t.display_name).join('、')}`;
  }

  /**
   * 生成小组推荐理由
   */
  private generateGroupRecommendationReason(commonTagCount: number, memberCount: number): string {
    return `${commonTagCount} 个共同标签 · ${memberCount} 名成员`;
  }
}

interface UserTag {
  user_id: string;
  tag_id: string;
  confidence: number;
  weight: number;
  category: string;
}

interface UserRecommendation {
  user_id: string;
  similarity_score: number;
  common_tags: string[];
  reason: string;
}

interface GroupRecommendation {
  group_id: string;
  group_name: string;
  match_score: number;
  member_count: number;
  common_tags: string[];
  reason: string;
}
```

---

### 10.3 偏好测试实现

**创建文件：`components/onboarding/PreferenceTest.tsx`**

```typescript
import React, { useState } from 'react';
import { motion, AnimatePresence } from 'framer-motion';

interface Question {
  id: number;
  scenario: string;
  options: Array<{
    label: string;
    tags: string[]; // 选择此选项会增加的标签
    emoji: string;
  }>;
}

const QUESTIONS: Question[] = [
  {
    id: 1,
    scenario: 'BTC 突然暴跌 10%，你的第一反应是？',
    options: [
      { label: '立即止损离场', tags: ['conservative', 'day_trading'], emoji: '🛡️' },
      { label: '加仓抄底', tags: ['aggressive', 'swing'], emoji: '🚀' },
      { label: '观望，等待更多信息', tags: ['balanced', 'technical'], emoji: '👀' },
      { label: '不关心短期波动，长期持有', tags: ['long_term', 'fundamental'], emoji: '🎯' }
    ]
  },
  {
    id: 2,
    scenario: '选择一个你最信任的数据来源：',
    options: [
      { label: '技术图表和指标', tags: ['technical'], emoji: '📊' },
      { label: '链上数据和地址行为', tags: ['onchain', 'crypto'], emoji: '⛓️' },
      { label: '宏观经济新闻', tags: ['macro', 'stocks'], emoji: '🌍' },
      { label: '社交媒体情绪', tags: ['sentiment'], emoji: '💭' }
    ]
  },
  {
    id: 3,
    scenario: '你通常持仓多久？',
    options: [
      { label: '几分钟到几小时', tags: ['scalping', 'day_trading'], emoji: '⚡' },
      { label: '几天到几周', tags: ['swing'], emoji: '📈' },
      { label: '几个月到几年', tags: ['long_term'], emoji: '🎯' },
      { label: '我用程序自动交易', tags: ['quantitative'], emoji: '🤖' }
    ]
  },
  {
    id: 4,
    scenario: '你最感兴趣的市场是？',
    options: [
      { label: '加密货币（BTC, ETH, DeFi）', tags: ['crypto'], emoji: '₿' },
      { label: '美股（科技股、成长股）', tags: ['stocks'], emoji: '📈' },
      { label: '外汇和贵金属', tags: ['forex', 'commodities'], emoji: '💱' },
      { label: '都感兴趣', tags: [], emoji: '🌐' }
    ]
  },
  {
    id: 5,
    scenario: '如果一笔交易亏损 20%，你会？',
    options: [
      { label: '严格止损，控制风险', tags: ['conservative'], emoji: '🛡️' },
      { label: '分析原因，调整策略', tags: ['balanced', 'technical'], emoji: '🔍' },
      { label: '加倍下注，相信反弹', tags: ['aggressive'], emoji: '🚀' },
      { label: '不管它，等长期回本', tags: ['long_term'], emoji: '⏳' }
    ]
  },
  {
    id: 6,
    scenario: '你认为哪个最重要？',
    options: [
      { label: '准确的买卖时机', tags: ['technical', 'day_trading'], emoji: '⏰' },
      { label: '选对资产比时机重要', tags: ['fundamental', 'long_term'], emoji: '🎯' },
      { label: '风险控制和资金管理', tags: ['conservative'], emoji: '💰' },
      { label: '跟随趋势和市场情绪', tags: ['sentiment', 'swing'], emoji: '🌊' }
    ]
  },
  {
    id: 7,
    scenario: '你更喜欢看什么内容？',
    options: [
      { label: '详细的图表和技术分析', tags: ['technical'], emoji: '📊' },
      { label: '数据驱动的量化报告', tags: ['quantitative', 'onchain'], emoji: '📈' },
      { label: '宏观经济分析和新闻解读', tags: ['macro', 'fundamental'], emoji: '🗞️' },
      { label: '简洁的结论和操作建议', tags: ['day_trading'], emoji: '✅' }
    ]
  },
  {
    id: 8,
    scenario: '你希望 Agent 多久推送一次？',
    options: [
      { label: '实时推送重要信息', tags: ['day_trading', 'scalping'], emoji: '📲' },
      { label: '每天 1-2 次摘要', tags: ['swing', 'balanced'], emoji: '📅' },
      { label: '每周复盘即可', tags: ['long_term'], emoji: '📆' },
      { label: '只在重大事件时通知', tags: ['conservative'], emoji: '🚨' }
    ]
  }
];

export const PreferenceTest: React.FC<{
  onComplete: (tags: string[]) => void;
}> = ({ onComplete }) => {
  const [currentQuestion, setCurrentQuestion] = useState(0);
  const [answers, setAnswers] = useState<Record<number, number>>({});
  const [isComplete, setIsComplete] = useState(false);

  const handleAnswer = (optionIndex: number) => {
    setAnswers({ ...answers, [currentQuestion]: optionIndex });

    if (currentQuestion < QUESTIONS.length - 1) {
      setTimeout(() => setCurrentQuestion(currentQuestion + 1), 300);
    } else {
      // 测试完成，计算标签
      setTimeout(() => {
        const tags = calculateTags(answers);
        setIsComplete(true);
        onComplete(tags);
      }, 300);
    }
  };

  const progress = ((currentQuestion + 1) / QUESTIONS.length) * 100;
  const question = QUESTIONS[currentQuestion];

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-purple-50 flex items-center justify-center p-4">
      <div className="max-w-3xl w-full">
        {/* Progress bar */}
        <div className="mb-8">
          <div className="h-2 bg-gray-200 rounded-full overflow-hidden">
            <motion.div
              initial={{ width: 0 }}
              animate={{ width: `${progress}%` }}
              transition={{ duration: 0.5 }}
              className="h-full bg-gradient-to-r from-blue-500 to-purple-600"
            />
          </div>
          <div className="mt-2 text-sm text-gray-600 text-center">
            问题 {currentQuestion + 1} / {QUESTIONS.length}
          </div>
        </div>

        {/* Question card */}
        <AnimatePresence mode="wait">
          {!isComplete ? (
            <motion.div
              key={currentQuestion}
              initial={{ opacity: 0, x: 50 }}
              animate={{ opacity: 1, x: 0 }}
              exit={{ opacity: 0, x: -50 }}
              transition={{ duration: 0.3 }}
              className="bg-white rounded-2xl shadow-2xl p-8"
            >
              <h2 className="text-2xl font-bold text-gray-900 mb-6 text-center">
                {question.scenario}
              </h2>

              <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                {question.options.map((option, index) => (
                  <motion.button
                    key={index}
                    whileHover={{ scale: 1.05, boxShadow: '0 10px 40px rgba(99, 102, 241, 0.3)' }}
                    whileTap={{ scale: 0.95 }}
                    onClick={() => handleAnswer(index)}
                    className="p-6 rounded-xl border-2 border-gray-200 hover:border-blue-500 transition-all text-left"
                  >
                    <div className="text-4xl mb-3">{option.emoji}</div>
                    <div className="font-semibold text-gray-900">{option.label}</div>
                  </motion.button>
                ))}
              </div>
            </motion.div>
          ) : (
            <motion.div
              initial={{ opacity: 0, scale: 0.9 }}
              animate={{ opacity: 1, scale: 1 }}
              className="bg-white rounded-2xl shadow-2xl p-12 text-center"
            >
              <div className="text-6xl mb-6">🎉</div>
              <h2 className="text-3xl font-bold text-gray-900 mb-4">
                完成！
              </h2>
              <p className="text-gray-600 text-lg mb-6">
                正在为你生成专属 Agent...
              </p>
              <motion.div
                animate={{ rotate: 360 }}
                transition={{ duration: 2, repeat: Infinity, ease: 'linear' }}
                className="inline-block text-4xl"
              >
                🤖
              </motion.div>
            </motion.div>
          )}
        </AnimatePresence>
      </div>
    </div>
  );
};

/**
 * 根据答案计算用户标签
 */
function calculateTags(answers: Record<number, number>): string[] {
  const tagCounts: Record<string, number> = {};

  Object.entries(answers).forEach(([questionIndex, optionIndex]) => {
    const question = QUESTIONS[parseInt(questionIndex)];
    const selectedOption = question.options[optionIndex];

    selectedOption.tags.forEach(tag => {
      tagCounts[tag] = (tagCounts[tag] || 0) + 1;
    });
  });

  // 返回出现次数 >= 2 的标签
  return Object.entries(tagCounts)
    .filter(([tag, count]) => count >= 2)
    .map(([tag]) => tag);
}
```

---

## 章节十一：Agent 自定义 - 补全部分

### 11.1 进阶自定义 - 自然语言调教

**创建文件：`components/agent/NaturalLanguageCustomization.tsx`**

```typescript
import React, { useState } from 'react';
import { motion } from 'framer-motion';

export const NaturalLanguageCustomization: React.FC<{
  agentId: string;
}> = ({ agentId }) => {
  const [input, setInput] = useState('');
  const [isProcessing, setIsProcessing] = useState(false);
  const [history, setHistory] = useState<Array<{
    input: string;
    interpretation: string;
    appliedChanges: string[];
  }>>([]);

  const handleSubmit = async () => {
    if (!input.trim()) return;

    setIsProcessing(true);

    try {
      const response = await fetch(`/api/agents/${agentId}/customize`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ instruction: input })
      });

      const result = await response.json();

      setHistory([...history, {
        input: input,
        interpretation: result.interpretation,
        appliedChanges: result.changes
      }]);

      setInput('');
    } catch (error) {
      console.error('自定义失败:', error);
    } finally {
      setIsProcessing(false);
    }
  };

  return (
    <div className="bg-white rounded-xl shadow-lg p-6">
      <h2 className="text-2xl font-bold mb-4">自然语言调教 Agent</h2>

      {/* Input */}
      <div className="mb-6">
        <textarea
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="告诉 Agent 你想要什么改变...

例如：
· 我更关注链上数据，技术面不太看
· 当 RSI 低于 30 的时候重点提醒我
· 我不关心 meme 币，只看 BTC 和 ETH
· 每天早上 8 点推送市场摘要"
          className="w-full h-32 p-4 border border-gray-300 rounded-lg resize-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
        />

        <motion.button
          whileHover={{ scale: 1.02 }}
          whileTap={{ scale: 0.98 }}
          onClick={handleSubmit}
          disabled={!input.trim() || isProcessing}
          className="mt-4 w-full py-3 bg-blue-600 text-white font-semibold rounded-lg disabled:opacity-50"
        >
          {isProcessing ? '处理中...' : '应用更改'}
        </motion.button>
      </div>

      {/* History */}
      {history.length > 0 && (
        <div className="space-y-4">
          <h3 className="text-lg font-semibold">调教历史</h3>
          {history.map((item, index) => (
            <div key={index} className="p-4 bg-gray-50 rounded-lg">
              <div className="font-medium text-gray-900 mb-2">
                你说："{item.input}"
              </div>
              <div className="text-sm text-gray-600 mb-2">
                理解为：{item.interpretation}
              </div>
              <div className="text-sm">
                <span className="font-semibold">已应用：</span>
                <ul className="mt-1 space-y-1">
                  {item.appliedChanges.map((change, i) => (
                    <li key={i} className="flex items-center gap-2">
                      <span className="text-green-600">✓</span>
                      <span>{change}</span>
                    </li>
                  ))}
                </ul>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};
```

**后端 API 实现：**

```typescript
// apps/api/src/routes/agents/customize.ts
import { FastifyInstance } from 'fastify';
import { z } from 'zod';

const CustomizeSchema = z.object({
  instruction: z.string().min(5).max(500)
});

export async function customizeRoutes(fastify: FastifyInstance) {
  fastify.post('/agents/:agentId/customize', async (request, reply) => {
    const { agentId } = request.params as { agentId: string };
    const { instruction } = CustomizeSchema.parse(request.body);

    // 1. 调用 AI 解析用户指令
    const interpretation = await interpretInstruction(instruction);

    // 2. 应用更改到 Agent 配置
    const changes = await applyCustomization(agentId, interpretation);

    // 3. 重启 Agent 容器（热重载配置）
    await restartAgentContainer(agentId);

    return {
      interpretation: interpretation.summary,
      changes: changes.map(c => c.description)
    };
  });
}

/**
 * AI 解析用户指令
 */
async function interpretInstruction(instruction: string): Promise<any> {
  const aiPrompt = `用户想调整他的金融 Agent，他说：

"${instruction}"

请解析他的意图，输出JSON格式：
{
  "category": "analysis_weight" | "notification" | "data_source" | "strategy",
  "actions": [
    {"type": "adjust_weight", "dimension": "onchain", "value": 0.4},
    {"type": "add_alert", "condition": "RSI < 30", "action": "notify"},
    {"type": "filter_asset", "exclude": ["DOGE", "SHIB"]},
    {"type": "schedule_notification", "time": "08:00", "type": "daily_summary"}
  ],
  "summary": "用户希望更关注链上数据，当 RSI 低于 30 时提醒"
}`;

  const response = await callAI(aiPrompt);
  return JSON.parse(response);
}

/**
 * 应用自定义到 Agent 配置
 */
async function applyCustomization(agentId: string, interpretation: any): Promise<any[]> {
  const changes: any[] = [];

  for (const action of interpretation.actions) {
    if (action.type === 'adjust_weight') {
      // 修改 SOUL.md 中的权重配置
      await updateSOUL(agentId, {
        [`analysis_weights.${action.dimension}`]: action.value
      });

      changes.push({
        description: `调整 ${action.dimension} 权重至 ${Math.round(action.value * 100)}%`
      });
    }
    // ... 其他 action 类型处理
  }

  return changes;
}
```

---

### 11.2 高级自定义 - SOUL.md 编辑器

**创建文件：`components/agent/SOULEditor.tsx`**

```typescript
import React, { useState, useEffect } from 'react';
import Editor from '@monaco-editor/react';

export const SOULEditor: React.FC<{
  agentId: string;
}> = ({ agentId }) => {
  const [soulContent, setSoulContent] = useState('');
  const [isSaving, setIsSaving] = useState(false);
  const [hasChanges, setHasChanges] = useState(false);

  useEffect(() => {
    loadSOUL();
  }, [agentId]);

  async function loadSOUL() {
    const response = await fetch(`/api/agents/${agentId}/soul`);
    const data = await response.json();
    setSoulContent(data.content);
  }

  async function saveSOUL() {
    setIsSaving(true);

    try {
      await fetch(`/api/agents/${agentId}/soul`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          content: soulContent,
          auto_restart: true
        })
      });

      setHasChanges(false);
      alert('保存成功！Agent 将在 30 秒内重启应用新配置。');
    } catch (error) {
      alert('保存失败');
    } finally {
      setIsSaving(false);
    }
  }

  return (
    <div className="h-screen flex flex-col bg-gray-900">
      {/* Header */}
      <div className="flex items-center justify-between p-4 bg-gray-800 border-b border-gray-700">
        <h1 className="text-xl font-bold text-white">SOUL.md 编辑器</h1>
        <div className="flex items-center gap-4">
          {hasChanges && (
            <span className="text-yellow-400 text-sm">未保存</span>
          )}
          <button
            onClick={saveSOUL}
            disabled={!hasChanges || isSaving}
            className="px-4 py-2 bg-blue-600 text-white rounded-lg disabled:opacity-50 hover:bg-blue-700 transition"
          >
            {isSaving ? '保存中...' : '保存并重启'}
          </button>
        </div>
      </div>

      {/* Editor */}
      <div className="flex-1">
        <Editor
          height="100%"
          defaultLanguage="markdown"
          theme="vs-dark"
          value={soulContent}
          onChange={(value) => {
            setSoulContent(value || '');
            setHasChanges(true);
          }}
          options={{
            fontSize: 14,
            minimap: { enabled: false },
            wordWrap: 'on',
            lineNumbers: 'on'
          }}
        />
      </div>

      {/* Tips */}
      <div className="p-4 bg-gray-800 border-t border-gray-700 text-sm text-gray-400">
        💡 提示：SOUL.md 定义了 Agent 的身份、分析权重、策略等。修改后需要重启 Agent 生效。
      </div>
    </div>
  );
};
```

---

## 章节十二：多语言与多时区 - 完整实现

### 12.1 自动翻译系统

**创建文件：`packages/i18n/src/translator.ts`**

```typescript
import { db } from '@finverse/database';

/**
 * 自动翻译服务
 * 使用用户自己的 AI API key 进行翻译
 */
export class TranslationService {
  
  /**
   * 翻译文本
   */
  async translate(
    text: string,
    targetLanguage: string,
    userId: string,
    context?: string
  ): Promise<string> {
    // 1. 检查缓存
    const cacheKey = this.getCacheKey(text, targetLanguage);
    const cached = await db.redis.get(cacheKey);
    
    if (cached) {
      return cached;
    }

    // 2. 获取用户的 AI API key
    const aiKey = await this.getUserAIKey(userId);

    // 3. 调用 AI 进行翻译
    const prompt = context
      ? `请将以下${context}从英文翻译为${this.getLanguageName(targetLanguage)}，保持专业术语准确：\n\n${text}`
      : `Translate to ${targetLanguage}:\n\n${text}`;

    const translated = await this.callAI(aiKey, prompt);

    // 4. 缓存结果（30 天）
    await db.redis.setex(cacheKey, 2592000, translated);

    return translated;
  }

  /**
   * 批量翻译（用于信号摘要）
   */
  async translateBatch(
    texts: string[],
    targetLanguage: string,
    userId: string
  ): Promise<string[]> {
    const promises = texts.map(text => this.translate(text, targetLanguage, userId));
    return await Promise.all(promises);
  }

  /**
   * 翻译结构化信号
   */
  async translateSignal(
    signal: any,
    targetLanguage: string,
    userId: string
  ): Promise<any> {
    const translated = { ...signal };

    // 翻译维度摘要
    for (const [dim, data] of Object.entries(signal.dimensions)) {
      (translated.dimensions as any)[dim].summary = await this.translate(
        (data as any).summary,
        targetLanguage,
        userId,
        '金融分析摘要'
      );
    }

    // 翻译推理过程
    translated.reasoning = await this.translate(
      signal.reasoning,
      targetLanguage,
      userId,
      '金融分析推理'
    );

    return translated;
  }

  private getCacheKey(text: string, lang: string): string {
    const hash = require('crypto').createHash('md5').update(text).digest('hex');
    return `translation:${lang}:${hash}`;
  }

  private getLanguageName(code: string): string {
    const names: Record<string, string> = {
      'zh-CN': '简体中文',
      'zh-TW': '繁体中文',
      'ja': '日文',
      'ko': '韩文',
      'es': '西班牙文',
      'fr': '法文',
      'de': '德文'
    };
    return names[code] || code;
  }

  private async getUserAIKey(userId: string): Promise<string> {
    // 从加密存储中获取
    // 实现省略
    return '';
  }

  private async callAI(apiKey: string, prompt: string): Promise<string> {
    // 调用 AI API
    // 实现省略
    return '';
  }
}
```

---

### 12.2 时区转换中间件

**创建文件：`packages/timezone/src/converter.ts`**

```typescript
import { DateTime } from 'luxon';

/**
 * 时区转换工具
 */
export class TimezoneConverter {
  
  /**
   * 将 UTC 时间转换为用户时区
   */
  toUserTimezone(utcTime: Date, userTimezone: string): DateTime {
    return DateTime.fromJSDate(utcTime, { zone: 'utc' }).setZone(userTimezone);
  }

  /**
   * 将用户时区时间转换为 UTC
   */
  toUTC(localTime: Date, userTimezone: string): DateTime {
    return DateTime.fromJSDate(localTime, { zone: userTimezone }).toUTC();
  }

  /**
   * 格式化为用户友好的时间
   */
  format(time: DateTime, format: string = 'yyyy-MM-dd HH:mm:ss'): string {
    return time.toFormat(format);
  }

  /**
   * 相对时间（"3 小时前"）
   */
  relative(time: DateTime, now: DateTime = DateTime.now()): string {
    const diffSeconds = now.diff(time, 'seconds').seconds;

    if (diffSeconds < 60) return '刚刚';
    if (diffSeconds < 3600) return `${Math.floor(diffSeconds / 60)} 分钟前`;
    if (diffSeconds < 86400) return `${Math.floor(diffSeconds / 3600)} 小时前`;
    if (diffSeconds < 604800) return `${Math.floor(diffSeconds / 86400)} 天前`;
    
    return time.toFormat('yyyy-MM-dd');
  }

  /**
   * 计算用户的市场开盘时间
   */
  getMarketOpenTime(market: 'US' | 'CN' | 'EU', userTimezone: string): DateTime {
    const openTimes = {
      US: { timezone: 'America/New_York', time: '09:30' },
      CN: { timezone: 'Asia/Shanghai', time: '09:30' },
      EU: { timezone: 'Europe/London', time: '08:00' }
    };

    const { timezone, time } = openTimes[market];
    const [hour, minute] = time.split(':').map(Number);

    const marketOpen = DateTime.now()
      .setZone(timezone)
      .set({ hour, minute, second: 0, millisecond: 0 });

    return marketOpen.setZone(userTimezone);
  }
}
```

---

### 12.3 多语言 UI 支持

**创建文件：`apps/web/i18n/config.ts`**

```typescript
export const i18nConfig = {
  locales: ['zh-CN', 'zh-TW', 'en', 'ja', 'ko', 'es'],
  defaultLocale: 'en',
  localeDetection: true
};

export const translations = {
  'zh-CN': {
    common: {
      signIn: '登录',
      signUp: '注册',
      dashboard: '仪表板',
      settings: '设置',
      logout: '退出'
    },
    signals: {
      bullish: '看多',
      bearish: '看空',
      neutral: '中性',
      confidence: '置信度',
      consensus: '共识'
    },
    dimensions: {
      on_chain: '链上',
      technical: '技术',
      macro: '宏观',
      sentiment: '情绪'
    }
  },
  'en': {
    common: {
      signIn: 'Sign In',
      signUp: 'Sign Up',
      dashboard: 'Dashboard',
      settings: 'Settings',
      logout: 'Logout'
    },
    signals: {
      bullish: 'Bullish',
      bearish: 'Bearish',
      neutral: 'Neutral',
      confidence: 'Confidence',
      consensus: 'Consensus'
    },
    dimensions: {
      on_chain: 'On-Chain',
      technical: 'Technical',
      macro: 'Macro',
      sentiment: 'Sentiment'
    }
  }
  // ... 其他语言
};
```

**Next.js i18n 配置：**

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

## 总结

以上补全了以下章节的完整开发提示词：

✅ **第 6 章：公域（信号系统）**
- 信号池数据库完整设计
- 信号聚合算法实现
- 共识热力图前端组件

✅ **第 9 章：可视化（补全部分）**
- 图表模式的图层系统完整实现
- 数据模式完整实现
- AI 推理链可视化组件

✅ **第 10 章：用户分类与匹配系统**
- 多维标签数据库设计
- 匹配算法实现
- 偏好测试实现

✅ **第 11 章：Agent 自定义（补全部分）**
- 进阶自定义（自然语言调教）
- 高级自定义（SOUL.md 编辑器）

✅ **第 12 章：多语言与多时区**
- 自动翻译系统
- 时区转换中间件
- 多语言 UI 支持

现在所有遗漏的章节都已补全！
