# Performance Guidelines and Optimization Strategies

## Philosophy of Performance in SPARC Methodology

### Performance as a First-Class Design Concern
In SPARC methodology, **performance is not an afterthought but a fundamental design constraint** that influences every phase from Specification to Completion. Performance requirements should be explicitly defined during the Specification phase, architecturally planned during the Architecture phase, and continuously validated during implementation.

**Core performance principles:**
- **Performance budgets**: Every feature has measurable performance constraints
- **Observability-driven development**: Instrument code to measure what matters
- **Progressive optimization**: Start with clear, correct code, then optimize based on measured bottlenecks
- **Security-performance harmony**: Never sacrifice security for performance gains
- **User-centric metrics**: Optimize for user experience, not vanity metrics

### The Performance-Security-Maintainability Triangle
SPARC recognizes that performance optimizations often create tension with security and maintainability. Our approach prioritizes:

1. **Security first**: Never compromise security for performance
2. **Maintainability second**: Prefer readable optimizations over clever hacks
3. **Performance third**: Optimize systematically based on real measurements

```typescript
// GOOD: Secure, maintainable, and performant
const cacheUserProfiles = new Map<UserId, { profile: UserProfile; cachedAt: Date }>();

const getUserProfile = async (userId: UserId): Promise<UserProfile> => {
  // Check cache first (performance optimization)
  const cached = cacheUserProfiles.get(userId);
  if (cached && isWithinCacheWindow(cached.cachedAt)) {
    return cached.profile;
  }

  // Fetch from database with explicit authorization check (security first)
  const profile = await userRepository.findProfileById(userId);
  if (!profile) {
    throw new NotFoundError(`User profile not found: ${userId}`);
  }

  // Cache for future requests (performance optimization)
  cacheUserProfiles.set(userId, { profile, cachedAt: new Date() });
  
  return profile;
};

// AVOID: Fast but insecure and unmaintainable
const profiles = new Map(); // No type safety
const getProfile = (id) => profiles.get(id) || db.query(`SELECT * FROM users WHERE id = ${id}`); // SQL injection risk
```

### Performance Budget Philosophy
Every application component must operate within defined performance budgets:

```typescript
// Performance budget configuration
interface PerformanceBudgets {
  readonly api: {
    readonly responseTime95thPercentile: number; // milliseconds
    readonly responseTime99thPercentile: number;
    readonly throughputPerSecond: number;
    readonly errorRateThreshold: number; // percentage
  };
  readonly frontend: {
    readonly firstContentfulPaint: number; // milliseconds
    readonly largestContentfulPaint: number;
    readonly cumulativeLayoutShift: number;
    readonly firstInputDelay: number;
  };
  readonly database: {
    readonly queryTime95thPercentile: number;
    readonly connectionPoolUtilization: number; // percentage
    readonly lockWaitTime: number;
  };
  readonly memory: {
    readonly heapUtilization: number; // percentage
    readonly gcPauseTime: number; // milliseconds
    readonly memoryLeakThreshold: number; // MB per hour
  };
}

const PRODUCTION_BUDGETS: PerformanceBudgets = {
  api: {
    responseTime95thPercentile: 200,
    responseTime99thPercentile: 500,
    throughputPerSecond: 1000,
    errorRateThreshold: 0.1
  },
  frontend: {
    firstContentfulPaint: 1000,
    largestContentfulPaint: 2500,
    cumulativeLayoutShift: 0.1,
    firstInputDelay: 100
  },
  database: {
    queryTime95thPercentile: 50,
    connectionPoolUtilization: 80,
    lockWaitTime: 10
  },
  memory: {
    heapUtilization: 85,
    gcPauseTime: 50,
    memoryLeakThreshold: 10
  }
};
```

## TypeScript/JavaScript Optimization Patterns

### Algorithmic Efficiency and Data Structures
Choose appropriate data structures and algorithms based on access patterns:

