# FinVerse 开发提示词 Part C：第 13-18 章

> **极细颗粒度可执行开发指南**  
> 覆盖：市场接入 | 风险对策 | Agent 调度器 | 路线图 | 技术栈 | 初始化

---

## 十三、覆盖市场 - 数据接入层开发提示词

### 13.1 数据标准化 Schema

**创建文件：`packages/data-schema/src/types/market-data.ts`**

```typescript
/**
 * 统一市场数据接口
 * 所有数据源适配器必须输出此格式
 */

// 基础时间序列数据点
export interface PriceDataPoint {
  timestamp: number;           // Unix timestamp (ms)
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
  source: DataSource;
  reliability: number;         // 0-1, 数据可靠性评分
}

// 资产标识符
export interface AssetIdentifier {
  symbol: string;              // 统一符号 (BTC/USD, AAPL, EUR/USD)
  exchange?: string;           // 交易所/来源
  type: AssetType;
  category: MarketCategory;
}

export enum AssetType {
  STOCK = 'stock',
  CRYPTO = 'crypto',
  FOREX = 'forex',
  COMMODITY = 'commodity',
  INDEX = 'index',
  OPTION = 'option',
  ETF = 'etf'
}

export enum MarketCategory {
  US_EQUITY = 'us_equity',
  CRYPTO_SPOT = 'crypto_spot',
  CRYPTO_FUTURES = 'crypto_futures',
  FOREX_MAJOR = 'forex_major',
  FOREX_CROSS = 'forex_cross',
  PRECIOUS_METAL = 'precious_metal',
  GLOBAL_INDEX = 'global_index'
}

export enum DataSource {
  COINGECKO = 'coingecko',
  YAHOO_FINANCE = 'yahoo_finance',
  TWELVE_DATA = 'twelve_data',
  GLASSNODE = 'glassnode',
  POLYGON = 'polygon',
  OANDA = 'oanda',
  FRED = 'fred'
}

// 链上数据 (仅加密货币)
export interface OnChainData {
  timestamp: number;
  exchange_inflow: number;     // BTC 流入交易所
  exchange_outflow: number;
  whale_transactions: number;   // 大额转账次数
  active_addresses: number;
  total_value_locked?: number;  // DeFi TVL
  miner_revenue?: number;
  source: DataSource.GLASSNODE | DataSource.CRYPTOQUANT;
}

// 宏观经济数据
export interface MacroData {
  timestamp: number;
  indicator: MacroIndicator;
  value: number;
  country: string;             // ISO 国家代码
  source: DataSource;
}

export enum MacroIndicator {
  CPI = 'cpi',                 // 消费者物价指数
  UNEMPLOYMENT = 'unemployment',
  INTEREST_RATE = 'interest_rate',
  GDP = 'gdp',
  NFP = 'nfp'                  // 非农就业
}

// 统一错误类型
export class DataFetchError extends Error {
  constructor(
    public source: DataSource,
    public code: ErrorCode,
    message: string,
    public retryable: boolean = true
  ) {
    super(`[${source}] ${code}: ${message}`);
  }
}

export enum ErrorCode {
  RATE_LIMIT = 'rate_limit',
  API_KEY_INVALID = 'api_key_invalid',
  NETWORK_ERROR = 'network_error',
  DATA_NOT_FOUND = 'data_not_found',
  PARSE_ERROR = 'parse_error'
}
```

---

### 13.2 CoinGecko 适配器实现

**创建文件：`packages/data-adapters/src/coingecko/adapter.ts`**

```typescript
import axios, { AxiosInstance } from 'axios';
import { PriceDataPoint, DataSource, AssetIdentifier, DataFetchError, ErrorCode } from '@finverse/data-schema';
import { RateLimiter } from '../utils/rate-limiter';

/**
 * CoinGecko API 适配器
 * 免费限制: 10-30 calls/min
 * 文档: https://docs.coingecko.com/v3.0.1/reference/introduction
 */
export class CoinGeckoAdapter {
  private client: AxiosInstance;
  private rateLimiter: RateLimiter;
  private coinIdCache: Map<string, string> = new Map(); // symbol -> coin_id

  constructor(
    private apiKey?: string // Pro API key (可选)
  ) {
    this.client = axios.create({
      baseURL: apiKey 
        ? 'https://pro-api.coingecko.com/api/v3'
        : 'https://api.coingecko.com/api/v3',
      headers: apiKey ? { 'x-cg-pro-api-key': apiKey } : {},
      timeout: 15000
    });

    // 免费版: 10 calls/min, Pro: 500 calls/min
    this.rateLimiter = new RateLimiter(apiKey ? 500 : 10, 60000);
  }

  /**
   * 获取历史价格数据
   * @param asset 资产标识符
   * @param from Unix timestamp (秒)
   * @param to Unix timestamp (秒)
   * @returns 标准化价格数据点数组
   */
  async getHistoricalPrices(
    asset: AssetIdentifier,
    from: number,
    to: number
  ): Promise<PriceDataPoint[]> {
    await this.rateLimiter.acquire();

    try {
      const coinId = await this.resolveCoinId(asset.symbol);
      
      const response = await this.client.get(`/coins/${coinId}/market_chart/range`, {
        params: {
          vs_currency: 'usd',
          from,
          to
        }
      });

      // CoinGecko 返回格式: { prices: [[timestamp, price], ...], ... }
      return this.transformToStandard(response.data, coinId);
    } catch (error) {
      throw this.handleError(error);
    }
  }

  /**
   * 获取实时价格
   */
  async getCurrentPrice(asset: AssetIdentifier): Promise<PriceDataPoint> {
    await this.rateLimiter.acquire();

    try {
      const coinId = await this.resolveCoinId(asset.symbol);
      
      const response = await this.client.get(`/simple/price`, {
        params: {
          ids: coinId,
          vs_currencies: 'usd',
          include_24hr_vol: true,
          include_24hr_change: true,
          include_last_updated_at: true
        }
      });

      const data = response.data[coinId];
      return {
        timestamp: data.last_updated_at * 1000,
        open: data.usd * (1 - data.usd_24h_change / 100), // 近似值
        high: data.usd * 1.01, // 需要从其他端点获取精确值
        low: data.usd * 0.99,
        close: data.usd,
        volume: data.usd_24h_vol,
        source: DataSource.COINGECKO,
        reliability: 0.95 // CoinGecko 信誉评分
      };
    } catch (error) {
      throw this.handleError(error);
    }
  }

  /**
   * 符号转 CoinGecko coin_id
   * 缓存映射以减少 API 调用
   */
  private async resolveCoinId(symbol: string): Promise<string> {
    const normalized = symbol.replace(/\/USD$/, '').toLowerCase();
    
    if (this.coinIdCache.has(normalized)) {
      return this.coinIdCache.get(normalized)!;
    }

    // 常见映射
    const commonMappings: Record<string, string> = {
      'btc': 'bitcoin',
      'eth': 'ethereum',
      'usdt': 'tether',
      'bnb': 'binancecoin',
      'sol': 'solana',
      'xrp': 'ripple',
      'ada': 'cardano',
      'doge': 'dogecoin'
    };

    if (commonMappings[normalized]) {
      const coinId = commonMappings[normalized];
      this.coinIdCache.set(normalized, coinId);
      return coinId;
    }

    // 动态查询 (消耗 API 调用)
    await this.rateLimiter.acquire();
    const response = await this.client.get('/coins/list');
    const coin = response.data.find((c: any) => c.symbol.toLowerCase() === normalized);
    
    if (!coin) {
      throw new DataFetchError(
        DataSource.COINGECKO,
        ErrorCode.DATA_NOT_FOUND,
        `Coin not found: ${symbol}`,
        false
      );
    }

    this.coinIdCache.set(normalized, coin.id);
    return coin.id;
  }

  /**
   * 转换为标准格式
   */
  private transformToStandard(data: any, coinId: string): PriceDataPoint[] {
    const prices = data.prices || [];
    const volumes = data.total_volumes || [];

    return prices.map((pricePoint: [number, number], index: number) => {
      const [timestamp, close] = pricePoint;
      const volume = volumes[index]?.[1] || 0;

      return {
        timestamp,
        open: close, // CoinGecko 免费版无 OHLC,使用 close 近似
        high: close,
        low: close,
        close,
        volume,
        source: DataSource.COINGECKO,
        reliability: 0.85 // 无真实 OHLC 数据,可靠性降低
      };
    });
  }

  /**
   * 统一错误处理
   */
  private handleError(error: any): DataFetchError {
    if (axios.isAxiosError(error)) {
      const status = error.response?.status;
      
      if (status === 429) {
        return new DataFetchError(
          DataSource.COINGECKO,
          ErrorCode.RATE_LIMIT,
          'Rate limit exceeded',
          true
        );
      }
      
      if (status === 401 || status === 403) {
        return new DataFetchError(
          DataSource.COINGECKO,
          ErrorCode.API_KEY_INVALID,
          'Invalid API key',
          false
        );
      }

      if (status === 404) {
        return new DataFetchError(
          DataSource.COINGECKO,
          ErrorCode.DATA_NOT_FOUND,
          'Data not found',
          false
        );
      }
    }

    return new DataFetchError(
      DataSource.COINGECKO,
      ErrorCode.NETWORK_ERROR,
      error.message || 'Unknown error',
      true
    );
  }
}
```

**创建配套工具：`packages/data-adapters/src/utils/rate-limiter.ts`**

```typescript
/**
 * 简单的滑动窗口速率限制器
 */
export class RateLimiter {
  private timestamps: number[] = [];

  constructor(
    private maxRequests: number,
    private windowMs: number
  ) {}

  /**
   * 获取许可,如果超限则等待
   */
  async acquire(): Promise<void> {
    const now = Date.now();
    
    // 移除窗口外的时间戳
    this.timestamps = this.timestamps.filter(
      ts => now - ts < this.windowMs
    );

    if (this.timestamps.length >= this.maxRequests) {
      // 计算需要等待的时间
      const oldestInWindow = this.timestamps[0];
      const waitTime = this.windowMs - (now - oldestInWindow) + 100; // +100ms buffer
      
      await new Promise(resolve => setTimeout(resolve, waitTime));
      return this.acquire(); // 递归重试
    }

    this.timestamps.push(now);
  }
}
```

---

### 13.3 Yahoo Finance 适配器实现

**创建文件：`packages/data-adapters/src/yahoo-finance/adapter.ts`**

```typescript
import yahooFinance from 'yahoo-finance2';
import { PriceDataPoint, DataSource, AssetIdentifier } from '@finverse/data-schema';

/**
 * Yahoo Finance 适配器
 * 优势: 免费、无 API key、美股数据全
 * 限制: 无官方 API,使用社区库
 */
export class YahooFinanceAdapter {
  constructor() {
    // yahoo-finance2 会自动处理 rate limiting
  }

  /**
   * 获取历史数据
   */
  async getHistoricalPrices(
    asset: AssetIdentifier,
    from: Date,
    to: Date,
    interval: '1d' | '1h' | '5m' = '1d'
  ): Promise<PriceDataPoint[]> {
    try {
      const symbol = this.normalizeSymbol(asset.symbol);
      
      const result = await yahooFinance.historical(symbol, {
        period1: from,
        period2: to,
        interval
      });

      return result.map(candle => ({
        timestamp: candle.date.getTime(),
        open: candle.open,
        high: candle.high,
        low: candle.low,
        close: candle.close,
        volume: candle.volume,
        source: DataSource.YAHOO_FINANCE,
        reliability: 0.98 // Yahoo Finance 数据质量高
      }));
    } catch (error: any) {
      throw new DataFetchError(
        DataSource.YAHOO_FINANCE,
        this.mapErrorCode(error),
        error.message,
        true
      );
    }
  }

  /**
   * 获取实时报价
   */
  async getCurrentPrice(asset: AssetIdentifier): Promise<PriceDataPoint> {
    const symbol = this.normalizeSymbol(asset.symbol);
    
    const quote = await yahooFinance.quote(symbol);

    return {
      timestamp: Date.now(),
      open: quote.regularMarketOpen || quote.regularMarketPrice,
      high: quote.regularMarketDayHigh || quote.regularMarketPrice,
      low: quote.regularMarketDayLow || quote.regularMarketPrice,
      close: quote.regularMarketPrice,
      volume: quote.regularMarketVolume || 0,
      source: DataSource.YAHOO_FINANCE,
      reliability: 0.98
    };
  }

  /**
   * 标准化股票代码
   * AAPL -> AAPL (不变)
   * BRK.B -> BRK-B (Yahoo 使用 -)
   */
  private normalizeSymbol(symbol: string): string {
    return symbol.replace(/\./g, '-');
  }

  private mapErrorCode(error: any): ErrorCode {
    const message = error.message?.toLowerCase() || '';
    
    if (message.includes('not found') || message.includes('invalid')) {
      return ErrorCode.DATA_NOT_FOUND;
    }
    
    if (message.includes('rate limit')) {
      return ErrorCode.RATE_LIMIT;
    }
    
    return ErrorCode.NETWORK_ERROR;
  }
}
```

---

### 13.4 Glassnode 适配器 (付费链上数据)

**创建文件：`packages/data-adapters/src/glassnode/adapter.ts`**

