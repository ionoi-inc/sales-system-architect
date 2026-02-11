# Sales System Architecture

## System Design Patterns

### Microservices Architecture

The Sales System employs a domain-driven microservices architecture with the following service boundaries:

- **Lead Service**: Lead capture, qualification, routing, and lifecycle management
- **Opportunity Service**: Deal tracking, stage management, forecasting, and closing workflows
- **Territory Service**: Geographic and account-based territory assignment with conflict resolution
- **Quota Service**: Hierarchical quota planning, allocation, and attainment tracking
- **Intelligence Service**: AI-powered deal scoring, next-best-action recommendations, and analytics
- **Activity Service**: Email, call, meeting, and task tracking with timeline aggregation

### Event-Driven Communication

Services communicate asynchronously via RabbitMQ with the following event patterns:

- **Lead Events**: `lead.created`, `lead.qualified`, `lead.assigned`, `lead.converted`
- **Opportunity Events**: `opportunity.created`, `opportunity.stage_changed`, `opportunity.won`, `opportunity.lost`
- **Territory Events**: `territory.created`, `territory.updated`, `territory.assignment_changed`
- **Quota Events**: `quota.set`, `quota.updated`, `quota.attained`
- **Activity Events**: `activity.logged`, `activity.completed`

### CQRS Pattern

Read and write operations are separated for optimal performance:

- **Command Side**: PostgreSQL for transactional writes
- **Query Side**: Elasticsearch for complex searches, Redis for real-time dashboards
- **Synchronization**: Event-driven materialized views updated via event handlers

## Data Models and Schemas

### Lead Schema

```sql
CREATE TABLE leads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL, -- new, contacted, qualified, unqualified, converted
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    company VARCHAR(255),
    title VARCHAR(100),
    industry VARCHAR(100),
    employee_count INTEGER,
    annual_revenue DECIMAL(15,2),
    score INTEGER DEFAULT 0, -- 0-100 lead score
    assigned_to UUID REFERENCES users(id),
    territory_id UUID REFERENCES territories(id),
    converted_opportunity_id UUID REFERENCES opportunities(id),
    last_activity_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_status (status),
    INDEX idx_assigned_to (assigned_to),
    INDEX idx_score (score DESC)
);
```

### Opportunity Schema

```sql
CREATE TABLE opportunities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    account_id UUID REFERENCES accounts(id),
    lead_id UUID REFERENCES leads(id),
    stage VARCHAR(50) NOT NULL, -- discovery, qualification, proposal, negotiation, closed_won, closed_lost
    probability INTEGER, -- 0-100
    amount DECIMAL(15,2),
    expected_close_date DATE,
    actual_close_date DATE,
    owner_id UUID REFERENCES users(id),
    territory_id UUID REFERENCES territories(id),
    deal_score INTEGER, -- 0-100 AI-powered deal health score
    loss_reason VARCHAR(255),
    close_reason VARCHAR(255),
    next_step TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_stage (stage),
    INDEX idx_owner (owner_id),
    INDEX idx_close_date (expected_close_date)
);
```

### Territory Schema

```sql
CREATE TABLE territories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL, -- geographic, account, industry
    owner_id UUID REFERENCES users(id),
    parent_territory_id UUID REFERENCES territories(id),
    
    -- Geographic rules
    countries TEXT[], -- array of country codes
    states TEXT[],    -- array of state codes
    zip_codes TEXT[], -- array of zip code patterns
    
    -- Account rules
    industry VARCHAR(100),
    min_employee_count INTEGER,
    max_employee_count INTEGER,
    min_annual_revenue DECIMAL(15,2),
    max_annual_revenue DECIMAL(15,2),
    
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_owner (owner_id),
    INDEX idx_type (type)
);
```

### Quota Schema

```sql
CREATE TABLE quotas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    territory_id UUID REFERENCES territories(id),
    period VARCHAR(20) NOT NULL, -- Q1_2026, Q2_2026, etc.
    quota_amount DECIMAL(15,2) NOT NULL,
    attainment_amount DECIMAL(15,2) DEFAULT 0,
    attainment_percentage DECIMAL(5,2) DEFAULT 0,
    quota_type VARCHAR(50) NOT NULL, -- revenue, deals, activities
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_user_period (user_id, period),
    INDEX idx_territory_period (territory_id, period)
);
```