```typescript
// Efficient lookup patterns
class UserSessionManager {
  // O(1) lookup for active sessions
  private readonly activeSessions = new Map<SessionId, UserSession>();
  
  // O(1) lookup for user to session mapping
  private readonly userToSession = new Map<UserId, SessionId>();
  
  // Efficient expiry tracking using sorted structure
  private readonly sessionExpiryQueue = new Map<Date, Set<SessionId>>();

  async createSession(userId: UserId, expiresAt: Date): Promise<UserSession> {
    const sessionId = generateSecureSessionId();
    const session: UserSession = {
      id: sessionId,
      userId,
      createdAt: new Date(),
      expiresAt,
      lastAccessedAt: new Date()
    };

    // O(1) insertions
    this.activeSessions.set(sessionId, session);
    this.userToSession.set(userId, sessionId);
    
    // Efficient expiry tracking
    if (!this.sessionExpiryQueue.has(expiresAt)) {
      this.sessionExpiryQueue.set(expiresAt, new Set());
    }
    this.sessionExpiryQueue.get(expiresAt)!.add(sessionId);

    return session;
  }

  async getSessionByUser(userId: UserId): Promise<UserSession | null> {
    const sessionId = this.userToSession.get(userId);
    if (!sessionId) return null;

    const session = this.activeSessions.get(sessionId);
    if (!session || this.isExpired(session)) {
      await this.cleanupExpiredSession(sessionId);
      return null;
    }

    // Update last accessed time
    session.lastAccessedAt = new Date();
    return session;
  }

  // Efficient batch cleanup of expired sessions
  async cleanupExpiredSessions(): Promise<number> {
    const now = new Date();
    let cleanedCount = 0;

    for (const [expiryTime, sessionIds] of this.sessionExpiryQueue.entries()) {
      if (expiryTime <= now) {
        for (const sessionId of sessionIds) {
          await this.cleanupExpiredSession(sessionId);
          cleanedCount++;
        }
        this.sessionExpiryQueue.delete(expiryTime);
      }
    }

    return cleanedCount;
  }
}

// Memory-efficient object processing
const processLargeDataset = async (items: readonly DataItem[]): Promise<ProcessedItem[]> => {
  // Use generators for memory efficiency with large datasets
  async function* processItemsInBatches(
    items: readonly DataItem[], 
    batchSize: number = 100
  ): AsyncGenerator<ProcessedItem[], void, unknown> {
    for (let i = 0; i < items.length; i += batchSize) {
      const batch = items.slice(i, i + batchSize);
      const processed = await Promise.all(
        batch.map(item => processIndividualItem(item))
      );
      yield processed;
    }
  }

  const results: ProcessedItem[] = [];
  for await (const batch of processItemsInBatches(items)) {
    results.push(...batch);
    
    // Allow event loop to process other tasks
    await new Promise(resolve => setImmediate(resolve));
  }

  return results;
};

// Efficient string operations
const formatUserDisplayName = (user: User): string => {
  // Use template literals for better performance than concatenation
  const { firstName, lastName, displayName } = user;
  
  // Early return for common case
  if (displayName) {
    return displayName;
  }

  // Efficient string building
  const parts = [firstName, lastName].filter(Boolean);
  return parts.join(' ') || 'Anonymous User';
};

// Object creation optimization
interface UserPreferences {
  readonly theme: 'light' | 'dark';
  readonly language: string;
  readonly notifications: NotificationSettings;
  readonly privacy: PrivacySettings;
}

// Use object freezing for immutable objects to enable V8 optimizations
const createDefaultPreferences = (): UserPreferences => Object.freeze({
  theme: 'light',
  language: 'en-US',
  notifications: Object.freeze({
    email: true,
    push: false,
    sms: false
  }),
  privacy: Object.freeze({
    profileVisible: true,
    activityTracking: false,
    dataSharing: false
  })
});

// Efficient array operations
const findTopUsersByScore = (users: readonly User[], count: number): User[] => {
  // Use partial sort for better performance when count << users.length
  if (count >= users.length) {
    return [...users].sort((a, b) => b.score - a.score);
  }

  // Heap-based approach for top-k selection
  const topUsers = users.slice(0, count).sort((a, b) => a.score - b.score);
  
  for (let i = count; i < users.length; i++) {
    if (users[i].score > topUsers[0].score) {
      topUsers[0] = users[i];
      
      // Maintain heap property
      let current = 0;
      while (current < count) {
        const left = 2 * current + 1;
        const right = 2 * current + 2;
        let smallest = current;

        if (left < count && topUsers[left].score < topUsers[smallest].score) {
          smallest = left;
        }
        if (right < count && topUsers[right].score < topUsers[smallest].score) {
          smallest = right;
        }

        if (smallest === current) break;

        [topUsers[current], topUsers[smallest]] = [topUsers[smallest], topUsers[current]];
        current = smallest;
      }
    }
  }

  return topUsers.sort((a, b) => b.score - a.score);
};
```

### Asynchronous Programming Optimization
Optimize concurrent operations and avoid blocking the event loop:

```typescript
// Efficient concurrent operations with proper error handling
class EmailBatchProcessor {
  constructor(
    private readonly emailService: EmailService,
    private readonly maxConcurrency = 10,
    private readonly retryAttempts = 3
  ) {}

  async processBatch(emails: readonly Email[]): Promise<BatchResult> {
    const results: EmailResult[] = [];
    const errors: EmailError[] = [];

    // Process emails in controlled concurrency batches
    for (let i = 0; i < emails.length; i += this.maxConcurrency) {
      const batch = emails.slice(i, i + this.maxConcurrency);
      
      const batchPromises = batch.map(async (email, index) => {
        try {
          const result = await this.processEmailWithRetry(email);
          return { success: true, email, result, index: i + index };
        } catch (error) {
          return { success: false, email, error, index: i + index };
        }
      });

      const batchResults = await Promise.allSettled(batchPromises);
      
      batchResults.forEach(result => {
        if (result.status === 'fulfilled') {
          if (result.value.success) {
            results.push(result.value);
          } else {
            errors.push(result.value);
          }
        } else {
          errors.push({
            success: false,
            email: batch[0], // Fallback email reference
            error: result.reason,
            index: -1
          });
        }
      });

      // Allow event loop processing between batches
      await new Promise(resolve => setImmediate(resolve));
    }

    return { results, errors, totalProcessed: emails.length };
  }

  private async processEmailWithRetry(email: Email): Promise<EmailDeliveryResult> {
    let lastError: Error;

    for (let attempt = 1; attempt <= this.retryAttempts; attempt++) {
      try {
        return await this.emailService.send(email);
      } catch (error) {
        lastError = error as Error;
        
        // Exponential backoff with jitter
        if (attempt < this.retryAttempts) {
          const baseDelay = Math.pow(2, attempt) * 100;
          const jitter = Math.random() * 100;
          await new Promise(resolve => setTimeout(resolve, baseDelay + jitter));
        }
      }
    }

    throw lastError!;
  }
}

// Efficient database operations with connection pooling
class UserRepository {
  constructor(
    private readonly db: DatabasePool,
    private readonly cache: CacheManager
  ) {}

  async findUsersByIds(userIds: readonly UserId[]): Promise<Map<UserId, User>> {
    // Separate cached and uncached lookups
    const cached = new Map<UserId, User>();
    const uncachedIds: UserId[] = [];

    for (const id of userIds) {
      const cachedUser = await this.cache.get(`user:${id}`);
      if (cachedUser) {
        cached.set(id, cachedUser);
      } else {
        uncachedIds.push(id);
      }
    }

    // Batch database query for uncached users
    const dbUsers = new Map<UserId, User>();
    if (uncachedIds.length > 0) {
      const query = `
        SELECT id, email, first_name, last_name, created_at, updated_at
        FROM users 
        WHERE id = ANY($1)
      `;
      
      const rows = await this.db.query(query, [uncachedIds]);
      
      rows.forEach(row => {
        const user = this.mapRowToUser(row);
        dbUsers.set(user.id, user);
        
        // Cache for future lookups
        this.cache.set(`user:${user.id}`, user, { ttl: 3600 });
      });
    }

    // Merge cached and database results
    return new Map([...cached, ...dbUsers]);
  }

  async updateUserLastLogin(userId: UserId): Promise<void> {
    // Use fire-and-forget for non-critical updates
    setImmediate(async () => {
      try {
        await this.db.query(
          'UPDATE users SET last_login_at = NOW() WHERE id = $1',
          [userId]
        );
        
        // Invalidate cache
        await this.cache.delete(`user:${userId}`);
      } catch (error) {
        // Log error but don't fail the main request
        console.error('Failed to update last login:', error);
      }
    });
  }
}

// Streaming for large data processing
class DataExportService {
  async exportUserData(userId: UserId): Promise<ReadableStream<Uint8Array>> {
    return new ReadableStream({
      async start(controller) {
        try {
          // Stream header
          controller.enqueue(new TextEncoder().encode('{"users":['));
          
          let isFirst = true;
          
          // Stream user data in chunks
          await this.streamUserRecords(userId, (user) => {
            const prefix = isFirst ? '' : ',';
            isFirst = false;
            
            const chunk = `${prefix}${JSON.stringify(user)}`;
            controller.enqueue(new TextEncoder().encode(chunk));
          });
          
          // Stream footer
          controller.enqueue(new TextEncoder().encode(']}'));
          controller.close();
        } catch (error) {
          controller.error(error);
        }
      }
    });
  }

  private async streamUserRecords(
    userId: UserId, 
    onRecord: (user: User) => void
  ): Promise<void> {
    const cursor = this.db.createCursor(
      'SELECT * FROM users WHERE id = $1',
      [userId]
    );

    try {
      for await (const row of cursor) {
        const user = this.mapRowToUser(row);
        onRecord(user);
        
        // Yield control periodically
        await new Promise(resolve => setImmediate(resolve));
      }
    } finally {
      await cursor.close();
    }
  }
}
```