```typescript
import axios, { AxiosInstance } from 'axios';
import { OnChainData, DataSource } from '@finverse/data-schema';

/**
 * Glassnode API 适配器
 * 付费服务,用户提供自己的 API key
 * 文档: https://docs.glassnode.com/api/
 */
export class GlassnodeAdapter {
  private client: AxiosInstance;

  constructor(private apiKey: string) {
    this.client = axios.create({
      baseURL: 'https://api.glassnode.com/v1/metrics',
      params: { api_key: apiKey },
      timeout: 20000
    });
  }

  /**
   * 获取交易所流入流出
   */
  async getExchangeFlows(
    asset: 'BTC' | 'ETH',
    since: number, // Unix timestamp
    until: number
  ): Promise<OnChainData[]> {
    try {
      const [inflowData, outflowData] = await Promise.all([
        this.client.get('/transactions/transfers_volume_exchanges_net', {
          params: { a: asset, s: since, u: until, i: '24h' }
        }),
        this.client.get('/transactions/transfers_volume_from_exchanges', {
          params: { a: asset, s: since, u: until, i: '24h' }
        })
      ]);

      // 合并数据
      return inflowData.data.map((point: any, index: number) => ({
        timestamp: point.t * 1000,
        exchange_inflow: Math.abs(point.v),
        exchange_outflow: outflowData.data[index]?.v || 0,
        whale_transactions: 0, // 需要额外端点
        active_addresses: 0,   // 需要额外端点
        source: DataSource.GLASSNODE
      }));
    } catch (error) {
      throw this.handleError(error);
    }
  }

  /**
   * 获取活跃地址数
   */
  async getActiveAddresses(asset: 'BTC' | 'ETH', since: number, until: number): Promise<any[]> {
    const response = await this.client.get('/addresses/active_count', {
      params: { a: asset, s: since, u: until, i: '24h' }
    });
    
    return response.data;
  }

  private handleError(error: any): DataFetchError {
    if (axios.isAxiosError(error)) {
      const status = error.response?.status;
      
      if (status === 402) { // Glassnode 用 402 表示订阅不足
        return new DataFetchError(
          DataSource.GLASSNODE,
          ErrorCode.API_KEY_INVALID,
          'Insufficient subscription tier',
          false
        );
      }
    }
    
    return new DataFetchError(
      DataSource.GLASSNODE,
      ErrorCode.NETWORK_ERROR,
      error.message,
      true
    );
  }
}
```

---

### 13.5 定时数据拉取 Cron 配置

**创建文件：`packages/data-scheduler/src/cron-jobs.ts`**

```typescript
import cron from 'node-cron';
import { DataAggregator } from './aggregator';
import { logger } from '@finverse/logger';

/**
 * 数据拉取定时任务配置
 */
export class DataScheduler {
  private jobs: Map<string, cron.ScheduledTask> = new Map();

  constructor(private aggregator: DataAggregator) {}

  /**
   * 启动所有定时任务
   */
  start() {
    // 每 5 分钟拉取实时价格 (免费数据源)
    this.scheduleJob('realtime-prices', '*/5 * * * *', async () => {
      await this.aggregator.fetchRealtimePrices(['BTC/USD', 'ETH/USD', 'AAPL', 'TSLA']);
    });

    // 每小时拉取链上数据 (付费,仅订阅用户)
    this.scheduleJob('onchain-data', '0 * * * *', async () => {
      await this.aggregator.fetchOnChainData(['BTC', 'ETH']);
    });

    // 每天 UTC 00:00 拉取前一日完整数据
    this.scheduleJob('daily-aggregation', '0 0 * * *', async () => {
      const yesterday = new Date();
      yesterday.setDate(yesterday.getDate() - 1);
      yesterday.setHours(0, 0, 0, 0);
      
      await this.aggregator.fetchDailyData(yesterday);
    });

    // 宏观数据: 每周一 UTC 10:00 (覆盖主要经济数据发布)
    this.scheduleJob('macro-data', '0 10 * * 1', async () => {
      await this.aggregator.fetchMacroData();
    });

    logger.info(`Started ${this.jobs.size} data cron jobs`);
  }

  /**
   * 停止所有任务
   */
  stop() {
    this.jobs.forEach(job => job.stop());
    this.jobs.clear();
    logger.info('Stopped all data cron jobs');
  }

  private scheduleJob(name: string, schedule: string, task: () => Promise<void>) {
    const job = cron.schedule(schedule, async () => {
      logger.info(`[Cron] Running: ${name}`);
      try {
        await task();
        logger.info(`[Cron] Completed: ${name}`);
      } catch (error: any) {
        logger.error(`[Cron] Failed: ${name}`, { error: error.message });
        
        // 可重试错误: 5 分钟后重试一次
        if (error.retryable) {
          setTimeout(() => {
            logger.info(`[Cron] Retrying: ${name}`);
            task().catch(e => logger.error(`[Cron] Retry failed: ${name}`, e));
          }, 5 * 60 * 1000);
        }
      }
    });

    this.jobs.set(name, job);
  }
}
```

**创建数据聚合器：`packages/data-scheduler/src/aggregator.ts`**

```typescript
import { CoinGeckoAdapter } from '@finverse/data-adapters/coingecko';
import { YahooFinanceAdapter } from '@finverse/data-adapters/yahoo-finance';
import { GlassnodeAdapter } from '@finverse/data-adapters/glassnode';
import { PriceDataPoint, OnChainData } from '@finverse/data-schema';
import { db } from '@finverse/database';

/**
 * 统一数据聚合器
 * 从多个数据源拉取并存储到数据库
 */
export class DataAggregator {
  private coinGecko: CoinGeckoAdapter;
  private yahooFinance: YahooFinanceAdapter;
  private glassnode?: GlassnodeAdapter;

  constructor(config: {
    coinGeckoKey?: string;
    glassnodeKey?: string;
  }) {
    this.coinGecko = new CoinGeckoAdapter(config.coinGeckoKey);
    this.yahooFinance = new YahooFinanceAdapter();
    
    if (config.glassnodeKey) {
      this.glassnode = new GlassnodeAdapter(config.glassnodeKey);
    }
  }

  async fetchRealtimePrices(symbols: string[]): Promise<void> {
    const results = await Promise.allSettled(
      symbols.map(async (symbol) => {
        const isCrypto = symbol.includes('/');
        const adapter = isCrypto ? this.coinGecko : this.yahooFinance;
        
        const asset = {
          symbol,
          type: isCrypto ? 'crypto' : 'stock',
          category: isCrypto ? 'crypto_spot' : 'us_equity'
        };

        const price = await adapter.getCurrentPrice(asset as any);
        
        // 存储到 Redis (实时) + PostgreSQL (历史)
        await Promise.all([
          db.redis.setex(`price:${symbol}:latest`, 300, JSON.stringify(price)),
          db.postgres.query(
            `INSERT INTO price_data (symbol, timestamp, open, high, low, close, volume, source)
             VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
             ON CONFLICT (symbol, timestamp) DO UPDATE SET close = $6, volume = $7`,
            [symbol, new Date(price.timestamp), price.open, price.high, price.low, price.close, price.volume, price.source]
          )
        ]);
      })
    );

    const failed = results.filter(r => r.status === 'rejected');
    if (failed.length > 0) {
      logger.warn(`Failed to fetch ${failed.length}/${symbols.length} prices`);
    }
  }

  async fetchOnChainData(assets: ('BTC' | 'ETH')[]): Promise<void> {
    if (!this.glassnode) {
      logger.warn('Glassnode not configured, skipping on-chain data');
      return;
    }

    const now = Math.floor(Date.now() / 1000);
    const oneDayAgo = now - 86400;

    for (const asset of assets) {
      try {
        const flows = await this.glassnode.getExchangeFlows(asset, oneDayAgo, now);
        
        for (const data of flows) {
          await db.postgres.query(
            `INSERT INTO onchain_data (asset, timestamp, exchange_inflow, exchange_outflow, source)
             VALUES ($1, $2, $3, $4, $5)
             ON CONFLICT (asset, timestamp) DO NOTHING`,
            [asset, new Date(data.timestamp), data.exchange_inflow, data.exchange_outflow, data.source]
          );
        }
      } catch (error: any) {
        logger.error(`Failed to fetch on-chain data for ${asset}:`, error.message);
      }
    }
  }

  async fetchDailyData(date: Date): Promise<void> {
    // 拉取前一日完整 OHLCV 数据
    logger.info(`Fetching daily data for ${date.toISOString().split('T')[0]}`);
    
    // 实现省略,与 fetchRealtimePrices 类似但使用 getHistoricalPrices
  }

  async fetchMacroData(): Promise<void> {
    // 从 FRED API 拉取宏观数据
    logger.info('Fetching macro economic data');
    
    // 实现省略
  }
}
```

---

### 13.6 错误处理与重试机制

**创建文件：`packages/data-adapters/src/utils/retry.ts`**

```typescript
import { DataFetchError } from '@finverse/data-schema';
import { logger } from '@finverse/logger';

/**
 * 指数退避重试
 */
export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number;
    initialDelayMs?: number;
    maxDelayMs?: number;
    shouldRetry?: (error: any) => boolean;
  } = {}
): Promise<T> {
  const {
    maxRetries = 3,
    initialDelayMs = 1000,
    maxDelayMs = 30000,
    shouldRetry = (error) => error instanceof DataFetchError && error.retryable
  } = options;

  let lastError: any;
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      
      if (!shouldRetry(error) || attempt === maxRetries) {
        throw error;
      }

      const delay = Math.min(
        initialDelayMs * Math.pow(2, attempt),
        maxDelayMs
      );

      logger.warn(`Retry attempt ${attempt + 1}/${maxRetries} after ${delay}ms`, {
        error: error instanceof Error ? error.message : String(error)
      });

      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}

/**
 * 断路器模式
 * 防止持续调用失败的服务
 */
export class CircuitBreaker {
  private failureCount = 0;
  private lastFailureTime = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  constructor(
    private threshold: number = 5,          // 失败次数阈值
    private timeoutMs: number = 60000,      // 断路器打开时长
    private halfOpenAttempts: number = 3    // 半开状态测试次数
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailureTime > this.timeoutMs) {
        this.state = 'half-open';
        logger.info('Circuit breaker entering half-open state');
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    if (this.state === 'half-open') {
      this.state = 'closed';
      logger.info('Circuit breaker closed');
    }
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (this.failureCount >= this.threshold) {
      this.state = 'open';
      logger.error('Circuit breaker OPENED', { failureCount: this.failureCount });
    }
  }

  getState() {
    return this.state;
  }
}
```

---

## 十四、已知风险与对策 - 安全与信誉系统开发提示词

### 14.1 Agent 信誉评分算法

**创建文件：`packages/reputation/src/scoring.ts`**