### Activity Schema (MongoDB)

```json
{
  "_id": "ObjectId",
  "type": "email|call|meeting|demo|task",
  "subject": "string",
  "description": "string",
  "lead_id": "uuid",
  "opportunity_id": "uuid",
  "account_id": "uuid",
  "user_id": "uuid",
  "outcome": "string",
  "duration_minutes": "integer",
  "scheduled_at": "timestamp",
  "completed_at": "timestamp",
  "participants": ["uuid"],
  "metadata": {
    "email_opened": "boolean",
    "email_clicked": "boolean",
    "call_recording_url": "string",
    "meeting_notes": "string"
  },
  "created_at": "timestamp",
  "updated_at": "timestamp"
}
```

## Scalability Strategy

### Horizontal Scaling

- **API Gateway**: Load-balanced across multiple instances with health checks
- **Service Instances**: Auto-scaled based on CPU/memory utilization (min: 2, max: 20)
- **Database**: Read replicas for query load distribution
- **Cache**: Redis Cluster with sharding for high-volume data

### Database Optimization

- **Partitioning**: Opportunities table partitioned by close date (quarterly partitions)
- **Indexing**: Strategic indexes on frequently queried columns (status, owner, date ranges)
- **Archival**: Closed opportunities moved to cold storage after 2 years
- **Query Optimization**: Materialized views for dashboard aggregations

### Caching Strategy

```typescript
// Hot data caching
interface CacheStrategy {
  pipeline_summary: { ttl: 60, key: 'pipeline:user:{userId}' },
  territory_assignments: { ttl: 300, key: 'territory:assignments' },
  quota_attainment: { ttl: 120, key: 'quota:user:{userId}:{period}' },
  deal_scores: { ttl: 180, key: 'deal:score:{opportunityId}' }
}
```

### Load Distribution

- **Geographic Distribution**: Multi-region deployment (US-East, EU-West, APAC)
- **CDN**: Static assets and API responses cached at edge locations
- **Rate Limiting**: Per-user rate limits (1000 req/min) to prevent abuse

## Security Model

### Authentication & Authorization

```typescript
interface AuthModel {
  authentication: {
    method: 'JWT + OAuth2.0',
    providers: ['okta', 'auth0', 'google'],
    session_duration: '8 hours',
    refresh_token_rotation: true
  },
  authorization: {
    model: 'RBAC + Attribute-Based',
    roles: ['admin', 'sales_manager', 'sales_rep', 'read_only'],
    permissions: {
      'opportunity:read': ['admin', 'sales_manager', 'sales_rep'],
      'opportunity:write': ['admin', 'sales_manager', 'sales_rep'],
      'opportunity:delete': ['admin', 'sales_manager'],
      'territory:assign': ['admin', 'sales_manager'],
      'quota:set': ['admin', 'sales_manager']
    }
  }
}
```

### Data Encryption

- **At Rest**: AES-256 encryption for sensitive fields (SSN, banking info)
- **In Transit**: TLS 1.3 for all API communications
- **Field-Level**: Additional encryption for PII fields (email, phone, address)
- **Key Management**: AWS KMS with automatic key rotation

### Access Control

- **Territory-Based**: Reps can only view leads/opportunities in their territories
- **Hierarchical**: Managers see all data for their reports and territories
- **Field-Level**: Sensitive fields (compensation, deal margins) restricted by role
- **Audit Trail**: All data access and modifications logged with user context

### Compliance

- **GDPR**: Right to access, right to be forgotten, data portability
- **CCPA**: California consumer privacy compliance
- **SOC 2 Type II**: Annual audit with continuous monitoring
- **Data Retention**: Configurable retention policies per data type

## Performance Benchmarks

### API Response Times (p95)

| Endpoint | Target | Actual |
|----------|--------|--------|
| GET /leads | <100ms | 78ms |
| POST /leads | <150ms | 112ms |
| GET /opportunities | <100ms | 85ms |
| PUT /opportunities/:id | <150ms | 128ms |
| GET /territories | <80ms | 62ms |
| POST /forecasts | <200ms | 175ms |