## Database Query Optimization and Data Access Patterns

### Query Optimization Strategies
Implement efficient database access patterns with proper indexing and query structure:

```typescript
// Efficient query patterns with proper indexing
class AnalyticsRepository {
  // Use prepared statements for repeated queries
  private readonly preparedQueries = {
    userActivityByDateRange: this.db.prepare(`
      SELECT 
        DATE_TRUNC('day', created_at) as activity_date,
        COUNT(*) as activity_count,
        COUNT(DISTINCT user_id) as unique_users
      FROM user_activities 
      WHERE created_at BETWEEN $1 AND $2
        AND activity_type = ANY($3)
      GROUP BY DATE_TRUNC('day', created_at)
      ORDER BY activity_date DESC
    `),

    topActiveUsers: this.db.prepare(`
      SELECT 
        u.id, u.email, u.first_name, u.last_name,
        COUNT(ua.id) as activity_count,
        MAX(ua.created_at) as last_activity
      FROM users u
      INNER JOIN user_activities ua ON u.id = ua.user_id
      WHERE ua.created_at >= $1
        AND u.status = 'active'
      GROUP BY u.id, u.email, u.first_name, u.last_name
      HAVING COUNT(ua.id) >= $2
      ORDER BY activity_count DESC, last_activity DESC
      LIMIT $3
    `)
  };

  async getUserActivitySummary(
    startDate: Date,
    endDate: Date,
    activityTypes: readonly string[]
  ): Promise<ActivitySummary[]> {
    // Use prepared statement for better performance
    const rows = await this.preparedQueries.userActivityByDateRange.execute([
      startDate,
      endDate,
      activityTypes
    ]);

    return rows.map(row => ({
      date: row.activity_date,
      activityCount: parseInt(row.activity_count),
      uniqueUsers: parseInt(row.unique_users)
    }));
  }

  // Efficient pagination with cursor-based approach
  async getRecentActivities(
    cursor?: string,
    limit: number = 50
  ): Promise<PaginatedResult<Activity>> {
    const whereClause = cursor 
      ? 'WHERE created_at < $2' 
      : '';
    
    const params = cursor 
      ? [limit + 1, new Date(cursor)]
      : [limit + 1];

    const query = `
      SELECT id, user_id, activity_type, data, created_at
      FROM user_activities
      ${whereClause}
      ORDER BY created_at DESC
      LIMIT $1
    `;

    const rows = await this.db.query(query, params);
    
    const hasMore = rows.length > limit;
    const activities = rows.slice(0, limit).map(this.mapRowToActivity);
    
    const nextCursor = hasMore && activities.length > 0
      ? activities[activities.length - 1].createdAt.toISOString()
      : null;

    return {
      items: activities,
      hasMore,
      nextCursor
    };
  }

  // Efficient bulk operations
  async createActivitiesBatch(activities: readonly NewActivity[]): Promise<Activity[]> {
    // Use batch insert for better performance
    const values = activities.map(activity => [
      activity.userId,
      activity.type,
      JSON.stringify(activity.data),
      activity.createdAt
    ]);

    const query = `
      INSERT INTO user_activities (user_id, activity_type, data, created_at)
      SELECT * FROM UNNEST($1::uuid[], $2::text[], $3::jsonb[], $4::timestamp[])
      RETURNING id, user_id, activity_type, data, created_at
    `;

    const userIds = values.map(v => v[0]);
    const types = values.map(v => v[1]);
    const dataValues = values.map(v => v[2]);
    const timestamps = values.map(v => v[3]);

    const rows = await this.db.query(query, [userIds, types, dataValues, timestamps]);
    return rows.map(this.mapRowToActivity);
  }
}

// Connection pool optimization
class DatabaseManager {
  private readonly pool: Pool;

  constructor(config: DatabaseConfig) {
    this.pool = new Pool({
      host: config.host,
      port: config.port,
      database: config.database,
      user: config.user,
      password: config.password,
      
      // Connection pool optimization
      min: 5,                    // Minimum connections
      max: 20,                   // Maximum connections
      idleTimeoutMillis: 30000,  // Close idle connections after 30s
      connectionTimeoutMillis: 2000, // Wait max 2s for connection
      
      // Performance tuning
      statement_timeout: 30000,   // 30s query timeout
      query_timeout: 30000,       // 30s query timeout
      application_name: 'sparc-app',
      
      // Connection health monitoring
      keepAlive: true,
      keepAliveInitialDelayMillis: 10000
    });

    // Monitor pool health
    this.pool.on('error', (err) => {
      console.error('Database pool error:', err);
    });

    this.pool.on('connect', (client) => {
      console.debug('New database client connected');
    });

    this.pool.on('remove', (client) => {
      console.debug('Database client disconnected');
    });
  }

  async withTransaction<T>(
    operation: (client: PoolClient) => Promise<T>
  ): Promise<T> {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      const result = await operation(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async getPoolStats(): Promise<PoolStats> {
    return {
      totalConnections: this.pool.totalCount,
      idleConnections: this.pool.idleCount,
      waitingRequests: this.pool.waitingCount
    };
  }
}
```