```typescript
/**
 * Agent 信誉评分系统
 * 
 * 评分维度:
 * 1. 历史信号准确率 (40%)
 * 2. 信号频率稳定性 (20%)
 * 3. 账户年龄 (15%)
 * 4. 社区反馈 (15%)
 * 5. 异常行为惩罚 (10%)
 */

export interface SignalHistory {
  agentId: string;
  timestamp: number;
  asset: string;
  signal: 'bullish' | 'bearish' | 'neutral';
  confidence: number;       // 0-1
  timeframe: string;        // '24h', '48h', '1w'
  actualOutcome?: 'correct' | 'incorrect' | 'pending';
}

export interface ReputationScore {
  agentId: string;
  totalScore: number;       // 0-100
  breakdown: {
    accuracy: number;       // 0-40
    stability: number;      // 0-20
    age: number;           // 0-15
    community: number;      // 0-15
    penalty: number;        // 0 to -10
  };
  tier: 'novice' | 'reliable' | 'expert' | 'master';
  lastUpdated: Date;
}

export class ReputationEngine {
  /**
   * 计算 Agent 信誉评分
   */
  async calculateScore(agentId: string): Promise<ReputationScore> {
    const [signals, account, feedback, anomalies] = await Promise.all([
      this.getSignalHistory(agentId, 90), // 最近 90 天
      this.getAccountInfo(agentId),
      this.getCommunityFeedback(agentId),
      this.getAnomalyFlags(agentId)
    ]);

    const accuracy = this.calculateAccuracy(signals);
    const stability = this.calculateStability(signals);
    const age = this.calculateAgeScore(account.createdAt);
    const community = this.calculateCommunityScore(feedback);
    const penalty = this.calculatePenalty(anomalies);

    const totalScore = Math.max(0, Math.min(100,
      accuracy + stability + age + community + penalty
    ));

    return {
      agentId,
      totalScore,
      breakdown: { accuracy, stability, age, community, penalty },
      tier: this.getTier(totalScore),
      lastUpdated: new Date()
    };
  }

  /**
   * 准确率评分 (0-40 分)
   * 
   * 公式:
   * - 计算最近 N 个已验证信号的准确率
   * - 根据 confidence 加权 (高置信度错误惩罚更重)
   * - 时间衰减 (越近的信号权重越高)
   */
  private calculateAccuracy(signals: SignalHistory[]): number {
    const verified = signals.filter(s => s.actualOutcome !== 'pending');
    
    if (verified.length === 0) {
      return 20; // 新 Agent 默认中等分
    }

    let weightedCorrect = 0;
    let totalWeight = 0;

    verified.forEach((signal, index) => {
      // 时间衰减: 最新的信号权重 1.0, 最旧的 0.5
      const recencyWeight = 0.5 + 0.5 * (index / verified.length);
      
      // 置信度权重: 高置信度信号权重更大
      const confidenceWeight = 0.5 + signal.confidence * 0.5;
      
      const weight = recencyWeight * confidenceWeight;
      totalWeight += weight;

      if (signal.actualOutcome === 'correct') {
        weightedCorrect += weight;
      }
    });

    const accuracyRate = weightedCorrect / totalWeight;
    
    // 映射到 0-40 分
    // 50% 准确率 = 10 分 (随机猜测)
    // 70% 准确率 = 28 分
    // 90% 准确率 = 40 分
    return Math.max(0, (accuracyRate - 0.5) * 80);
  }

  /**
   * 稳定性评分 (0-20 分)
   * 
   * 评估信号发布频率的稳定性
   * - 过于频繁 (刷分) -> 低分
   * - 过于稀疏 (不活跃) -> 低分
   * - 稳定周期性 -> 高分
   */
  private calculateStability(signals: SignalHistory[]): number {
    if (signals.length < 10) {
      return 10; // 样本不足,给中等分
    }

    // 计算信号间隔的标准差
    const intervals: number[] = [];
    for (let i = 1; i < signals.length; i++) {
      intervals.push(signals[i].timestamp - signals[i - 1].timestamp);
    }

    const mean = intervals.reduce((a, b) => a + b, 0) / intervals.length;
    const variance = intervals.reduce((sum, interval) => 
      sum + Math.pow(interval - mean, 2), 0) / intervals.length;
    const stdDev = Math.sqrt(variance);

    // 变异系数 (CV) = stdDev / mean
    const cv = stdDev / mean;

    // CV 越小越稳定
    // CV < 0.3 -> 20 分
    // CV = 1.0 -> 10 分
    // CV > 2.0 -> 0 分
    if (cv < 0.3) return 20;
    if (cv > 2.0) return 0;
    return 20 - (cv - 0.3) / 1.7 * 20;
  }

  /**
   * 账户年龄评分 (0-15 分)
   * 
   * - 新注册: 0 分
   * - 1 个月: 5 分
   * - 3 个月: 10 分
   * - 6+ 个月: 15 分
   */
  private calculateAgeScore(createdAt: Date): number {
    const ageInDays = (Date.now() - createdAt.getTime()) / (1000 * 60 * 60 * 24);
    
    if (ageInDays < 30) return ageInDays / 30 * 5;
    if (ageInDays < 90) return 5 + (ageInDays - 30) / 60 * 5;
    if (ageInDays < 180) return 10 + (ageInDays - 90) / 90 * 5;
    return 15;
  }

  /**
   * 社区反馈评分 (0-15 分)
   * 
   * 基于其他用户对该 Agent 信号的反馈
   * - 点赞/点踩比例
   * - 评论质量
   */
  private calculateCommunityScore(feedback: any): number {
    const { upvotes = 0, downvotes = 0, totalViews = 0 } = feedback;
    
    if (totalViews === 0) return 7.5; // 默认中等分

    const ratio = upvotes / (upvotes + downvotes + 1); // +1 避免除零
    const engagement = (upvotes + downvotes) / totalViews;

    // 高点赞率 + 高参与度 -> 高分
    const ratioScore = ratio * 10;
    const engagementScore = Math.min(engagement * 100, 5);

    return ratioScore + engagementScore;
  }

  /**
   * 异常行为惩罚 (0 to -10 分)
   * 
   * 检测:
   * - 短时间大量发布相同信号
   * - 信号与市场数据严重矛盾
   * - 疑似操纵行为
   */
  private calculatePenalty(anomalies: AnomalyFlag[]): number {
    let penalty = 0;

    anomalies.forEach(flag => {
      switch (flag.type) {
        case 'spam':
          penalty -= 3;
          break;
        case 'contradiction':
          penalty -= 2;
          break;
        case 'manipulation':
          penalty -= 5;
          break;
      }
    });

    return Math.max(-10, penalty);
  }

  /**
   * 映射到等级
   */
  private getTier(score: number): 'novice' | 'reliable' | 'expert' | 'master' {
    if (score >= 80) return 'master';
    if (score >= 65) return 'expert';
    if (score >= 50) return 'reliable';
    return 'novice';
  }

  // 数据库查询方法 (实现省略)
  private async getSignalHistory(agentId: string, days: number): Promise<SignalHistory[]> {
    // 从数据库查询
    return [];
  }

  private async getAccountInfo(agentId: string): Promise<any> {
    return { createdAt: new Date() };
  }

  private async getCommunityFeedback(agentId: string): Promise<any> {
    return {};
  }

  private async getAnomalyFlags(agentId: string): Promise<AnomalyFlag[]> {
    return [];
  }
}

interface AnomalyFlag {
  type: 'spam' | 'contradiction' | 'manipulation';
  timestamp: Date;
  details: string;
}
```

---

### 14.2 异常检测规则

**创建文件：`packages/reputation/src/anomaly-detector.ts`**

```typescript
import { SignalHistory } from './scoring';
import { db } from '@finverse/database';
import { logger } from '@finverse/logger';

/**
 * 异常行为检测器
 */
export class AnomalyDetector {
  /**
   * 检测垃圾信号 (Spam)
   * 
   * 规则:
   * - 1 小时内发布 > 10 个信号
   * - 连续 5+ 个信号完全相同
   */
  async detectSpam(agentId: string): Promise<boolean> {
    const oneHourAgo = Date.now() - 3600000;
    const recentSignals = await db.query<SignalHistory>(
      'SELECT * FROM signals WHERE agent_id = $1 AND timestamp > $2 ORDER BY timestamp DESC',
      [agentId, oneHourAgo]
    );

    // 检查数量
    if (recentSignals.length > 10) {
      await this.flagAnomaly(agentId, 'spam', `${recentSignals.length} signals in 1 hour`);
      return true;
    }

    // 检查重复
    const duplicates = this.findConsecutiveDuplicates(recentSignals, 5);
    if (duplicates) {
      await this.flagAnomaly(agentId, 'spam', 'Consecutive duplicate signals');
      return true;
    }

    return false;
  }

  /**
   * 检测矛盾信号
   * 
   * 规则:
   * - 信号方向与实际价格走势完全相反
   * - 例: 发布 "bullish" 信号时价格暴跌 -5%
   */
  async detectContradiction(signal: SignalHistory): Promise<boolean> {
    // 获取信号发布后的实际价格变动
    const actualChange = await this.getPriceChange(
      signal.asset,
      signal.timestamp,
      signal.timeframe
    );

    const expectedDirection = signal.signal === 'bullish' ? 1 : -1;
    const actualDirection = Math.sign(actualChange);

    // 高置信度信号 + 相反走势 + 变动幅度 > 3%
    if (
      signal.confidence > 0.7 &&
      expectedDirection !== actualDirection &&
      Math.abs(actualChange) > 0.03
    ) {
      await this.flagAnomaly(
        signal.agentId,
        'contradiction',
        `Signal: ${signal.signal}, Actual: ${actualChange.toFixed(2)}%`
      );
      return true;
    }

    return false;
  }

  /**
   * 检测操纵行为
   * 
   * 规则:
   * - 与其他 Agent 高度协同 (疑似僵尸网络)
   * - 在小市值币种上发布极端信号
   */
  async detectManipulation(agentId: string): Promise<boolean> {
    // 检测与其他 Agent 的相关性
    const correlatedAgents = await this.findCorrelatedAgents(agentId);
    
    if (correlatedAgents.length > 5) {
      // 与 5+ 个 Agent 高度同步 (>90% 相同信号)
      await this.flagAnomaly(
        agentId,
        'manipulation',
        `Correlated with ${correlatedAgents.length} agents`
      );
      return true;
    }

    return false;
  }

  /**
   * 查找连续重复信号
   */
  private findConsecutiveDuplicates(signals: SignalHistory[], threshold: number): boolean {
    for (let i = 0; i < signals.length - threshold + 1; i++) {
      const slice = signals.slice(i, i + threshold);
      const first = slice[0];
      
      const allSame = slice.every(s =>
        s.asset === first.asset &&
        s.signal === first.signal &&
        Math.abs(s.confidence - first.confidence) < 0.05
      );

      if (allSame) return true;
    }
    return false;
  }

  /**
   * 计算价格变动
   */
  private async getPriceChange(asset: string, fromTimestamp: number, timeframe: string): Promise<number> {
    const duration = this.parseTimeframe(timeframe);
    const toTimestamp = fromTimestamp + duration;

    const [startPrice, endPrice] = await Promise.all([
      db.queryOne<number>('SELECT close FROM price_data WHERE asset = $1 AND timestamp >= $2 ORDER BY timestamp ASC LIMIT 1', [asset, fromTimestamp]),
      db.queryOne<number>('SELECT close FROM price_data WHERE asset = $1 AND timestamp <= $2 ORDER BY timestamp DESC LIMIT 1', [asset, toTimestamp])
    ]);

    if (!startPrice || !endPrice) return 0;
    return (endPrice - startPrice) / startPrice;
  }

  private parseTimeframe(timeframe: string): number {
    const match = timeframe.match(/(\d+)([hd])/);
    if (!match) return 86400000; // 默认 24h

    const [, num, unit] = match;
    const multiplier = unit === 'h' ? 3600000 : 86400000;
    return parseInt(num) * multiplier;
  }

  /**
   * 查找相关 Agent
   */
  private async findCorrelatedAgents(agentId: string): Promise<string[]> {
    // 复杂算法,省略实现
    // 思路: 计算信号发布时间和内容的皮尔逊相关系数
    return [];
  }

  /**
   * 记录异常标记
   */
  private async flagAnomaly(agentId: string, type: string, details: string): Promise<void> {
    await db.query(
      'INSERT INTO anomaly_flags (agent_id, type, details, timestamp) VALUES ($1, $2, $3, $4)',
      [agentId, type, details, Date.now()]
    );

    logger.warn(`Anomaly detected: ${agentId} - ${type}`, { details });
  }
}
```

---

### 14.3 API Key 加密方案 (AES-256-GCM)

**创建文件：`packages/security/src/encryption.ts`**

```typescript
import crypto from 'crypto';

/**
 * API Key 加密服务
 * 
 * 使用 AES-256-GCM (Galois/Counter Mode)
 * - 认证加密 (Authenticated Encryption)
 * - 防止篡改
 * - 每个 key 使用唯一 IV (初始化向量)
 */
export class EncryptionService {
  private algorithm = 'aes-256-gcm';
  private masterKey: Buffer;

  constructor(masterKeyHex: string) {
    // 主密钥从环境变量读取 (32 字节 = 256 位)
    this.masterKey = Buffer.from(masterKeyHex, 'hex');
    
    if (this.masterKey.length !== 32) {
      throw new Error('Master key must be 32 bytes (256 bits)');
    }
  }

  /**
   * 加密 API key
   * 
   * @returns 格式: iv:authTag:ciphertext (全部 hex 编码)
   */
  encrypt(apiKey: string): string {
    // 生成随机 IV (12 字节推荐用于 GCM)
    const iv = crypto.randomBytes(12);
    
    const cipher = crypto.createCipheriv(this.algorithm, this.masterKey, iv);
    
    let ciphertext = cipher.update(apiKey, 'utf8', 'hex');
    ciphertext += cipher.final('hex');
    
    // GCM 模式提供认证标签
    const authTag = cipher.getAuthTag();

    // 组合: iv:authTag:ciphertext
    return [
      iv.toString('hex'),
      authTag.toString('hex'),
      ciphertext
    ].join(':');
  }

  /**
   * 解密 API key
   */
  decrypt(encrypted: string): string {
    const [ivHex, authTagHex, ciphertext] = encrypted.split(':');
    
    if (!ivHex || !authTagHex || !ciphertext) {
      throw new Error('Invalid encrypted format');
    }

    const iv = Buffer.from(ivHex, 'hex');
    const authTag = Buffer.from(authTagHex, 'hex');

    const decipher = crypto.createDecipheriv(this.algorithm, this.masterKey, iv);
    decipher.setAuthTag(authTag);

    let plaintext = decipher.update(ciphertext, 'hex', 'utf8');
    plaintext += decipher.final('utf8');

    return plaintext;
  }

  /**
   * 生成新的主密钥 (仅用于初始化)
   */
  static generateMasterKey(): string {
    return crypto.randomBytes(32).toString('hex');
  }
}

/**
 * 使用示例
 */
// const encryption = new EncryptionService(process.env.MASTER_KEY!);
// const encrypted = encryption.encrypt('sk-abc123...');
// console.log(encrypted); // 输出: 3a7b...f2:8c1d...a9:d4e6...
// const decrypted = encryption.decrypt(encrypted);
// console.log(decrypted); // 输出: sk-abc123...
```

**数据库存储方案：`migrations/001_api_keys.sql`**

```sql
CREATE TABLE user_api_keys (
  user_id UUID NOT NULL,
  service VARCHAR(50) NOT NULL,  -- 'openai', 'anthropic', 'glassnode', etc.
  encrypted_key TEXT NOT NULL,    -- 加密后的 key
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_used_at TIMESTAMPTZ,
  
  PRIMARY KEY (user_id, service)
);

-- 索引
CREATE INDEX idx_api_keys_user ON user_api_keys(user_id);

-- 审计日志
CREATE TABLE api_key_audit (
  id SERIAL PRIMARY KEY,
  user_id UUID NOT NULL,
  service VARCHAR(50) NOT NULL,
  action VARCHAR(20) NOT NULL,  -- 'created', 'updated', 'deleted', 'decrypted'
  ip_address INET,
  timestamp TIMESTAMPTZ DEFAULT NOW()
);
```

**使用 API Key 时的安全流程：**