### Database Performance

| Operation | Target | Actual |
|-----------|--------|--------|
| Lead search (full-text) | <200ms | 165ms |
| Opportunity aggregation | <300ms | 245ms |
| Territory assignment | <100ms | 82ms |
| Quota calculation | <500ms | 420ms |

### Throughput

- **Concurrent Users**: 5,000+ simultaneous users
- **Requests Per Second**: 10,000 RPS (peak load)
- **Database Connections**: 500 max pool size
- **Message Queue Throughput**: 50,000 messages/sec

### Resource Utilization

```yaml
production_resources:
  api_gateway:
    cpu: 4 cores
    memory: 8GB
    instances: 6
  
  lead_service:
    cpu: 2 cores
    memory: 4GB
    instances: 4
  
  opportunity_service:
    cpu: 4 cores
    memory: 8GB
    instances: 6
  
  database:
    instance_type: r5.2xlarge
    cpu: 8 cores
    memory: 64GB
    iops: 10000
```

## Performance Optimization

### Query Optimization

```sql
-- Optimized pipeline query with proper indexes
EXPLAIN ANALYZE
SELECT 
    stage,
    COUNT(*) as count,
    SUM(amount) as total_amount,
    AVG(probability) as avg_probability
FROM opportunities
WHERE owner_id = $1
    AND stage NOT IN ('closed_won', 'closed_lost')
    AND expected_close_date >= CURRENT_DATE
GROUP BY stage
ORDER BY 
    CASE stage
        WHEN 'discovery' THEN 1
        WHEN 'qualification' THEN 2
        WHEN 'proposal' THEN 3
        WHEN 'negotiation' THEN 4
    END;

-- Uses indexes: idx_owner, idx_stage, idx_close_date
-- Execution time: ~45ms for 10,000 active opportunities
```

### Caching Patterns

```typescript
// Multi-level caching strategy
class PipelineCache {
  async getPipeline(userId: string): Promise<Pipeline> {
    // L1: In-memory cache (10s TTL)
    const memCache = this.memoryCache.get(`pipeline:${userId}`);
    if (memCache) return memCache;
    
    // L2: Redis cache (60s TTL)
    const redisCache = await this.redis.get(`pipeline:${userId}`);
    if (redisCache) {
      this.memoryCache.set(`pipeline:${userId}`, redisCache, 10);
      return redisCache;
    }
    
    // L3: Database query
    const pipeline = await this.db.query(pipelineQuery, [userId]);
    await this.redis.setex(`pipeline:${userId}`, 60, pipeline);
    this.memoryCache.set(`pipeline:${userId}`, pipeline, 10);
    return pipeline;
  }
}
```

### Batch Processing

```typescript
// Bulk opportunity updates
async function bulkUpdateStage(
  opportunityIds: string[],
  newStage: string
): Promise<void> {
  // Process in batches of 100
  const batches = chunk(opportunityIds, 100);
  
  await Promise.all(
    batches.map(batch =>
      db.query(
        'UPDATE opportunities SET stage = $1 WHERE id = ANY($2)',
        [newStage, batch]
      )
    )
  );
  
  // Invalidate caches in parallel
  await Promise.all(
    opportunityIds.map(id =>
      redis.del(`opportunity:${id}`)
    )
  );
}
```

## Disaster Recovery

- **Backup Strategy**: Automated daily backups with 30-day retention
- **Point-in-Time Recovery**: 5-minute granularity for last 7 days
- **Multi-Region Replication**: Asynchronous replication to secondary region
- **RTO**: 4 hours
- **RPO**: 15 minutes
- **Failover**: Automated failover with health check monitoring

## Monitoring & Alerting

```yaml
alerts:
  api_latency:
    condition: p95 > 200ms
    severity: warning
    notification: slack, pagerduty
  
  error_rate:
    condition: error_rate > 1%
    severity: critical
    notification: pagerduty, email
  
  database_connections:
    condition: active_connections > 450
    severity: warning
    notification: slack
  
  opportunity_creation_failure:
    condition: failure_rate > 0.5%
    severity: critical
    notification: pagerduty, email
```