### Caching Strategies and Implementation
Implement multi-level caching with appropriate invalidation strategies:

```typescript
// Multi-level cache implementation
class CacheManager {
  constructor(
    private readonly l1Cache: Map<string, CacheEntry>, // In-memory L1 cache
    private readonly l2Cache: RedisClient,              // Redis L2 cache
    private readonly l1MaxSize: number = 1000,
    private readonly l1TTL: number = 300000             // 5 minutes
  ) {}

  async get<T>(key: string): Promise<T | null> {
    // Check L1 cache first
    const l1Entry = this.l1Cache.get(key);
    if (l1Entry && !this.isExpired(l1Entry)) {
      return l1Entry.value as T;
    }

    // Check L2 cache
    try {
      const l2Value = await this.l2Cache.get(key);
      if (l2Value) {
        const parsed = JSON.parse(l2Value) as T;
        
        // Promote to L1 cache
        this.setL1(key, parsed);
        
        return parsed;
      }
    } catch (error) {
      console.warn('L2 cache read error:', error);
    }

    return null;
  }

  async set<T>(key: string, value: T, options?: CacheOptions): Promise<void> {
    const ttl = options?.ttl || this.l1TTL;

    // Set in L1 cache
    this.setL1(key, value, ttl);

    // Set in L2 cache
    try {
      const serialized = JSON.stringify(value);
      await this.l2Cache.setex(key, Math.floor(ttl / 1000), serialized);
    } catch (error) {
      console.warn('L2 cache write error:', error);
    }
  }

  async delete(key: string): Promise<void> {
    // Remove from L1
    this.l1Cache.delete(key);

    // Remove from L2
    try {
      await this.l2Cache.del(key);
    } catch (error) {
      console.warn('L2 cache delete error:', error);
    }
  }

  async invalidatePattern(pattern: string): Promise<void> {
    // Invalidate L1 cache entries matching pattern
    const regex = new RegExp(pattern.replace('*', '.*'));
    for (const key of this.l1Cache.keys()) {
      if (regex.test(key)) {
        this.l1Cache.delete(key);
      }
    }

    // Invalidate L2 cache entries
    try {
      const keys = await this.l2Cache.keys(pattern);
      if (keys.length > 0) {
        await this.l2Cache.del(...keys);
      }
    } catch (error) {
      console.warn('L2 cache pattern invalidation error:', error);
    }
  }

  private setL1<T>(key: string, value: T, ttl: number = this.l1TTL): void {
    // Implement LRU eviction if cache is full
    if (this.l1Cache.size >= this.l1MaxSize) {
      const oldestKey = this.l1Cache.keys().next().value;
      this.l1Cache.delete(oldestKey);
    }

    this.l1Cache.set(key, {
      value,
      expiresAt: Date.now() + ttl
    });
  }

  private isExpired(entry: CacheEntry): boolean {
    return Date.now() > entry.expiresAt;
  }
}

// Cache-aware service implementation
class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly cache: CacheManager
  ) {}

  async getUserProfile(userId: UserId): Promise<UserProfile | null> {
    const cacheKey = `user:profile:${userId}`;
    
    // Try cache first
    const cached = await this.cache.get<UserProfile>(cacheKey);
    if (cached) {
      return cached;
    }

    // Fetch from database
    const profile = await this.userRepository.findProfileById(userId);
    if (!profile) {
      return null;
    }

    // Cache for future requests
    await this.cache.set(cacheKey, profile, { ttl: 300000 }); // 5 minutes

    return profile;
  }

  async updateUserProfile(
    userId: UserId, 
    updates: ProfileUpdates
  ): Promise<UserProfile> {
    const profile = await this.userRepository.updateProfile(userId, updates);

    // Invalidate related cache entries
    await Promise.all([
      this.cache.delete(`user:profile:${userId}`),
      this.cache.delete(`user:full:${userId}`),
      this.cache.invalidatePattern(`user:search:*`) // Invalidate search results
    ]);

    return profile;
  }
}

// Application-specific caching patterns
class ContentCache {
  constructor(private readonly cache: CacheManager) {}

  // Cache expensive computations
  async getPopularContent(category: string, limit: number): Promise<Content[]> {
    const cacheKey = `popular:${category}:${limit}`;
    
    let content = await this.cache.get<Content[]>(cacheKey);
    if (!content) {
      content = await this.computePopularContent(category, limit);
      
      // Cache for 10 minutes with tag for bulk invalidation
      await this.cache.set(cacheKey, content, { ttl: 600000 });
    }

    return content;
  }

  // Cache with dependency tracking
  async getUserRecommendations(userId: UserId): Promise<Recommendation[]> {
    const cacheKey = `recommendations:${userId}`;
    
    let recommendations = await this.cache.get<Recommendation[]>(cacheKey);
    if (!recommendations) {
      recommendations = await this.computeUserRecommendations(userId);
      
      // Cache for 1 hour
      await this.cache.set(cacheKey, recommendations, { ttl: 3600000 });
    }

    return recommendations;
  }

  // Invalidate when user behavior changes
  async invalidateUserRecommendations(userId: UserId): Promise<void> {
    await this.cache.delete(`recommendations:${userId}`);
  }

  private async computePopularContent(category: string, limit: number): Promise<Content[]> {
    // Expensive computation logic here
    throw new Error('Implementation needed');
  }

  private async computeUserRecommendations(userId: UserId): Promise<Recommendation[]> {
    // ML-based recommendation computation
    throw new Error('Implementation needed');
  }
}
```