```typescript
// packages/agent-runtime/src/api-key-manager.ts
import { EncryptionService } from '@finverse/security';
import { db } from '@finverse/database';

export class ApiKeyManager {
  constructor(private encryption: EncryptionService) {}

  /**
   * 存储用户 API key
   */
  async storeKey(userId: string, service: string, apiKey: string): Promise<void> {
    const encrypted = this.encryption.encrypt(apiKey);
    
    await db.query(
      `INSERT INTO user_api_keys (user_id, service, encrypted_key)
       VALUES ($1, $2, $3)
       ON CONFLICT (user_id, service) DO UPDATE SET encrypted_key = $3`,
      [userId, service, encrypted]
    );

    // 审计日志
    await db.query(
      'INSERT INTO api_key_audit (user_id, service, action) VALUES ($1, $2, $3)',
      [userId, service, 'created']
    );
  }

  /**
   * 获取解密后的 API key (仅在 Agent 容器内调用)
   */
  async getKey(userId: string, service: string): Promise<string | null> {
    const result = await db.queryOne<{ encrypted_key: string }>(
      'SELECT encrypted_key FROM user_api_keys WHERE user_id = $1 AND service = $2',
      [userId, service]
    );

    if (!result) return null;

    const decrypted = this.encryption.decrypt(result.encrypted_key);

    // 更新最后使用时间
    await db.query(
      'UPDATE user_api_keys SET last_used_at = NOW() WHERE user_id = $1 AND service = $2',
      [userId, service]
    );

    return decrypted;
  }

  /**
   * 删除 API key
   */
  async deleteKey(userId: string, service: string): Promise<void> {
    await db.query(
      'DELETE FROM user_api_keys WHERE user_id = $1 AND service = $2',
      [userId, service]
    );

    await db.query(
      'INSERT INTO api_key_audit (user_id, service, action) VALUES ($1, $2, $3)',
      [userId, service, 'deleted']
    );
  }
}
```

---

### 14.4 免责声明组件

**创建文件：`packages/ui/src/components/DisclaimerBanner.tsx`**

```tsx
import React, { useState, useEffect } from 'react';
import { AlertTriangle, X } from 'lucide-react';

/**
 * 免责声明横幅
 * 
 * 位置:
 * - 每个 AI 分析结果下方
 * - 信号发布页面顶部
 * - 小组分析页面
 */
export const DisclaimerBanner: React.FC<{
  variant?: 'default' | 'compact';
  dismissible?: boolean;
}> = ({ variant = 'default', dismissible = false }) => {
  const [dismissed, setDismissed] = useState(false);

  useEffect(() => {
    // 检查本地存储是否已永久关闭
    const permanentlyDismissed = localStorage.getItem('disclaimer_dismissed');
    if (permanentlyDismissed === 'true') {
      setDismissed(true);
    }
  }, []);

  if (dismissed) return null;

  const handleDismiss = () => {
    setDismissed(true);
    if (dismissible) {
      localStorage.setItem('disclaimer_dismissed', 'true');
    }
  };

  if (variant === 'compact') {
    return (
      <div className="flex items-center gap-2 px-3 py-2 bg-yellow-50 border-l-4 border-yellow-400 text-sm text-yellow-800">
        <AlertTriangle size={16} />
        <span>AI 分析仅供参考，不构成投资建议</span>
      </div>
    );
  }

  return (
    <div className="relative bg-gradient-to-r from-yellow-50 to-orange-50 border border-yellow-200 rounded-lg p-4 shadow-sm">
      {dismissible && (
        <button
          onClick={handleDismiss}
          className="absolute top-2 right-2 text-yellow-600 hover:text-yellow-800"
          aria-label="关闭"
        >
          <X size={20} />
        </button>
      )}
      
      <div className="flex items-start gap-3">
        <AlertTriangle className="text-yellow-600 flex-shrink-0 mt-1" size={24} />
        
        <div className="flex-1">
          <h3 className="font-semibold text-yellow-900 mb-2">投资风险提示</h3>
          
          <ul className="space-y-1 text-sm text-yellow-800">
            <li>• 所有分析均由 AI 生成，<strong>不构成投资建议</strong></li>
            <li>• FinVerse 不提供金融咨询服务，不承担任何投资损失责任</li>
            <li>• AI 可能产生错误、偏见或过时的信息</li>
            <li>• 请基于您自己的研究和风险承受能力做出投资决策</li>
            <li>• 过往表现不代表未来收益</li>
          </ul>

          <p className="mt-3 text-xs text-yellow-700">
            使用本平台即表示您同意 <a href="/terms" className="underline">服务条款</a> 和 <a href="/risk" className="underline">风险声明</a>
          </p>
        </div>
      </div>
    </div>
  );
};

/**
 * AI 分析卡片包装器
 * 自动在分析下方附加免责声明
 */
export const AIAnalysisCard: React.FC<{
  children: React.ReactNode;
  showDisclaimer?: boolean;
}> = ({ children, showDisclaimer = true }) => {
  return (
    <div className="space-y-3">
      <div className="bg-white rounded-lg border border-gray-200 p-4">
        {children}
      </div>
      
      {showDisclaimer && <DisclaimerBanner variant="compact" />}
    </div>
  );
};
```

**法律文本：`public/legal/risk-disclosure.md`**

```markdown
# 风险披露声明

**最后更新: 2026-02-07**

## 1. 服务性质

FinVerse 是一个技术平台，提供以下服务：
- AI Agent 托管和运行环境
- 数据可视化工具
- 社区协作功能

FinVerse **不是**：
- 持牌金融顾问
- 投资建议提供商
- 经纪商或交易平台

## 2. AI 生成内容的限制

平台上所有 AI 生成的分析、信号、预测均存在以下限制：

- **非确定性**: AI 模型基于概率，输出可能不准确
- **数据延迟**: 实时数据可能有延迟，影响分析时效性
- **模型偏见**: AI 可能继承训练数据中的偏见
- **不可预测**: 金融市场受复杂因素影响，AI 无法预测所有情况

## 3. 用户责任

您理解并同意：

- 您对自己的投资决策完全负责
- 您应该进行独立研究和尽职调查
- 您应该咨询持牌专业人士（如有需要）
- 您理解投资存在本金损失风险

## 4. 免责条款

在法律允许的最大范围内，FinVerse 及其关联方：

- 不对任何投资损失负责
- 不保证平台服务的准确性、完整性、及时性
- 不保证 AI 分析的质量或可靠性
- 不对第三方数据源的错误负责

## 5. 合规声明

FinVerse 在不同司法管辖区可能受不同监管：

- 美国: 不提供 SEC/FINRA 监管的投资建议
- 欧盟: 遵守 MiFID II 和 GDPR
- 中国: 不面向中国大陆居民提供服务

请确保您所在地区允许使用此类服务。

## 6. 同意确认

使用 FinVerse 即表示您已阅读、理解并同意本风险披露声明。

如有疑问，请联系: legal@finverse.ai
```

---

## 十五、Agent 调度器 - 容器编排开发提示词

### 15.1 Docker Compose 完整配置

**创建文件：`docker-compose.yml`**

```yaml
version: '3.9'

services:
  # ============================================
  # 核心服务
  # ============================================
  
  postgres:
    image: postgres:16-alpine
    container_name: finverse-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: finverse
      POSTGRES_USER: finverse
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U finverse"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - finverse-network

  redis:
    image: redis:7-alpine
    container_name: finverse-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - finverse-network

  # ============================================
  # FinVerse 应用层
  # ============================================

  api:
    build:
      context: .
      dockerfile: ./apps/api/Dockerfile
    container_name: finverse-api
    restart: unless-stopped
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://finverse:${POSTGRES_PASSWORD}@postgres:5432/finverse
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      JWT_SECRET: ${JWT_SECRET}
      MASTER_ENCRYPTION_KEY: ${MASTER_ENCRYPTION_KEY}
    ports:
      - "3001:3001"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - finverse-network
    volumes:
      - ./logs:/app/logs

  web:
    build:
      context: .
      dockerfile: ./apps/web/Dockerfile
    container_name: finverse-web
    restart: unless-stopped
    environment:
      NEXT_PUBLIC_API_URL: ${API_URL}
    ports:
      - "3000:3000"
    depends_on:
      - api
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - finverse-network

  # ============================================
  # Agent 调度器
  # ============================================

  agent-orchestrator:
    build:
      context: .
      dockerfile: ./apps/agent-orchestrator/Dockerfile
    container_name: finverse-orchestrator
    restart: unless-stopped
    environment:
      DATABASE_URL: postgresql://finverse:${POSTGRES_PASSWORD}@postgres:5432/finverse
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      DOCKER_HOST: unix:///var/run/docker.sock
      MAX_AGENTS: ${MAX_AGENTS:-1000}
      AGENT_IMAGE: finverse/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # Docker-in-Docker
      - agent_data:/app/data
    depends_on:
      - postgres
      - redis
    networks:
      - finverse-network
      - agent-network  # 独立网络用于 Agent 容器

  # ============================================
  # 监控与日志
  # ============================================

  prometheus:
    image: prom/prometheus:latest
    container_name: finverse-prometheus
    restart: unless-stopped
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    ports:
      - "9090:9090"
    networks:
      - finverse-network

  grafana:
    image: grafana/grafana:latest
    container_name: finverse-grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
    ports:
      - "3003:3000"
    depends_on:
      - prometheus
    networks:
      - finverse-network

  # ============================================
  # 反向代理 (生产环境)
  # ============================================

  nginx:
    image: nginx:alpine
    container_name: finverse-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
      - ./nginx/logs:/var/log/nginx
    depends_on:
      - web
      - api
    networks:
      - finverse-network

volumes:
  postgres_data:
  redis_data:
  agent_data:
  prometheus_data:
  grafana_data:

networks:
  finverse-network:
    driver: bridge
  agent-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

---

### 15.2 Agent 容器 Dockerfile

**创建文件：`apps/agent-runtime/Dockerfile`**

```dockerfile
# ============================================
# Stage 1: 构建 OpenClaw
# ============================================
FROM node:22-alpine AS builder

WORKDIR /build

# 复制 package files
COPY package*.json ./
COPY packages/openclaw/package*.json ./packages/openclaw/

# 安装依赖
RUN npm ci --production=false

# 复制源代码
COPY packages/openclaw ./packages/openclaw
COPY apps/agent-runtime ./apps/agent-runtime

# 构建
RUN npm run build

# ============================================
# Stage 2: 运行时镜像
# ============================================
FROM node:22-alpine

# 安全性: 非 root 用户运行
RUN addgroup -g 1001 -S agentuser && \
    adduser -S agentuser -u 1001

WORKDIR /app

# 只复制必要的文件
COPY --from=builder --chown=agentuser:agentuser /build/node_modules ./node_modules
COPY --from=builder --chown=agentuser:agentuser /build/packages/openclaw/dist ./openclaw
COPY --from=builder --chown=agentuser:agentuser /build/apps/agent-runtime/dist ./runtime

# 安装系统依赖 (Python for 数据分析脚本)
RUN apk add --no-cache python3 py3-pip curl

# Agent 工作目录
RUN mkdir -p /app/workspace && chown -R agentuser:agentuser /app/workspace

# 切换到非 root 用户
USER agentuser

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3100/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"

# 暴露端口 (内部健康检查)
EXPOSE 3100

# 启动命令
CMD ["node", "runtime/index.js"]
```

---

### 15.3 Agent 调度器实现

**创建文件：`apps/agent-orchestrator/src/orchestrator.ts`**

```typescript
import Docker from 'dockerode';
import { EventEmitter } from 'events';
import { db } from '@finverse/database';
import { logger } from '@finverse/logger';

/**
 * Agent 生命周期状态
 */
export enum AgentState {
  CREATING = 'creating',
  ACTIVE = 'active',
  STANDBY = 'standby',
  HIBERNATED = 'hibernated',
  STOPPING = 'stopping',
  STOPPED = 'stopped',
  ERROR = 'error'
}

interface AgentConfig {
  userId: string;
  apiKeys: Record<string, string>;  // service -> encrypted key
  preferences: any;
  chatChannels: string[];
}

/**
 * Agent 调度器
 * 
 * 职责:
 * - 创建/销毁 Agent 容器
 * - 健康检查
 * - 自动扩缩容
 * - 故障恢复
 */
export class AgentOrchestrator extends EventEmitter {
  private docker: Docker;
  private agents: Map<string, AgentInstance> = new Map();
  private healthCheckInterval: NodeJS.Timeout | null = null;

  constructor() {
    super();
    this.docker = new Docker({ socketPath: '/var/run/docker.sock' });
  }

  /**
   * 启动调度器
   */
  async start(): Promise<void> {
    logger.info('Starting Agent Orchestrator');

    // 恢复已存在的 Agent 容器
    await this.recoverExistingAgents();

    // 启动健康检查循环
    this.startHealthCheck();

    // 启动状态机循环
    this.startStateMachine();

    logger.info('Agent Orchestrator started');
  }