## Memory Management and Resource Optimization

### Memory Leak Prevention
Implement patterns to prevent common memory leaks in Node.js applications:

```typescript
// Proper event listener management
class EventDrivenService {
  private readonly eventListeners = new Map<string, (...args: any[]) => void>();
  private readonly timers = new Set<NodeJS.Timeout>();
  private readonly intervals = new Set<NodeJS.Timeout>();

  constructor(private readonly eventBus: EventBus) {}

  addEventHandler<T>(event: string, handler: (data: T) => void): void {
    // Wrap handler to enable cleanup
    const wrappedHandler = (data: T) => {
      try {
        handler(data);
      } catch (error) {
        console.error(`Error in event handler for ${event}:`, error);
      }
    };

    this.eventListeners.set(event, wrappedHandler);
    this.eventBus.on(event, wrappedHandler);
  }

  scheduleTask(task: () => void, delay: number): void {
    const timer = setTimeout(() => {
      try {
        task();
      } catch (error) {
        console.error('Scheduled task error:', error);
      } finally {
        this.timers.delete(timer);
      }
    }, delay);

    this.timers.add(timer);
  }

  scheduleRecurringTask(task: () => void, interval: number): void {
    const timer = setInterval(() => {
      try {
        task();
      } catch (error) {
        console.error('Recurring task error:', error);
      }
    }, interval);

    this.intervals.add(timer);
  }

  // Cleanup method to prevent memory leaks
  async cleanup(): Promise<void> {
    // Remove event listeners
    for (const [event, handler] of this.eventListeners) {
      this.eventBus.removeListener(event, handler);
    }
    this.eventListeners.clear();

    // Clear timers
    for (const timer of this.timers) {
      clearTimeout(timer);
    }
    this.timers.clear();

    // Clear intervals
    for (const interval of this.intervals) {
      clearInterval(interval);
    }
    this.intervals.clear();
  }
}

// Memory-efficient data processing
class StreamingDataProcessor {
  private readonly maxMemoryUsage = 100 * 1024 * 1024; // 100MB

  async processLargeDataset<T, R>(
    dataSource: AsyncIterable<T>,
    processor: (item: T) => Promise<R>,
    options: ProcessingOptions = {}
  ): Promise<void> {
    const { batchSize = 100, outputHandler } = options;
    
    let batch: T[] = [];
    let processedCount = 0;

    for await (const item of dataSource) {
      batch.push(item);

      if (batch.length >= batchSize) {
        await this.processBatch(batch, processor, outputHandler);
        batch = []; // Clear batch to free memory
        processedCount += batchSize;

        // Monitor memory usage
        const memoryUsage = process.memoryUsage();
        if (memoryUsage.heapUsed > this.maxMemoryUsage) {
          // Force garbage collection if available
          if (global.gc) {
            global.gc();
          }
          
          // Log warning about high memory usage
          console.warn(`High memory usage: ${memoryUsage.heapUsed / 1024 / 1024}MB`);
        }

        // Yield control to event loop
        await new Promise(resolve => setImmediate(resolve));
      }
    }

    // Process remaining items
    if (batch.length > 0) {
      await this.processBatch(batch, processor, outputHandler);
    }

    console.info(`Processed ${processedCount} items`);
  }

  private async processBatch<T, R>(
    batch: T[],
    processor: (item: T) => Promise<R>,
    outputHandler?: (results: R[]) => Promise<void>
  ): Promise<void> {
    const results = await Promise.all(batch.map(processor));
    
    if (outputHandler) {
      await outputHandler(results);
    }
  }
}

// WeakMap usage for memory-efficient associations
class ObjectMetadataManager {
  // Use WeakMap to avoid memory leaks - entries are GC'd when objects are
  private readonly metadata = new WeakMap<object, ObjectMetadata>();
  private readonly accessTimes = new WeakMap<object, Date>();

  setMetadata(obj: object, metadata: ObjectMetadata): void {
    this.metadata.set(obj, metadata);
    this.accessTimes.set(obj, new Date());
  }

  getMetadata(obj: object): ObjectMetadata | undefined {
    const metadata = this.metadata.get(obj);
    if (metadata) {
      this.accessTimes.set(obj, new Date());
    }
    return metadata;
  }

  // No explicit cleanup needed - WeakMap entries are automatically GC'd
}

// Resource pool for expensive objects
class ConnectionPool<T> {
  private readonly available: T[] = [];
  private readonly inUse = new Set<T>();
  private readonly maxSize: number;

  constructor(
    private readonly factory: () => Promise<T>,
    private readonly destroyer: (resource: T) => Promise<void>,
    maxSize: number = 10
  ) {
    this.maxSize = maxSize;
  }

  async acquire(): Promise<PooledResource<T>> {
    let resource: T;

    if (this.available.length > 0) {
      resource = this.available.pop()!;
    } else if (this.inUse.size < this.maxSize) {
      resource = await this.factory();
    } else {
      // Wait for a resource to become available
      await new Promise(resolve => setTimeout(resolve, 10));
      return this.acquire(); // Retry
    }

    this.inUse.add(resource);

    return {
      resource,
      release: async () => {
        this.inUse.delete(resource);
        this.available.push(resource);
      }
    };
  }

  async cleanup(): Promise<void> {
    // Destroy all available resources
    await Promise.all(this.available.map(this.destroyer));
    this.available.length = 0;

    // Note: In-use resources should be released by their users
    if (this.inUse.size > 0) {
      console.warn(`${this.inUse.size} resources still in use during cleanup`);
    }
  }

  getStats(): PoolStats {
    return {
      available: this.available.length,
      inUse: this.inUse.size,
      total: this.available.length + this.inUse.size
    };
  }
}
```

## Monitoring and Performance Profiling

### Application Performance Monitoring Integration
Implement comprehensive performance monitoring across all application layers:

```typescript
// Performance monitoring infrastructure
class PerformanceMonitor {
  private readonly metrics = new Map<string, Metric[]>();
  private readonly thresholds: PerformanceThresholds;

  constructor(thresholds: PerformanceThresholds) {
    this.thresholds = thresholds;
    
    // Start periodic metric collection
    setInterval(() => this.collectSystemMetrics(), 30000); // Every 30s
  }

  // Time function execution with automatic alerting
  async timeExecution<T>(
    operation: string,
    fn: () => Promise<T>,
    context?: Record<string, any>
  ): Promise<T> {
    const startTime = process.hrtime.bigint();
    const startMemory = process.memoryUsage();

    try {
      const result = await fn();
      
      const endTime = process.hrtime.bigint();
      const endMemory = process.memoryUsage();
      
      const duration = Number(endTime - startTime) / 1_000_000; // Convert to milliseconds
      const memoryDelta = endMemory.heapUsed - startMemory.heapUsed;

      this.recordMetric(operation, {
        duration,
        memoryDelta,
        status: 'success',
        context,
        timestamp: new Date()
      });

      // Check performance thresholds
      await this.checkThresholds(operation, duration, memoryDelta);

      return result;
    } catch (error) {
      const endTime = process.hrtime.bigint();
      const duration = Number(endTime - startTime) / 1_000_000;

      this.recordMetric(operation, {
        duration,
        memoryDelta: 0,
        status: 'error',
        error: error.message,
        context,
        timestamp: new Date()
      });

      throw error;
    }
  }

  // Database query performance monitoring
  monitorDatabaseQuery<T>(
    query: string,
    params: any[],
    executor: (query: string, params: any[]) => Promise<T>
  ): Promise<T> {
    return this.timeExecution(
      'database_query',
      () => executor(query, params),
      { 
        query: query.substring(0, 100), // Truncate for logging
        paramCount: params.length 
      }
    );
  }

  // HTTP request monitoring middleware
  createExpressMiddleware() {
    return (req: Request, res: Response, next: NextFunction) => {
      const startTime = process.hrtime.bigint();

      res.on('finish', () => {
        const endTime = process.hrtime.bigint();
        const duration = Number(endTime - startTime) / 1_000_000;

        this.recordMetric('http_request', {
          duration,
          memoryDelta: 0,
          status: res.statusCode < 400 ? 'success' : 'error',
          context: {
            method: req.method,
            path: req.path,
            statusCode: res.statusCode,
            userAgent: req.get('User-Agent')
          },
          timestamp: new Date()
        });
      });

      next();
    };
  }

  private recordMetric(operation: string, metric: Metric): void {
    if (!this.metrics.has(operation)) {
      this.metrics.set(operation, []);
    }

    const operationMetrics = this.metrics.get(operation)!;
    operationMetrics.push(metric);

    // Keep only last 1000 metrics per operation
    if (operationMetrics.length > 1000) {
      operationMetrics.splice(0, operationMetrics.length - 1000);
    }
  }

  private async checkThresholds(
    operation: string,
    duration: number,
    memoryDelta: number
  ): Promise<void> {
    const threshold = this.thresholds[operation];
    if (!threshold) return;

    if (duration > threshold.maxDuration) {
      await this.alertSlowOperation(operation, duration, threshold.maxDuration);
    }

    if (memoryDelta > threshold.maxMemoryDelta) {
      await this.alertHighMemoryUsage(operation, memoryDelta, threshold.maxMemoryDelta);
    }
  }

  private async alertSlowOperation(
    operation: string,
    actual: number,
    threshold: number
  ): Promise<void> {
    console.warn(`Slow operation detected: ${operation} took ${actual}ms (threshold: ${threshold}ms)`);
    
    // Send alert to monitoring system
    // await this.alertingService.sendAlert({
    //   type: 'performance',
    //   severity: 'warning',
    //   message: `Operation ${operation} exceeded duration threshold`,
    //   metadata: { actual, threshold }
    // });
  }

  private collectSystemMetrics(): void {
    const memoryUsage = process.memoryUsage();
    const cpuUsage = process.cpuUsage();

    this.recordMetric('system_memory', {
      duration: 0,
      memoryDelta: 0,
      status: 'success',
      context: {
        heapUsed: memoryUsage.heapUsed,
        heapTotal: memoryUsage.heapTotal,
        external: memoryUsage.external,
        rss: memoryUsage.rss
      },
      timestamp: new Date()
    });

    this.recordMetric('system_cpu', {
      duration: 0,
      memoryDelta: 0,
      status: 'success',
      context: {
        user: cpuUsage.user,
        system: cpuUsage.system
      },
      timestamp: new Date()
    });
  }

  getMetricsSummary(operation: string): MetricsSummary | null {
    const metrics = this.metrics.get(operation);
    if (!metrics || metrics.length === 0) return null;

    const durations = metrics.map(m => m.duration);
    const successCount = metrics.filter(m => m.status === 'success').length;

    return {
      operation,
      totalRequests: metrics.length,
      successRate: successCount / metrics.length,
      averageDuration: durations.reduce((sum, d) => sum + d, 0) / durations.length,
      p50: this.percentile(durations, 0.5),
      p95: this.percentile(durations, 0.95),
      p99: this.percentile(durations, 0.99),
      maxDuration: Math.max(...durations),
      minDuration: Math.min(...durations)
    };
  }

  private percentile(values: number[], p: number): number {
    const sorted = [...values].sort((a, b) => a - b);
    const index = Math.ceil(sorted.length * p) - 1;
    return sorted[index] || 0;
  }
}

// Custom performance profiler for critical paths
class ProfiledFunction {
  private callCount = 0;
  private totalDuration = 0;
  private slowCalls: Array<{ duration: number; timestamp: Date; context?: any }> = [];

  constructor(
    private readonly name: string,
    private readonly slowThreshold: number = 100 // ms
  ) {}

  async execute<T>(fn: () => Promise<T>, context?: any): Promise<T> {
    const startTime = process.hrtime.bigint();
    
    try {
      const result = await fn();
      this.recordCall(startTime, context);
      return result;
    } catch (error) {
      this.recordCall(startTime, { ...context, error: error.message });
      throw error;
    }
  }

  private recordCall(startTime: bigint, context?: any): void {
    const endTime = process.hrtime.bigint();
    const duration = Number(endTime - startTime) / 1_000_000;

    this.callCount++;
    this.totalDuration += duration;

    if (duration > this.slowThreshold) {
      this.slowCalls.push({
        duration,
        timestamp: new Date(),
        context
      });

      // Keep only last 10 slow calls
      if (this.slowCalls.length > 10) {
        this.slowCalls.shift();
      }
    }
  }

  getStats(): FunctionStats {
    return {
      name: this.name,
      callCount: this.callCount,
      averageDuration: this.callCount > 0 ? this.totalDuration / this.callCount : 0,
      slowCallCount: this.slowCalls.length,
      recentSlowCalls: this.slowCalls.slice(-5)
    };
  }
}
```