  /**
   * 创建新 Agent
   */
  async createAgent(config: AgentConfig): Promise<AgentInstance> {
    const { userId } = config;

    logger.info(`Creating agent for user ${userId}`);

    try {
      // 1. 创建专用网络 (可选,用于隔离)
      // const network = await this.docker.createNetwork({
      //   Name: `agent-${userId}`,
      //   Driver: 'bridge'
      // });

      // 2. 创建容器
      const container = await this.docker.createContainer({
        Image: 'finverse/agent:latest',
        name: `agent-${userId}`,
        
        Env: [
          `USER_ID=${userId}`,
          `API_KEYS=${JSON.stringify(config.apiKeys)}`,
          `PREFERENCES=${JSON.stringify(config.preferences)}`,
          `CHAT_CHANNELS=${config.chatChannels.join(',')}`
        ],

        // 资源限制
        HostConfig: {
          Memory: 512 * 1024 * 1024,      // 512 MB
          MemorySwap: 1024 * 1024 * 1024, // 1 GB (含 swap)
          NanoCpus: 1 * 1e9,              // 1 CPU core
          CpuShares: 1024,                 // CPU 权重
          
          // 存储卷
          Binds: [
            `agent-${userId}-data:/app/workspace`
          ],

          // 网络隔离
          NetworkMode: 'agent-network',

          // 日志限制
          LogConfig: {
            Type: 'json-file',
            Config: {
              'max-size': '10m',
              'max-file': '3'
            }
          },

          // 安全性
          SecurityOpt: ['no-new-privileges'],
          ReadonlyRootfs: false,  // Agent 需要写入工作目录
          
          // 重启策略
          RestartPolicy: {
            Name: 'unless-stopped'
          }
        },

        Labels: {
          'finverse.agent': 'true',
          'finverse.user_id': userId,
          'finverse.created_at': new Date().toISOString()
        }
      });

      // 3. 启动容器
      await container.start();

      // 4. 等待健康检查通过
      const healthy = await this.waitForHealthy(container, 30000);
      if (!healthy) {
        throw new Error('Agent failed health check');
      }

      // 5. 创建 Agent 实例记录
      const instance: AgentInstance = {
        userId,
        containerId: container.id,
        state: AgentState.ACTIVE,
        createdAt: new Date(),
        lastActiveAt: new Date(),
        resources: {
          memoryMB: 512,
          cpuCores: 1
        }
      };

      this.agents.set(userId, instance);

      // 6. 保存到数据库
      await db.query(
        `INSERT INTO agent_instances (user_id, container_id, state, created_at)
         VALUES ($1, $2, $3, $4)`,
        [userId, container.id, AgentState.ACTIVE, instance.createdAt]
      );

      logger.info(`Agent created for user ${userId}: ${container.id}`);
      this.emit('agent:created', instance);

      return instance;

    } catch (error: any) {
      logger.error(`Failed to create agent for user ${userId}:`, error);
      throw error;
    }
  }

  /**
   * 停止 Agent
   */
  async stopAgent(userId: string, graceful: boolean = true): Promise<void> {
    const instance = this.agents.get(userId);
    if (!instance) {
      throw new Error(`Agent not found: ${userId}`);
    }

    logger.info(`Stopping agent for user ${userId}`);

    try {
      const container = this.docker.getContainer(instance.containerId);

      if (graceful) {
        // 优雅停止: 发送 SIGTERM,等待 10 秒
        await container.stop({ t: 10 });
      } else {
        // 强制停止
        await container.kill();
      }

      instance.state = AgentState.STOPPED;
      this.agents.delete(userId);

      await db.query(
        'UPDATE agent_instances SET state = $1, stopped_at = NOW() WHERE user_id = $2',
        [AgentState.STOPPED, userId]
      );

      this.emit('agent:stopped', instance);

    } catch (error: any) {
      logger.error(`Failed to stop agent for user ${userId}:`, error);
      throw error;
    }
  }

  /**
   * 休眠 Agent (降低资源占用)
   */
  async hibernateAgent(userId: string): Promise<void> {
    const instance = this.agents.get(userId);
    if (!instance) return;

    logger.info(`Hibernating agent for user ${userId}`);

    try {
      const container = this.docker.getContainer(instance.containerId);
      
      // 暂停容器 (freeze cgroup)
      await container.pause();

      instance.state = AgentState.HIBERNATED;

      await db.query(
        'UPDATE agent_instances SET state = $1 WHERE user_id = $2',
        [AgentState.HIBERNATED, userId]
      );

      this.emit('agent:hibernated', instance);

    } catch (error: any) {
      logger.error(`Failed to hibernate agent for user ${userId}:`, error);
    }
  }

  /**
   * 唤醒休眠的 Agent
   */
  async wakeAgent(userId: string): Promise<void> {
    const instance = this.agents.get(userId);
    if (!instance || instance.state !== AgentState.HIBERNATED) return;

    logger.info(`Waking agent for user ${userId}`);

    try {
      const container = this.docker.getContainer(instance.containerId);
      
      await container.unpause();

      instance.state = AgentState.ACTIVE;
      instance.lastActiveAt = new Date();

      await db.query(
        'UPDATE agent_instances SET state = $1, last_active_at = NOW() WHERE user_id = $2',
        [AgentState.ACTIVE, userId]
      );

      this.emit('agent:woken', instance);

    } catch (error: any) {
      logger.error(`Failed to wake agent for user ${userId}:`, error);
    }
  }

  /**
   * 健康检查循环
   */
  private startHealthCheck(): void {
    this.healthCheckInterval = setInterval(async () => {
      for (const [userId, instance] of this.agents.entries()) {
        try {
          const container = this.docker.getContainer(instance.containerId);
          const inspect = await container.inspect();

          // 检查容器状态
          if (!inspect.State.Running) {
            logger.warn(`Agent container not running: ${userId}`);
            await this.handleUnhealthyAgent(userId, 'not_running');
            continue;
          }

          // 检查健康检查状态
          if (inspect.State.Health?.Status === 'unhealthy') {
            logger.warn(`Agent health check failed: ${userId}`);
            await this.handleUnhealthyAgent(userId, 'health_check_failed');
          }

        } catch (error: any) {
          logger.error(`Health check error for ${userId}:`, error);
          await this.handleUnhealthyAgent(userId, 'error');
        }
      }
    }, 60000); // 每分钟检查一次
  }

  /**
   * 状态机循环 (自动扩缩容)
   */
  private startStateMachine(): void {
    setInterval(async () => {
      const now = Date.now();
      const ONE_HOUR = 3600000;
      const ONE_DAY = 86400000;
      const ONE_WEEK = 604800000;

      for (const [userId, instance] of this.agents.entries()) {
        const inactiveTime = now - instance.lastActiveAt.getTime();

        // 状态转换规则
        if (instance.state === AgentState.ACTIVE && inactiveTime > ONE_HOUR) {
          // 活跃 -> 待机 (1 小时无活动)
          instance.state = AgentState.STANDBY;
          logger.info(`Agent ${userId} moved to STANDBY`);
        }
        else if (instance.state === AgentState.STANDBY && inactiveTime > ONE_DAY) {
          // 待机 -> 休眠 (1 天无活动)
          await this.hibernateAgent(userId);
        }
        else if (instance.state === AgentState.HIBERNATED && inactiveTime > ONE_WEEK) {
          // 休眠 -> 停止 (1 周无活动,可选)
          // await this.stopAgent(userId, true);
        }
      }
    }, 300000); // 每 5 分钟运行一次
  }

  /**
   * 处理不健康的 Agent
   */
  private async handleUnhealthyAgent(userId: string, reason: string): Promise<void> {
    logger.error(`Handling unhealthy agent ${userId}: ${reason}`);

    const instance = this.agents.get(userId);
    if (!instance) return;

    instance.state = AgentState.ERROR;

    // 尝试重启
    try {
      await this.stopAgent(userId, false);
      
      // 等待 5 秒
      await new Promise(resolve => setTimeout(resolve, 5000));

      // 重新创建
      const config = await this.loadAgentConfig(userId);
      await this.createAgent(config);

      logger.info(`Successfully restarted agent ${userId}`);
    } catch (error: any) {
      logger.error(`Failed to restart agent ${userId}:`, error);
      
      // 通知用户
      this.emit('agent:failed', { userId, reason, error: error.message });
    }
  }

  /**
   * 等待容器健康检查通过
   */
  private async waitForHealthy(container: Docker.Container, timeoutMs: number): Promise<boolean> {
    const startTime = Date.now();

    while (Date.now() - startTime < timeoutMs) {
      try {
        const inspect = await container.inspect();
        
        if (inspect.State.Health?.Status === 'healthy') {
          return true;
        }

        await new Promise(resolve => setTimeout(resolve, 2000)); // 每 2 秒检查一次
      } catch (error) {
        // 容器可能还在启动中
        await new Promise(resolve => setTimeout(resolve, 2000));
      }
    }

    return false;
  }

  /**
   * 恢复已存在的容器
   */
  private async recoverExistingAgents(): Promise<void> {
    const containers = await this.docker.listContainers({
      all: true,
      filters: { label: ['finverse.agent=true'] }
    });

    for (const containerInfo of containers) {
      const userId = containerInfo.Labels['finverse.user_id'];
      
      const instance: AgentInstance = {
        userId,
        containerId: containerInfo.Id,
        state: containerInfo.State === 'running' ? AgentState.ACTIVE : AgentState.STOPPED,
        createdAt: new Date(containerInfo.Created * 1000),
        lastActiveAt: new Date(),
        resources: { memoryMB: 512, cpuCores: 1 }
      };

      this.agents.set(userId, instance);
      logger.info(`Recovered agent: ${userId}`);
    }
  }

  /**
   * 从数据库加载 Agent 配置
   */
  private async loadAgentConfig(userId: string): Promise<AgentConfig> {
    const result = await db.queryOne<any>(
      'SELECT api_keys, preferences, chat_channels FROM users WHERE id = $1',
      [userId]
    );

    return {
      userId,
      apiKeys: result.api_keys,
      preferences: result.preferences,
      chatChannels: result.chat_channels
    };
  }

  /**
   * 停止调度器
   */
  async stop(): Promise<void> {
    if (this.healthCheckInterval) {
      clearInterval(this.healthCheckInterval);
    }

    // 优雅停止所有 Agent
    const promises = Array.from(this.agents.keys()).map(userId =>
      this.stopAgent(userId, true).catch(err => 
        logger.error(`Failed to stop agent ${userId}:`, err)
      )
    );

    await Promise.all(promises);
    logger.info('Agent Orchestrator stopped');
  }
}

interface AgentInstance {
  userId: string;
  containerId: string;
  state: AgentState;
  createdAt: Date;
  lastActiveAt: Date;
  resources: {
    memoryMB: number;
    cpuCores: number;
  };
}
```

---

### 15.4 健康检查 API 实现

**创建文件：`apps/agent-runtime/src/health.ts`**

```typescript
import http from 'http';
import { db } from '@finverse/database';

/**
 * Agent 容器内健康检查服务
 * 
 * Docker HEALTHCHECK 会定期调用此端点
 */
export function startHealthCheckServer(port: number = 3100): http.Server {
  const server = http.createServer(async (req, res) => {
    if (req.url === '/health') {
      try {
        // 检查 1: 数据库连接
        await db.query('SELECT 1');

        // 检查 2: Redis 连接
        await db.redis.ping();

        // 检查 3: 内存使用
        const memUsage = process.memoryUsage();
        const heapUsedMB = memUsage.heapUsed / 1024 / 1024;
        
        if (heapUsedMB > 400) { // 超过 400 MB 报警
          throw new Error(`High memory usage: ${heapUsedMB.toFixed(2)} MB`);
        }

        // 检查 4: OpenClaw 进程存活
        if (!global.openclawInstance || !global.openclawInstance.isAlive()) {
          throw new Error('OpenClaw instance not alive');
        }

        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          status: 'healthy',
          timestamp: new Date().toISOString(),
          memory: {
            heapUsed: `${heapUsedMB.toFixed(2)} MB`,
            rss: `${(memUsage.rss / 1024 / 1024).toFixed(2)} MB`
          }
        }));

      } catch (error: any) {
        res.writeHead(503, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          status: 'unhealthy',
          error: error.message,
          timestamp: new Date().toISOString()
        }));
      }
    } else {
      res.writeHead(404);
      res.end();
    }
  });

  server.listen(port, () => {
    console.log(`Health check server listening on port ${port}`);
  });

  return server;
}
```

---

### 15.5 灰度发布脚本

**创建文件：`scripts/rolling-update.sh`**

```bash
#!/bin/bash
# ============================================
# Agent 容器滚动升级脚本
# 
# 用法: ./rolling-update.sh <new-image-tag> [batch-size] [delay-seconds]
# 例子: ./rolling-update.sh v1.2.0 10 30
# ============================================

set -euo pipefail

NEW_IMAGE_TAG="${1:-latest}"
BATCH_SIZE="${2:-10}"
DELAY_SECONDS="${3:-30}"

echo "🚀 Starting rolling update to finverse/agent:${NEW_IMAGE_TAG}"
echo "   Batch size: ${BATCH_SIZE}"
echo "   Delay between batches: ${DELAY_SECONDS}s"

# 1. 拉取新镜像
echo "📥 Pulling new image..."
docker pull "finverse/agent:${NEW_IMAGE_TAG}"

# 2. 获取所有 Agent 容器
AGENTS=$(docker ps --filter "label=finverse.agent=true" --format "{{.ID}}" | sort)
TOTAL=$(echo "$AGENTS" | wc -l | tr -d ' ')

echo "📊 Found ${TOTAL} agent containers"

if [ "$TOTAL" -eq 0 ]; then
  echo "✅ No agents to update"
  exit 0
fi

# 3. 批量升级
BATCH_NUM=1
COUNT=0

for CONTAINER_ID in $AGENTS; do
  COUNT=$((COUNT + 1))
  
  # 获取用户 ID
  USER_ID=$(docker inspect "$CONTAINER_ID" --format '{{index .Config.Labels "finverse.user_id"}}')
  
  echo "🔄 [$COUNT/$TOTAL] Updating agent for user $USER_ID ($CONTAINER_ID)"
  
  # 获取当前配置
  CURRENT_ENV=$(docker inspect "$CONTAINER_ID" --format '{{json .Config.Env}}')
  CURRENT_VOLUMES=$(docker inspect "$CONTAINER_ID" --format '{{json .HostConfig.Binds}}')
  
  # 停止旧容器
  echo "   ⏸  Stopping old container..."
  docker stop "$CONTAINER_ID" >/dev/null 2>&1 || true
  
  # 启动新容器 (使用相同配置)
  echo "   ▶️  Starting new container..."
  NEW_CONTAINER_ID=$(docker run -d \
    --name "agent-${USER_ID}-new" \
    --label "finverse.agent=true" \
    --label "finverse.user_id=${USER_ID}" \
    --env-file <(echo "$CURRENT_ENV" | jq -r '.[]') \
    --volume "agent-${USER_ID}-data:/app/workspace" \
    --network agent-network \
    --memory 512m \
    --cpus 1 \
    "finverse/agent:${NEW_IMAGE_TAG}")
  
  # 等待健康检查
  echo "   🏥 Waiting for health check..."
  MAX_RETRIES=15
  RETRY=0
  
  while [ $RETRY -lt $MAX_RETRIES ]; do
    HEALTH=$(docker inspect "$NEW_CONTAINER_ID" --format '{{.State.Health.Status}}' 2>/dev/null || echo "starting")
    
    if [ "$HEALTH" = "healthy" ]; then
      echo "   ✅ Agent healthy"
      break
    fi
    
    RETRY=$((RETRY + 1))
    sleep 2
  done
  
  if [ "$HEALTH" != "healthy" ]; then
    echo "   ❌ Health check failed, rolling back..."
    docker stop "$NEW_CONTAINER_ID" >/dev/null 2>&1 || true
    docker rm "$NEW_CONTAINER_ID" >/dev/null 2>&1 || true
    docker start "$CONTAINER_ID" >/dev/null 2>&1
    
    echo "   ⚠️  Rollback completed for $USER_ID"
    continue
  fi
  
  # 删除旧容器
  docker rm "$CONTAINER_ID" >/dev/null 2>&1 || true
  
  # 重命名新容器
  docker rename "$NEW_CONTAINER_ID" "agent-${USER_ID}" >/dev/null 2>&1 || true
  
  # 批次延迟
  if [ $((COUNT % BATCH_SIZE)) -eq 0 ] && [ $COUNT -lt $TOTAL ]; then
    echo "⏳ Batch ${BATCH_NUM} completed. Waiting ${DELAY_SECONDS}s before next batch..."
    sleep "$DELAY_SECONDS"
    BATCH_NUM=$((BATCH_NUM + 1))
  fi
done

echo ""
echo "🎉 Rolling update completed!"
echo "   Total agents: ${TOTAL}"
echo "   New image: finverse/agent:${NEW_IMAGE_TAG}"
```

---

## 十六、路线图 - Sprint 任务拆解

### Month 1: 核心基础

#### Sprint 1.1: 用户系统与认证 (Week 1)

**任务 1.1.1: 用户注册与登录**
- 实现 JWT 认证
- 密码哈希 (bcrypt)
- Email 验证流程
- OAuth 登录 (Google/GitHub)

*验收标准*:
- [ ] 用户可通过 email/password 注册
- [ ] 登录返回有效 JWT token (30 天过期)
- [ ] 注册邮件发送成功率 > 95%
- [ ] OAuth 登录流程完整 (重定向正确)

**任务 1.1.2: API Key 管理**
- 加密存储实现 (AES-256-GCM)
- CRUD API 端点
- API key 验证中间件

*验收标准*:
- [ ] 用户可添加 OpenAI/Anthropic/DeepSeek API key
- [ ] 存储的 key 已加密 (数据库中不可见明文)
- [ ] 解密后的 key 可成功调用对应 API
- [ ] 支持更新/删除 key

**任务 1.1.3: 偏好测试系统**
- 设计 8-10 个情景题
- 前端交互界面 (游戏化)
- 标签算法实现

*验收标准*:
- [ ] 测试时长 < 3 分钟
- [ ] 生成至少 5 个准确标签 (trading_style, risk_preference, etc.)
- [ ] 标签保存到用户 profile
- [ ] UI 体验流畅,无卡顿

---

#### Sprint 1.2: OpenClaw Agent 实例化 (Week 2)

**任务 1.2.1: Agent 调度器基础**
- Docker API 集成
- 容器创建/停止逻辑
- Agent 实例数据库表

*验收标准*:
- [ ] 可通过 API 触发创建 Agent 容器
- [ ] 容器成功启动并通过健康检查
- [ ] 数据库记录 Agent 状态 (user_id, container_id, state)
- [ ] 容器资源限制生效 (512MB memory, 1 CPU)

**任务 1.2.2: Agent 初始化流程**
- 生成 SOUL.md (基于用户标签)
- 注入 API keys (环境变量)
- Cron 任务预配置

*验收标准*:
- [ ] SOUL.md 包含用户偏好 (从标签生成)
- [ ] Agent 可成功调用用户的 AI API
- [ ] 预设 cron 任务已配置 (daily-summary, market-scan)

**任务 1.2.3: 聊天软件连接 (Telegram)**
- Telegram Bot API 集成
- 用户扫码绑定流程
- 消息路由 (Telegram <-> Agent)

*验收标准*:
- [ ] 用户扫码后收到 Agent 欢迎消息
- [ ] 可在 Telegram 发送消息与 Agent 对话
- [ ] Agent 回复延迟 < 5 秒
- [ ] 支持 inline keyboard (快捷操作)

---

#### Sprint 1.3: 基础数据接入 (Week 3)

**任务 1.3.1: CoinGecko 适配器**
- 实现 `getHistoricalPrices()`
- 实现 `getCurrentPrice()`
- 速率限制器
- 错误处理与重试

*验收标准*:
- [ ] 可获取 BTC/ETH 历史价格 (最近 90 天)
- [ ] 实时价格延迟 < 10 秒
- [ ] 速率限制生效 (免费版 10 calls/min)
- [ ] 网络错误自动重试 (最多 3 次)

**任务 1.3.2: Yahoo Finance 适配器**
- 使用 `yahoo-finance2` 库
- 获取美股数据 (AAPL, TSLA, SPY, etc.)
- 数据标准化

*验收标准*:
- [ ] 可获取 AAPL 历史 OHLCV 数据
- [ ] 数据格式符合 `PriceDataPoint` schema
- [ ] 支持 1d/1h 时间粒度

**任务 1.3.3: 定时数据拉取**
- Cron 任务配置 (每 5 分钟实时价格)
- 数据存储 (PostgreSQL + Redis)
- 监控与日志

*验收标准*:
- [ ] Cron 准时执行 (误差 < 10 秒)
- [ ] 数据成功写入 PostgreSQL
- [ ] Redis 缓存最新价格 (5 分钟 TTL)
- [ ] 失败时有日志记录

---

#### Sprint 1.4: 可视化界面 (Week 4)

**任务 1.4.1: 摘要模式**
- 设计 UI (Figma/直接代码)
- 实现一句话结论卡片
- 多维信号条 (链上/宏观/技术/情绪)
- 异常预警卡片

*验收标准*:
- [ ] 打开页面 1 秒内看到核心信息
- [ ] 信号条颜色正确 (红=看跌,绿=看涨,黄=中性)
- [ ] 异常预警按严重程度排序
- [ ] 移动端自适应

**任务 1.4.2: 图表模式 (基础)**
- 集成 `lightweight-charts`
- 显示 K 线图
- 基础交互 (缩放,平移)

*验收标准*:
- [ ] K 线图渲染流畅 (60fps)
- [ ] 支持 1000+ 数据点不卡顿
- [ ] 可切换时间粒度 (1d/1h/5m)
- [ ] 十字光标显示 OHLCV 数据

**任务 1.4.3: 数据模式**
- 高密度数字面板
- 实时价格更新 (WebSocket)

*验收标准*:
- [ ] 显示至少 20 个数据点 (价格/成交量/持仓/费率等)
- [ ] WebSocket 连接稳定
- [ ] 价格变动时数字闪烁提示
- [ ] 延迟 < 2 秒

---

### Month 2: 公域 + 异常检测

#### Sprint 2.1: 信号系统 (Week 5)

**任务 2.1.1: 信号标准格式**
- 定义 JSON Schema
- 验证器实现
- 数据库表设计

*验收标准*:
- [ ] Schema 包含所有必需字段 (agent_id, asset, signal, confidence, dimensions)
- [ ] 无效信号会被拒绝 (验证器)
- [ ] 数据库索引优化 (按 asset + timestamp 查询)

**任务 2.1.2: 信号发布 API**
- POST `/api/signals` 端点
- Agent 调用示例
- 速率限制 (防刷分)

*验收标准*:
- [ ] Agent 可成功发布信号
- [ ] 信号存储到数据库
- [ ] 速率限制: 每 Agent 最多 10 signals/hour
- [ ] API 响应时间 < 500ms

**任务 2.1.3: 信号订阅与聚合**
- 订阅逻辑 (按用户偏好)
- 信号聚合算法 (加权平均)
- 共识热力图计算

*验收标准*:
- [ ] Agent 可获取相关资产的信号 (top 20)
- [ ] 聚合结果准确 (手动验证 3 个案例)
- [ ] 热力图数据格式正确 (percentage, tier分布)

---

#### Sprint 2.2: 异常检测 (Week 6)

**任务 2.2.1: Spam 检测**
- 1 小时内信号数量检测
- 连续重复检测

*验收标准*:
- [ ] 1 小时 > 10 信号触发 anomaly flag
- [ ] 连续 5 个相同信号触发 flag
- [ ] Flag 记录到数据库 (anomaly_flags 表)

**任务 2.2.2: 矛盾检测**
- 信号 vs 实际走势比对
- 置信度惩罚

*验收标准*:
- [ ] 高置信度错误信号被标记
- [ ] 检测延迟 < 5 分钟 (信号发布后)

**任务 2.2.3: 信誉评分 v1**
- 实现 `ReputationEngine`
- 准确率计算
- 等级划分 (novice/reliable/expert/master)

*验收标准*:
- [ ] 评分算法通过单元测试 (10+ test cases)
- [ ] 新 Agent 默认 50 分
- [ ] 准确率 90% Agent 评分 > 80

---

#### Sprint 2.3: AI 推理链可视化 (Week 7)

**任务 2.3.1: 多图层系统**
- AI 标注层
- 宏观事件层
- 链上数据层

*验收标准*:
- [ ] 每个图层可独立开关
- [ ] 标注点悬停显示详情
- [ ] 图层叠加不影响性能 (60fps)

**任务 2.3.2: 推理链展示**
- 步骤分解 UI
- 权重可视化

*验收标准*:
- [ ] 点击标注点展开推理步骤
- [ ] 显示每步权重百分比
- [ ] 权重总和 = 100%

---

#### Sprint 2.4: 监控脚本基础设施 (Week 8)

**任务 2.4.1: Python 监控脚本**
- 价格异常检测 (>5% 突变)
- 成交量异常
- 链上异常 (需 Glassnode)

*验收标准*:
- [ ] 脚本持续运行 (systemd service)
- [ ] 检测到异常 < 1 分钟
- [ ] 异常记录到 Redis

**任务 2.4.2: Agent 唤醒机制**
- 监控脚本 -> 调度器 API
- 调度器唤醒对应 Agent
- Agent 分析 -> 推送用户

*验收标准*:
- [ ] 异常触发到用户收到通知 < 3 分钟
- [ ] 只唤醒相关 Agent (按持仓/关注列表)
- [ ] 通知内容包含异常详情

---

### Month 3: 私域 + 市场扩展

#### Sprint 3.1: 异步分析小组 (Week 9-10)

**任务 3.1.1: 小组功能基础**
- 数据库设计 (groups, group_members, analyses)
- CRUD API
- 权限控制 (创建者/成员/公开)

*验收标准*:
- [ ] 用户可创建小组 (名称/描述/公开性)
- [ ] 可邀请成员 (via invite link)
- [ ] Pro 用户可创建私密小组
- [ ] 免费用户只能加入公开小组

**任务 3.1.2: AI 辅助分析发布**
- 自然语言输入
- Agent 结构化呈现
- 图表自动生成

*验收标准*:
- [ ] 用户说 "我觉得 BTC 要跌" -> Agent 生成完整分析
- [ ] 分析包含: 标题/摘要/数据支撑/图表/结论
- [ ] 生成时间 < 10 秒

**任务 3.1.3: 评论与反馈**
- 点赞/点踩
- 评论 (AI 辅助)
- @提及

*验收标准*:
- [ ] 可对分析点赞/点踩
- [ ] 评论支持 markdown
- [ ] @member 发送通知

**任务 3.1.4: 共识报告**
- 每日/每主题自动生成
- 汇总看多/看空比例
- 分歧点分析

*验收标准*:
- [ ] 每天 UTC 00:00 自动生成报告
- [ ] 报告准确反映小组共识
- [ ] 推送到所有成员 (Telegram/网站)

---

#### Sprint 3.2: 历史复盘 (Week 11)

**任务 3.2.1: 判断准确率追踪**
- 信号验证逻辑 (24h/48h/1w 后)
- 个人准确率统计
- 可视化趋势图

*验收标准*:
- [ ] 用户可查看自己的准确率 (按时间框架)
- [ ] 显示最近 30 天趋势图
- [ ] 数据隐私 (只显示给自己)

**任务 3.2.2: 维度分析**
- 哪些维度最有效
- 错误案例回顾

*验收标准*:
- [ ] 显示 "你的链上分析准确率 72%,技术分析 58%"
- [ ] 列出 3 个最大失误案例

---

#### Sprint 3.3: 市场扩展 (Week 12)

**任务 3.3.1: 美股数据**
- Polygon.io 适配器 (付费,用户 key)
- 扩展到 500+ 股票
- 期权数据 (可选)

*验收标准*:
- [ ] 支持 SPY/QQQ/AAPL/TSLA 等主流股票
- [ ] 数据延迟 < 15 分钟 (免费 tier)
- [ ] 用户可添加 Polygon.io key 获取实时数据

**任务 3.3.2: 外汇数据**
- Twelve Data 适配器
- 主流货币对 (EUR/USD, GBP/USD, etc.)

*验收标准*:
- [ ] 支持 10+ 货币对
- [ ] 数据粒度: 1h/4h/1d

**任务 3.3.3: 贵金属**
- 黄金/白银数据
- Kitco API (免费)

*验收标准*:
- [ ] 显示 XAU/USD, XAG/USD 实时价格
- [ ] 历史数据至少 1 年

---

#### Sprint 3.4: Pro 版订阅 (Week 12)

**任务 3.4.1: Stripe 集成**
- 订阅计划配置 ($29/月)
- 支付流程
- Webhook 处理

*验收标准*:
- [ ] 用户可成功订阅 Pro 版
- [ ] 支付成功后立即解锁功能
- [ ] 取消订阅流程正常

**任务 3.4.2: 功能门控**
- 小组数量限制
- 高级可视化
- Agent 自定义

*验收标准*:
- [ ] 免费用户: 1 个小组,基础图表
- [ ] Pro 用户: 无限小组,高级图层,自定义 SOUL.md

---

## 十七、技术栈 - 完整配置

### 17.1 技术选型与版本

```json
{
  "frontend": {
    "framework": "Next.js 15.1.0",
    "runtime": "React 19.0.0",
    "language": "TypeScript 5.7.0",
    "styling": "Tailwind CSS 4.0.0",
    "charts": "lightweight-charts 4.2.0",
    "state": "Zustand 5.0.0",
    "forms": "React Hook Form 7.54.0",
    "http": "Axios 1.7.0"
  },
  "backend": {
    "runtime": "Node.js 22.22.0",
    "framework": "Fastify 5.2.0",
    "language": "TypeScript 5.7.0",
    "validation": "Zod 3.24.0",
    "orm": "Drizzle ORM 0.37.0"
  },
  "database": {
    "primary": "PostgreSQL 16.3",
    "cache": "Redis 7.4",
    "search": "MeiliSearch 1.11.0 (可选)"
  },
  "agent": {
    "runtime": "OpenClaw (latest)",
    "base": "Node.js 22.22.0",
    "scripting": "Python 3.12.0"
  },
  "infrastructure": {
    "containerization": "Docker 27.4.0",
    "orchestration": "Docker Compose 2.31.0",
    "proxy": "Nginx 1.27.3",
    "monitoring": "Prometheus 3.0.0 + Grafana 11.4.0"
  }
}
```

---

### 17.2 package.json 依赖列表

**创建文件：`package.json` (monorepo root)**

```json
{
  "name": "finverse",
  "version": "1.0.0",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "format": "prettier --write \"**/*.{ts,tsx,js,jsx,json,md}\"",
    "docker:build": "docker-compose build",
    "docker:up": "docker-compose up -d",
    "docker:down": "docker-compose down",
    "migrate": "drizzle-kit migrate",
    "migrate:generate": "drizzle-kit generate"
  },
  "devDependencies": {
    "@types/node": "^22.10.0",
    "turbo": "^2.3.0",
    "prettier": "^3.4.0",
    "typescript": "^5.7.0",
    "drizzle-kit": "^0.29.0"
  }
}
```

**创建文件：`apps/web/package.json`**

```json
{
  "name": "@finverse/web",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "^15.1.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "typescript": "^5.7.0",
    "@types/react": "^19.0.0",
    "@types/node": "^22.10.0",
    "tailwindcss": "^4.0.0",
    "autoprefixer": "^10.4.20",
    "postcss": "^8.4.49",
    "lightweight-charts": "^4.2.0",
    "zustand": "^5.0.0",
    "react-hook-form": "^7.54.0",
    "zod": "^3.24.0",
    "axios": "^1.7.0",
    "lucide-react": "^0.468.0",
    "date-fns": "^4.1.0",
    "@radix-ui/react-dialog": "^1.1.2",
    "@radix-ui/react-dropdown-menu": "^2.1.2",
    "@radix-ui/react-tabs": "^1.1.1"
  }
}
```

**创建文件：`apps/api/package.json`**

```json
{
  "name": "@finverse/api",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "fastify": "^5.2.0",
    "@fastify/cors": "^10.0.1",
    "@fastify/jwt": "^9.0.1",
    "@fastify/websocket": "^11.0.1",
    "drizzle-orm": "^0.37.0",
    "postgres": "^3.4.5",
    "ioredis": "^5.4.1",
    "zod": "^3.24.0",
    "bcrypt": "^5.1.1",
    "@types/bcrypt": "^5.0.2",
    "nanoid": "^5.0.9",
    "pino": "^9.5.0",
    "pino-pretty": "^13.0.0"
  },
  "devDependencies": {
    "tsx": "^4.19.2",
    "typescript": "^5.7.0",
    "@types/node": "^22.10.0"
  }
}
```

**创建文件：`packages/data-adapters/package.json`**

```json
{
  "name": "@finverse/data-adapters",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test": "vitest"
  },
  "dependencies": {
    "axios": "^1.7.0",
    "yahoo-finance2": "^2.13.2",
    "@finverse/data-schema": "workspace:*"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "vitest": "^2.1.0"
  }
}
```

---

### 17.3 Dockerfile 配置

**创建文件：`apps/api/Dockerfile`**

```dockerfile
# ============================================
# Multi-stage build for production API
# ============================================