## Security-Performance Balance Considerations

### Secure Performance Optimizations
Implement performance optimizations that maintain security integrity:

```typescript
// Rate limiting with performance optimization
class AdaptiveRateLimiter {
  private readonly limits = new Map<string, RateLimitState>();
  private readonly suspiciousIPs = new Set<string>();

  constructor(
    private readonly redis: RedisClient,
    private readonly config: RateLimitConfig
  ) {}

  async checkLimit(
    identifier: string,
    action: string,
    clientIP: string
  ): Promise<RateLimitResult> {
    const key = `rate_limit:${action}:${identifier}`;
    
    // Enhanced limits for suspicious IPs
    const isSuspicious = this.suspiciousIPs.has(clientIP);
    const limit = isSuspicious 
      ? this.config.suspiciousLimit 
      : this.config.normalLimit;

    // Use Lua script for atomic operations
    const luaScript = `
      local key = KEYS[1]
      local limit = tonumber(ARGV[1])
      local window = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      
      local current = redis.call('GET', key)
      if current == false then
        redis.call('SET', key, 1)
        redis.call('EXPIRE', key, window)
        return {1, limit}
      end
      
      current = tonumber(current)
      if current < limit then
        redis.call('INCR', key)
        return {current + 1, limit}
      else
        return {current, limit}
      end
    `;

    const [current, maxLimit] = await this.redis.eval(
      luaScript,
      1,
      key,
      limit.toString(),
      this.config.windowSeconds.toString(),
      Date.now().toString()
    ) as [number, number];

    const isAllowed = current <= maxLimit;
    
    // Track failed attempts for security
    if (!isAllowed) {
      await this.trackFailedAttempt(clientIP, action);
    }

    return {
      allowed: isAllowed,
      current,
      limit: maxLimit,
      resetTime: Date.now() + (this.config.windowSeconds * 1000)
    };
  }

  private async trackFailedAttempt(clientIP: string, action: string): Promise<void> {
    const key = `failed_attempts:${clientIP}`;
    const attempts = await this.redis.incr(key);
    await this.redis.expire(key, 3600); // 1 hour window

    // Mark as suspicious after threshold
    if (attempts > this.config.suspiciousThreshold) {
      this.suspiciousIPs.add(clientIP);
      
      // Expire suspicious status after 24 hours
      setTimeout(() => {
        this.suspiciousIPs.delete(clientIP);
      }, 24 * 60 * 60 * 1000);

      // Log security event
      console.warn(`IP ${clientIP} marked as suspicious after ${attempts} failed attempts`);
    }
  }
}

// Secure caching with encryption for sensitive data
class SecureCache {
  constructor(
    private readonly cache: CacheManager,
    private readonly encryptionKey: string
  ) {}

  async setSecure<T>(
    key: string, 
    value: T, 
    options?: CacheOptions & { encrypt?: boolean }
  ): Promise<void> {
    if (options?.encrypt) {
      const encrypted = await this.encrypt(JSON.stringify(value));
      await this.cache.set(key, { encrypted: true, data: encrypted }, options);
    } else {
      await this.cache.set(key, { encrypted: false, data: value }, options);
    }
  }

  async getSecure<T>(key: string): Promise<T | null> {
    const cached = await this.cache.get<CachedData>(key);
    if (!cached) return null;

    if (cached.encrypted) {
      const decrypted = await this.decrypt(cached.data as string);
      return JSON.parse(decrypted) as T;
    } else {
      return cached.data as T;
    }
  }

  private async encrypt(data: string): Promise<string> {
    const crypto = await import('crypto');
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipher('aes-256-gcm', this.encryptionKey);
    
    let encrypted = cipher.update(data, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return JSON.stringify({
      iv: iv.toString('hex'),
      encrypted,
      authTag: authTag.toString('hex')
    });
  }

  private async decrypt(encryptedData: string): Promise<string> {
    const crypto = await import('crypto');
    const { iv, encrypted, authTag } = JSON.parse(encryptedData);
    
    const decipher = crypto.createDecipher('aes-256-gcm', this.encryptionKey);
    decipher.setAuthTag(Buffer.from(authTag, 'hex'));
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}

// Performance-optimized input validation
class FastValidator {
  private readonly compiledSchemas = new Map<string, CompiledSchema>();

  constructor() {
    // Pre-compile frequently used schemas for better performance
    this.precompileCommonSchemas();
  }

  async validateInput<T>(
    schemaName: string,
    input: unknown,
    options?: ValidationOptions
  ): Promise<ValidationResult<T>> {
    const startTime = process.hrtime.bigint();
    
    try {
      const schema = this.getCompiledSchema(schemaName);
      if (!schema) {
        throw new Error(`Schema not found: ${schemaName}`);
      }

      // Fast path for simple validations
      if (schema.isSimple && !options?.strictMode) {
        return this.fastValidate(schema, input);
      }

      // Full validation for complex schemas
      const result = schema.validate(input);
      
      const duration = Number(process.hrtime.bigint() - startTime) / 1_000_000;
      
      // Log slow validations for optimization
      if (duration > 10) { // 10ms threshold
        console.warn(`Slow validation: ${schemaName} took ${duration}ms`);
      }

      return result;
    } catch (error) {
      return {
        success: false,
        error: error.message,
        validationErrors: []
      };
    }
  }

  private precompileCommonSchemas(): void {
    // Pre-compile schemas for better runtime performance
    const commonSchemas = [
      'userRegistration',
      'loginCredentials',
      'profileUpdate',
      'paymentInfo'
    ];

    for (const schemaName of commonSchemas) {
      this.compileSchema(schemaName);
    }
  }

  private fastValidate<T>(schema: CompiledSchema, input: unknown): ValidationResult<T> {
    // Simplified validation for performance-critical paths
    if (typeof input !== 'object' || input === null) {
      return {
        success: false,
        error: 'Input must be an object',
        validationErrors: []
      };
    }

    // Basic type checking without full schema validation
    return {
      success: true,
      data: input as T
    };
  }
}
```