FROM node:22-alpine AS base
RUN apk add --no-cache libc6-compat
WORKDIR /app

# ============================================
# Dependencies stage
# ============================================
FROM base AS deps
COPY package*.json ./
COPY apps/api/package*.json ./apps/api/
COPY packages/*/package*.json ./packages/

RUN npm ci --production=false

# ============================================
# Builder stage
# ============================================
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build API and shared packages
RUN npm run build --workspace=@finverse/api

# ============================================
# Production runner
# ============================================
FROM base AS runner

ENV NODE_ENV=production

RUN addgroup -g 1001 -S nodejs && \
    adduser -S apiuser -u 1001

COPY --from=builder --chown=apiuser:nodejs /app/apps/api/dist ./dist
COPY --from=builder --chown=apiuser:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=apiuser:nodejs /app/package.json ./

USER apiuser

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3001/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"

CMD ["node", "dist/index.js"]
```

**创建文件：`apps/web/Dockerfile`**

```dockerfile
FROM node:22-alpine AS base

# ============================================
# Dependencies
# ============================================
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

COPY package*.json ./
COPY apps/web/package*.json ./apps/web/
RUN npm ci

# ============================================
# Builder
# ============================================
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL

RUN npm run build --workspace=@finverse/web

# ============================================
# Runner
# ============================================
FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production

RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

COPY --from=builder /app/apps/web/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

---

### 17.4 Nginx 配置

**创建文件：`nginx/nginx.conf`**

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 2048;
    use epoll;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 20M;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;

    # Upstream servers
    upstream api_backend {
        least_conn;
        server api:3001 max_fails=3 fail_timeout=30s;
    }

    upstream web_backend {
        least_conn;
        server web:3000 max_fails=3 fail_timeout=30s;
    }

    # HTTP -> HTTPS redirect
    server {
        listen 80;
        server_name finverse.ai www.finverse.ai;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$server_name$request_uri;
        }
    }

    # HTTPS server
    server {
        listen 443 ssl http2;
        server_name finverse.ai www.finverse.ai;

        # SSL certificates (Let's Encrypt)
        ssl_certificate /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_session_tickets off;

        # Modern SSL configuration
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
        ssl_prefer_server_ciphers off;

        # HSTS
        add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;

        # API routes
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            
            proxy_pass http://api_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
            
            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # WebSocket for real-time data
        location /ws {
            proxy_pass http://api_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_read_timeout 86400;
        }

        # Login endpoint (stricter rate limit)
        location /api/auth/login {
            limit_req zone=login_limit burst=3 nodelay;
            
            proxy_pass http://api_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # Next.js frontend
        location / {
            proxy_pass http://web_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }

        # Next.js static files (cache aggressively)
        location /_next/static/ {
            proxy_pass http://web_backend;
            proxy_cache_valid 200 365d;
            add_header Cache-Control "public, max-age=31536000, immutable";
        }

        # Public assets
        location /assets/ {
            proxy_pass http://web_backend;
            expires 7d;
            add_header Cache-Control "public, max-age=604800";
        }
    }
}
```

---

### 17.5 SSL 证书配置 (Let's Encrypt)

**创建文件：`scripts/setup-ssl.sh`**

```bash
#!/bin/bash
# ============================================
# Let's Encrypt SSL 证书自动配置
# 
# 前提:
# - 域名已解析到服务器 IP
# - 80/443 端口开放
# ============================================

set -euo pipefail

DOMAIN="finverse.ai"
EMAIL="admin@finverse.ai"

echo "🔒 Setting up SSL for ${DOMAIN}"

# 1. 安装 Certbot
if ! command -v certbot &> /dev/null; then
    echo "📦 Installing Certbot..."
    apt-get update
    apt-get install -y certbot python3-certbot-nginx
fi

# 2. 获取证书
echo "📜 Obtaining SSL certificate..."
certbot certonly --nginx \
    -d "${DOMAIN}" \
    -d "www.${DOMAIN}" \
    --email "${EMAIL}" \
    --agree-tos \
    --non-interactive

# 3. 复制证书到 Nginx 目录
mkdir -p ./nginx/ssl
cp /etc/letsencrypt/live/${DOMAIN}/fullchain.pem ./nginx/ssl/
cp /etc/letsencrypt/live/${DOMAIN}/privkey.pem ./nginx/ssl/

echo "✅ SSL certificate installed"

# 4. 设置自动续期
echo "⏰ Setting up auto-renewal..."
(crontab -l 2>/dev/null; echo "0 0 * * * certbot renew --quiet --deploy-hook 'docker exec finverse-nginx nginx -s reload'") | crontab -

echo "🎉 SSL setup complete!"
echo "   Certificate: ${DOMAIN}"
echo "   Expiry: 90 days"
echo "   Auto-renewal: enabled"
```

---

### 17.6 项目初始化完整命令序列

**创建文件：`scripts/init-project.sh`**

```bash
#!/bin/bash
# ============================================
# FinVerse 项目完整初始化脚本
# 
# 从零开始到第一次部署的所有命令
# ============================================

set -euo pipefail

echo "🚀 Initializing FinVerse project..."

# ============================================
# 1. Git 初始化
# ============================================
echo "📦 [1/10] Initializing Git repository..."
git init
git add .
git commit -m "Initial commit: FinVerse project structure"

# 创建 .gitignore
cat > .gitignore << 'EOF'
# Dependencies
node_modules/
.pnp
.pnp.js

# Build outputs
dist/
build/
.next/
out/

# Environment variables
.env
.env.local
.env.*.local

# Logs
logs/
*.log
npm-debug.log*

# OS files
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.swp

# Docker
.dockerignore

# SSL
nginx/ssl/*.pem
EOF

git add .gitignore
git commit -m "Add .gitignore"

# ============================================
# 2. 环境变量配置
# ============================================
echo "🔐 [2/10] Setting up environment variables..."

# 生成主加密密钥
MASTER_KEY=$(openssl rand -hex 32)

cat > .env << EOF
# Database
POSTGRES_PASSWORD=$(openssl rand -base64 32)
REDIS_PASSWORD=$(openssl rand -base64 32)

# Application
JWT_SECRET=$(openssl rand -base64 64)
MASTER_ENCRYPTION_KEY=${MASTER_KEY}
API_URL=https://api.finverse.ai

# Monitoring
GRAFANA_PASSWORD=$(openssl rand -base64 16)

# Agent Orchestrator
MAX_AGENTS=1000

# Optional: Production settings
# NODE_ENV=production
# DATABASE_URL=postgresql://...
# REDIS_URL=redis://...
EOF

echo "✅ .env file created with secure random values"
echo "⚠️  IMPORTANT: Back up your MASTER_ENCRYPTION_KEY securely!"

# ============================================
# 3. 安装依赖
# ============================================
echo "📥 [3/10] Installing dependencies..."
npm install

# ============================================
# 4. 数据库初始化
# ============================================
echo "🗄️  [4/10] Setting up database schema..."

# 创建迁移文件目录
mkdir -p migrations

# 生成数据库 schema
cat > migrations/001_initial_schema.sql << 'EOSQL'
-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- User profiles
CREATE TABLE user_profiles (
  user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  preferences JSONB DEFAULT '{}'::jsonb,
  tags TEXT[] DEFAULT ARRAY[]::TEXT[],
  chat_channels TEXT[] DEFAULT ARRAY[]::TEXT[],
  subscription_tier VARCHAR(20) DEFAULT 'free',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- API keys (encrypted)
CREATE TABLE user_api_keys (
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  service VARCHAR(50) NOT NULL,
  encrypted_key TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_used_at TIMESTAMPTZ,
  PRIMARY KEY (user_id, service)
);

-- Agent instances
CREATE TABLE agent_instances (
  user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  container_id VARCHAR(255) NOT NULL,
  state VARCHAR(20) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_active_at TIMESTAMPTZ DEFAULT NOW(),
  stopped_at TIMESTAMPTZ
);

CREATE INDEX idx_agents_state ON agent_instances(state);

-- Price data
CREATE TABLE price_data (
  symbol VARCHAR(20) NOT NULL,
  timestamp TIMESTAMPTZ NOT NULL,
  open NUMERIC(20, 8),
  high NUMERIC(20, 8),
  low NUMERIC(20, 8),
  close NUMERIC(20, 8) NOT NULL,
  volume NUMERIC(30, 8),
  source VARCHAR(50) NOT NULL,
  PRIMARY KEY (symbol, timestamp, source)
);

CREATE INDEX idx_price_symbol_time ON price_data(symbol, timestamp DESC);

-- On-chain data
CREATE TABLE onchain_data (
  asset VARCHAR(10) NOT NULL,
  timestamp TIMESTAMPTZ NOT NULL,
  exchange_inflow NUMERIC(20, 8),
  exchange_outflow NUMERIC(20, 8),
  whale_transactions INT,
  active_addresses INT,
  source VARCHAR(50) NOT NULL,
  PRIMARY KEY (asset, timestamp)
);

-- Signals
CREATE TABLE signals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID NOT NULL REFERENCES users(id),
  asset VARCHAR(20) NOT NULL,
  signal VARCHAR(10) NOT NULL CHECK (signal IN ('bullish', 'bearish', 'neutral')),
  confidence NUMERIC(3, 2) NOT NULL CHECK (confidence >= 0 AND confidence <= 1),
  dimensions JSONB NOT NULL,
  timeframe VARCHAR(10) NOT NULL,
  timestamp TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_signals_asset_time ON signals(asset, timestamp DESC);
CREATE INDEX idx_signals_agent ON signals(agent_id);

-- Anomaly flags
CREATE TABLE anomaly_flags (
  id SERIAL PRIMARY KEY,
  agent_id UUID NOT NULL REFERENCES users(id),
  type VARCHAR(50) NOT NULL,
  details TEXT,
  timestamp TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_anomaly_agent ON anomaly_flags(agent_id);

-- Reputation scores
CREATE TABLE reputation_scores (
  agent_id UUID PRIMARY KEY REFERENCES users(id),
  total_score NUMERIC(5, 2) NOT NULL,
  breakdown JSONB NOT NULL,
  tier VARCHAR(20) NOT NULL,
  last_updated TIMESTAMPTZ DEFAULT NOW()
);

-- Groups
CREATE TABLE groups (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  description TEXT,
  creator_id UUID NOT NULL REFERENCES users(id),
  is_public BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_groups_public ON groups(is_public) WHERE is_public = true;

-- Group members
CREATE TABLE group_members (
  group_id UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role VARCHAR(20) DEFAULT 'member',
  joined_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (group_id, user_id)
);

-- Analyses
CREATE TABLE analyses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  group_id UUID NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
  author_id UUID NOT NULL REFERENCES users(id),
  title VARCHAR(200) NOT NULL,
  content JSONB NOT NULL,
  upvotes INT DEFAULT 0,
  downvotes INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_analyses_group_time ON analyses(group_id, created_at DESC);

-- Audit log
CREATE TABLE api_key_audit (
  id SERIAL PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  service VARCHAR(50) NOT NULL,
  action VARCHAR(20) NOT NULL,
  ip_address INET,
  timestamp TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_user_time ON api_key_audit(user_id, timestamp DESC);
EOSQL

echo "✅ Database schema created"

# ============================================
# 5. 构建 Docker 镜像
# ============================================
echo "🐳 [5/10] Building Docker images..."
docker-compose build

# ============================================
# 6. 启动基础服务
# ============================================
echo "▶️  [6/10] Starting core services..."
docker-compose up -d postgres redis

# 等待数据库就绪
echo "⏳ Waiting for PostgreSQL..."
until docker exec finverse-postgres pg_isready -U finverse > /dev/null 2>&1; do
  sleep 1
done

# 运行迁移
echo "🔄 Running database migrations..."
docker exec -i finverse-postgres psql -U finverse -d finverse < migrations/001_initial_schema.sql

# ============================================
# 7. 启动应用服务
# ============================================
echo "🚀 [7/10] Starting application services..."
docker-compose up -d api web agent-orchestrator

# ============================================
# 8. SSL 证书设置 (可选,生产环境)
# ============================================
read -p "📜 Setup SSL certificate? (y/N): " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo "🔒 [8/10] Setting up SSL..."
    ./scripts/setup-ssl.sh
else
    echo "⏭️  [8/10] Skipped SSL setup"
fi

# ============================================
# 9. 启动 Nginx
# ============================================
echo "🌐 [9/10] Starting Nginx..."
docker-compose up -d nginx

# ============================================
# 10. 健康检查
# ============================================
echo "🏥 [10/10] Running health checks..."

services=("postgres" "redis" "api" "web")
for service in "${services[@]}"; do
  echo -n "   Checking $service... "
  
  max_attempts=30
  attempt=0
  
  while [ $attempt -lt $max_attempts ]; do
    if docker-compose ps | grep "$service" | grep -q "Up"; then
      echo "✅"
      break
    fi
    
    attempt=$((attempt + 1))
    sleep 1
  done
  
  if [ $attempt -eq $max_attempts ]; then
    echo "❌ Failed"
  fi
done

# ============================================
# 完成
# ============================================
echo ""
echo "🎉 FinVerse initialization complete!"
echo ""
echo "📋 Next steps:"
echo "   1. Review .env file and update any necessary values"
echo "   2. Access the web app: http://localhost:3000"
echo "   3. Access API: http://localhost:3001"
echo "   4. Grafana monitoring: http://localhost:3003 (admin / [from .env])"
echo ""
echo "📚 Useful commands:"
echo "   View logs:        docker-compose logs -f [service]"
echo "   Stop all:         docker-compose down"
echo "   Restart service:  docker-compose restart [service]"
echo "   Run migrations:   docker exec -i finverse-postgres psql -U finverse -d finverse < migrations/[file].sql"
echo ""
echo "⚠️  IMPORTANT: Backup your .env file securely!"
```

**使用方式：**

```bash
# 1. 克隆仓库或创建项目目录
mkdir finverse && cd finverse

# 2. 复制所有项目文件到此目录

# 3. 赋予脚本执行权限
chmod +x scripts/*.sh

# 4. 运行初始化
./scripts/init-project.sh

# 5. 首次部署完成！
```

---

## 十八、一句话总结 - 项目核心价值主张

### 18.1 技术视角总结

> **FinVerse 是基于 OpenClaw Agent 运行时的金融数据可视化与协作平台。用户自带 AI 和数据的钥匙（API keys），平台提供零成本的 Agent 托管、结构化信号广播、异步分析小组和多维可视化界面。在 AI 个性化界面到来之前的 5-10 年窗口期，抓住「信息呈现设计」这个核心不可替代价值。**

### 18.2 商业视角总结

> **平台模式 + 用户自费 AI = 超低边际成本。不卖数据、不卖 AI，卖的是连接、呈现和社区。从加密货币切入（监管宽松），逐步扩展到美股、外汇、贵金属。免费版吸引用户，Pro 版（$29/月）变现，团队版（$79/月）服务专业交易者。**

### 18.3 用户视角总结

> **我的 AI Agent 活在 Telegram 里，每天推送市场摘要和异常预警。想深入分析时，点链接跳到网站看可视化图表和 AI 推理链。加入小组与同风格的人异步协作，AI 帮我把想法变成结构化分析。每周自动复盘我的判断准确率，帮助我持续进步。**

### 18.4 技术架构总结

```
用户 API key (自费 AI)
    ↓
OpenClaw Agent 容器 (512MB/1CPU,自动扩缩容)
    ↓
多数据源聚合 (CoinGecko/Yahoo/Glassnode/...)
    ↓
结构化信号池 (Agent 间广播订阅)
    ↓
可视化层 (摘要/图表/数据三模式 + 多图层推理链)
    ↓
异步协作小组 (AI 辅助发布 + 共识报告 + 历史复盘)
    ↓
用户聊天软件 (Telegram/WhatsApp/Discord)
```

### 18.5 差异化竞争总结

| 维度 | 传统金融网站 | FinVerse |
|------|-------------|---------|
| 数据来源 | 自有/买断 | 用户自带 API key |
| AI 分析 | 无或平台统一 | 每用户专属 Agent |
| 成本结构 | 高（数据+AI） | 低（仅基础设施） |
| 社区形态 | 论坛/评论 | AI 驱动的结构化协作 |
| 可视化 | 静态图表 | AI 推理链 + 多图层叠加 |
| 个性化 | 有限 | 完全（Agent 记忆用户偏好） |

### 18.6 风险与对策总结

| 风险 | 概率 | 影响 | 对策 |
|------|------|------|------|
| AI 互相欺骗 | 中 | 高 | 信誉系统 + 异常检测 + 交叉验证 |
| 合规雷区 | 高 | 极高 | 从加密切入 + 免责声明 + 不做投顾 |
| 冷启动 | 高 | 中 | 官方 Agent 保底 + 公域信号质量先行 |
| 社交从众亏钱 | 中 | 高 | AI 去情绪化 + 多方观点呈现 + 复盘反思 |
| API key 泄露 | 低 | 极高 | AES-256-GCM 加密 + 审计日志 |

### 18.7 成功标准（3个月MVP）

- **技术指标**:
  - [ ] Agent 创建成功率 > 99%
  - [ ] 健康检查通过率 > 99.5%
  - [ ] API 响应时间 p95 < 500ms
  - [ ] 数据拉取成功率 > 98%

- **产品指标**:
  - [ ] 注册到首次 Agent 交互 < 5 分钟
  - [ ] 可视化页面加载 < 2 秒
  - [ ] 异常检测到用户通知 < 3 分钟
  - [ ] 小组分析发布 < 10 秒

- **用户指标**:
  - [ ] 100+ Beta 测试用户
  - [ ] 日活 Agent > 50 个
  - [ ] 公域信号 > 1000 条/天
  - [ ] 至少 10 个活跃小组

- **商业指标**:
  - [ ] Pro 版转化率 > 5%
  - [ ] 用户留存（7天）> 40%
  - [ ] NPS > 50

---

## 附录：快速参考

### 关键命令速查

```bash
# 项目初始化
./scripts/init-project.sh

# 开发环境
npm run dev                    # 启动所有服务（开发模式）
docker-compose up -d           # 启动生产环境

# 数据库
npm run migrate                # 运行迁移
npm run migrate:generate       # 生成迁移文件

# Docker 管理
docker-compose logs -f api     # 查看 API 日志
docker-compose restart web     # 重启前端
docker ps | grep agent-        # 查看所有 Agent 容器

# Agent 管理
./scripts/rolling-update.sh v1.2.0 10 30  # 滚动升级 Agent

# 监控
http://localhost:9090          # Prometheus
http://localhost:3003          # Grafana
```

### 重要文件路径

```
finverse/
├── apps/
│   ├── api/                  # 后端 API
│   ├── web/                  # 前端应用
│   ├── agent-runtime/        # Agent 运行时
│   └── agent-orchestrator/   # Agent 调度器
├── packages/
│   ├── data-schema/          # 数据类型定义
│   ├── data-adapters/        # 数据源适配器
│   ├── security/             # 加密服务
│   └── reputation/           # 信誉系统
├── migrations/               # 数据库迁移
├── nginx/                    # Nginx 配置
├── scripts/                  # 运维脚本
└── docker-compose.yml        # 容器编排
```

### 环境变量清单

```bash
# 必需
POSTGRES_PASSWORD=xxx
REDIS_PASSWORD=xxx
JWT_SECRET=xxx
MASTER_ENCRYPTION_KEY=xxx  # 32 字节 hex

# 生产环境
API_URL=https://api.finverse.ai
NODE_ENV=production

# 可选
MAX_AGENTS=1000
GRAFANA_PASSWORD=xxx
```

---

## 总结

本开发提示词文档涵盖了 FinVerse 项目第 13-18 章的**极细颗粒度可执行开发指南**，包括：

✅ **第十三章**：6 个数据源适配器的完整实现代码（CoinGecko/Yahoo Finance/Glassnode等）  
✅ **第十四章**：信誉评分算法公式、异常检测规则、AES-256-GCM 加密、免责声明组件  
✅ **第十五章**：Docker Compose 配置、Agent 调度器完整实现、健康检查、灰度发布脚本  
✅ **第十六章**：3 个月 12 个 sprint 任务拆解，每个任务含验收标准  
✅ **第十七章**：完整技术栈版本、package.json、Dockerfile、Nginx 配置、SSL 设置、项目初始化脚本  
✅ **第十八章**：多视角总结、差异化竞争、风险对策、成功标准

**下一步行动**：
1. 执行 `./scripts/init-project.sh` 完成项目初始化
2. 开始 Sprint 1.1 开发（用户系统与认证）
3. 每周进行 sprint review 和验收标准检查

**文档已完成！准备好开始构建 FinVerse 了。** 🚀