## Memory Bank Integration and Performance Decision Tracking

### Performance Decision Documentation Template
Document performance-related decisions in `memory-bank/decisionLog.md`:

```markdown
## Performance Decision: [Optimization Name]
**Date**: [YYYY-MM-DD]
**Phase**: Implementation
**Decision Type**: Performance Optimization
**Implementer**: [Mode/Human identifier]

### Context
- **Performance Issue**: [Specific performance problem being addressed]
- **Current Metrics**: [Baseline performance measurements]
- **Business Impact**: [How performance affects user experience/business]
- **Constraints**: [Resource, security, or maintainability limitations]

### Options Considered
1. **Option A**: [Optimization approach - e.g., "In-memory caching"]
   - **Performance Gain**: [Expected improvement]
   - **Resource Cost**: [Memory/CPU/storage requirements]
   - **Security Implications**: [Any security concerns]
   - **Complexity**: [Implementation and maintenance complexity]

2. **Option B**: [Alternative approach]
   - **Performance Gain**: [Expected improvement]
   - **Resource Cost**: [Memory/CPU/storage requirements]
   - **Security Implications**: [Any security concerns]
   - **Complexity**: [Implementation and maintenance complexity]

### Decision
**Chosen**: [Selected optimization]
**Rationale**: [Why this option was selected]

### Implementation Details
- **Technology Used**: [Specific tools/libraries/patterns]
- **Configuration**: [Key configuration parameters]
- **Security Measures**: [How security is maintained]
- **Monitoring**: [How performance will be measured]

### Success Criteria
- **Performance Target**: [Specific performance goals]
- **Metrics to Track**: [KPIs for measuring success]
- **Acceptable Trade-offs**: [What compromises are acceptable]

### Results (Updated Post-Implementation)
- **Actual Performance Gain**: [Measured improvement]
- **Resource Usage**: [Actual resource consumption]
- **Unexpected Issues**: [Any problems encountered]
- **Lessons Learned**: [Key insights for future optimizations]

### Review Schedule
- **Next Review**: [Date for performance review]
- **Success Threshold**: [Minimum acceptable performance]
- **Failure Triggers**: [Conditions requiring rollback or changes]
```

### Performance Pattern Registry
Maintain performance patterns in `memory-bank/systemPatterns.md`:

```markdown
## Performance Patterns Registry

### Caching Patterns
**Pattern**: Multi-Level Cache with L1/L2 Strategy
**Use Case**: Frequently accessed data with varying access patterns
**Implementation**: In-memory L1 + Redis L2 with promotion/demotion
**Performance**: ~10x improvement for cache hits
**Security**: Encryption for sensitive cached data
**Monitoring**: Hit/miss ratios + cache eviction metrics

### Database Optimization Patterns
**Pattern**: Connection Pooling with Health Monitoring
**Use Case**: High-concurrency database access
**Implementation**: pg.Pool with adaptive sizing
**Performance**: 3x improvement in concurrent request handling
**Security**: Connection-level authentication + query timeout
**Monitoring**: Pool utilization + query performance metrics

### Async Processing Patterns
**Pattern**: Controlled Concurrency with Backpressure
**Use Case**: Batch processing of large datasets
**Implementation**: Semaphore-based concurrency control
**Performance**: 80% memory reduction for large operations
**Security**: Rate limiting + resource bounds
**Monitoring**: Queue depth + processing latency

### Memory Management Patterns
**Pattern**: WeakMap Associations for Metadata
**Use Case**: Temporary object metadata without memory leaks
**Implementation**: WeakMap for automatic garbage collection
**Performance**: Zero memory overhead after object cleanup
**Security**: No data retention after object destruction
**Monitoring**: Heap usage tracking + GC metrics
```

### Performance Monitoring Integration
Track performance improvements in `memory-bank/progress.md`:

```markdown
## Performance Optimization Progress

### Active Performance Work
- **API Response Time Optimization**: Target <200ms 95th percentile (Currently: 150ms ✅)
- **Database Query Optimization**: Target <50ms average (Currently: 35ms ✅) 
- **Memory Usage Reduction**: Target <500MB heap (Currently: 420MB ✅)
- **Cache Hit Rate Improvement**: Target >90% (Currently: 87% ⚠️)

### Performance Metrics Trends
- **Response Time**: 40% improvement over last month
- **Throughput**: 150% increase in requests/second
- **Memory Usage**: 25% reduction in peak memory
- **Error Rate**: Reduced from 0.5% to 0.1%

### Performance Debt Registry
- **Priority High**: N+1 query pattern in user dashboard (affects 1000+ users)
- **Priority Medium**: Inefficient image processing (20% of CPU usage)
- **Priority Low**: Unoptimized search queries (rarely used feature)

### Security-Performance Balance
- **Encryption Overhead**: 5ms average added to API calls (acceptable)
- **Rate Limiting Impact**: 99.9% of legitimate requests unaffected
- **Input Validation**: <1ms overhead for 95% of requests
- **Authentication**: JWT validation adds 2ms average (within budget)

### Monitoring and Alerting
- **Performance Budgets**: All within targets except cache hit rate
- **SLA Compliance**: 99.8% uptime with <200ms response time
- **Alert Coverage**: 95% of performance issues detected within 1 minute
- **Capacity Planning**: Current usage at 60% of provisioned capacity
```

This comprehensive performance guide ensures that all optimization efforts in the SPARC methodology maintain the critical balance between speed, security, and maintainability while providing clear measurement and accountability for performance improvements.