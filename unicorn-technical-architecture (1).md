  trajectory: Trajectory!
  opportunities: [Opportunity!]!
}

type Trajectory {
  predictedMilestone: String!
  probability: Float!
  timelineMonths: Int!
  keyFactors: [String!]!
  risks: [String!]!
}
```

### SDK Design

```javascript
// JavaScript/TypeScript SDK
class UnicornSDK {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.baseUrl = 'https://api.unicorn.new/v1';
    this.ws = null;
  }

  // UAuth methods
  async connect(provider, companyId) {
    const response = await this.post('/auth/connect', {
      provider,
      company_id: companyId
    });
    return response.redirect_url;
  }

  async getConnections(companyId) {
    return this.get(`/company/${companyId}/connections`);
  }

  // Verification methods
  async verify(companyId) {
    return this.post(`/verify/${companyId}`);
  }

  async getVerification(companyId) {
    return this.get(`/company/${companyId}/verification`);
  }

  // Badge methods
  badge(companyId, options = {}) {
    const { type = 'standard', theme = 'light' } = options;
    
    return {
      url: `${this.baseUrl}/badge/${companyId}?type=${type}`,
      embed: `<iframe src="${this.baseUrl}/badge/${companyId}?type=${type}" width="320" height="180" frameborder="0"></iframe>`,
      async getData() {
        return this.get(`/badge/${companyId}/data`);
      }
    };
  }

  // Ark methods
  ark = {
    getPosition: (companyId) => {
      return this.get(`/ark/company/${companyId}/position`);
    },
    
    findOpportunities: (companyId) => {
      return this.get(`/ark/company/${companyId}/opportunities`);
    },
    
    searchCompanies: (filters) => {
      return this.post('/ark/search', filters);
    },
    
    subscribe: (companyId, callback) => {
      this.connectWebSocket();
      this.ws.on('message', (data) => {
        const message = JSON.parse(data);
        if (message.company_id === companyId) {
          callback(message);
        }
      });
    }
  };

  // Real-time subscriptions
  subscribe(companyId, events, callback) {
    this.connectWebSocket();
    
    this.ws.send(JSON.stringify({
      action: 'subscribe',
      company_id: companyId,
      events: events
    }));
    
    this.ws.on('message', (data) => {
      const message = JSON.parse(data);
      if (message.company_id === companyId && events.includes(message.event)) {
        callback(message);
      }
    });
  }

  // Helper methods
  async post(endpoint, data) {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(data)
    });
    
    if (!response.ok) {
      throw new Error(`API error: ${response.statusText}`);
    }
    
    return response.json();
  }

  async get(endpoint) {
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`
      }
    });
    
    if (!response.ok) {
      throw new Error(`API error: ${response.statusText}`);
    }
    
    return response.json();
  }

  connectWebSocket() {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      return;
    }
    
    this.ws = new WebSocket(`wss://api.unicorn.new/v1/ws?api_key=${this.apiKey}`);
    
    this.ws.on('open', () => {
      console.log('Connected to Unicorn real-time updates');
    });
    
    this.ws.on('error', (error) => {
      console.error('WebSocket error:', error);
    });
  }
}

// Usage examples
const unicorn = new UnicornSDK('uni_live_abc123');

// Connect data source
const authUrl = await unicorn.connect('stripe', 'acme-corp');
window.location.href = authUrl;

// Verify company
const verification = await unicorn.verify('acme-corp');
console.log(`Verification score: ${verification.score}`);

// Create badge
const badge = unicorn.badge('acme-corp', { type: 'detailed' });
document.getElementById('badge-container').innerHTML = badge.embed;

// Get Ark position
const position = await unicorn.ark.getPosition('acme-corp');
console.log(`Network rank: #${position.rank}`);

// Subscribe to real-time updates
unicorn.subscribe('acme-corp', ['metric_update', 'position_change'], (update) => {
  console.log('Real-time update:', update);
});
```

## Scalability Architecture

### Horizontal Scaling Strategy

```yaml
# Kubernetes Deployment Configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unicorn-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: unicorn-api
  template:
    metadata:
      labels:
        app: unicorn-api
    spec:
      containers:
      - name: api
        image: unicorn/api:latest
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: unicorn-secrets
              key: database-url
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: unicorn-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: unicorn-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Database Sharding Strategy

```python
# Database Sharding Implementation
class ShardManager:
    def __init__(self):
        self.shards = self._initialize_shards()
        self.shard_count = len(self.shards)
    
    def _initialize_shards(self):
        """Initialize connection pools for each shard"""
        shards = {}
        for i in range(SHARD_COUNT):
            shards[i] = asyncpg.create_pool(
                host=f'shard-{i}.db.unicorn.internal',
                database='unicorn',
                user=DB_USER,
                password=DB_PASSWORD,
                min_size=10,
                max_size=50
            )
        return shards
    
    def get_shard_id(self, company_id: str) -> int:
        """Determine shard based on company ID"""
        # Use consistent hashing for even distribution
        hash_value = int(hashlib.md5(company_id.encode()).hexdigest(), 16)
        return hash_value % self.shard_count
    
    async def execute_on_shard(self, company_id: str, query: str, *args):
        """Execute query on appropriate shard"""
        shard_id = self.get_shard_id(company_id)
        pool = self.shards[shard_id]
        
        async with pool.acquire() as connection:
            return await connection.fetch(query, *args)
    
    async def execute_on_all_shards(self, query: str, *args):
        """Execute query across all shards (for aggregations)"""
        tasks = []
        for shard_id, pool in self.shards.items():
            async with pool.acquire() as connection:
                tasks.append(connection.fetch(query, *args))
        
        results = await asyncio.gather(*tasks)
        # Merge results from all shards
        return self._merge_results(results)
```

### Load Balancing

```nginx
# NGINX Load Balancer Configuration
upstream unicorn_api {
    least_conn;
    
    server api-1.unicorn.internal:8000 weight=5;
    server api-2.unicorn.internal:8000 weight=5;
    server api-3.unicorn.internal:8000 weight=5;
    
    # Health checks
    keepalive 32;
    keepalive_requests 100;
    keepalive_timeout 30s;
}

upstream unicorn_websocket {
    ip_hash;  # Sticky sessions for WebSocket
    
    server ws-1.unicorn.internal:8001;
    server ws-2.unicorn.internal:8001;
    server ws-3.unicorn.internal:8001;
}

server {
    listen 443 ssl http2;
    server_name api.unicorn.new;
    
    # SSL configuration
    ssl_certificate /etc/ssl/certs/unicorn.crt;
    ssl_certificate_key /etc/ssl/private/unicorn.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # API endpoints
    location /v1/ {
        proxy_pass http://unicorn_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Caching
        proxy_cache unicorn_cache;
        proxy_cache_valid 200 5m;
        proxy_cache_use_stale error timeout invalid_header updating;
    }
    
    # WebSocket endpoints
    location /v1/ws {
        proxy_pass http://unicorn_websocket;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    # Badge serving with CDN
    location /badge/ {
        proxy_pass http://unicorn_api;
        proxy_cache badge_cache;
        proxy_cache_valid 200 5m;
        proxy_cache_key "$uri$is_args$args";
        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

### Performance Optimization

```python
# Performance Optimization Strategies
class PerformanceOptimizer:
    def __init__(self):
        self.connection_pool = self._create_connection_pool()
        self.query_cache = {}
        
    def _create_connection_pool(self):
        """Create optimized connection pool"""
        return asyncpg.create_pool(
            DATABASE_URL,
            min_size=20,
            max_size=100,
            max_queries=50000,
            max_inactive_connection_lifetime=300,
            command_timeout=60
        )
    
    async def batch_fetch_metrics(self, company_ids: List[str]) -> Dict[str, Any]:
        """Batch fetch metrics for multiple companies"""
        if not company_ids:
            return {}
        
        # Use prepared statement for performance
        query = """
        SELECT company_id, metric_name, metric_value, recorded_at
        FROM metrics
        WHERE company_id = ANY($1::uuid[])
        AND recorded_at > NOW() - INTERVAL '24 hours'
        ORDER BY company_id, metric_name, recorded_at DESC
        """
        
        async with self.connection_pool.acquire() as connection:
            # Prepare statement once
            stmt = await connection.prepare(query)
            
            # Execute with array parameter
            rows = await stmt.fetch(company_ids)
            
            # Group by company
            results = defaultdict(lambda: defaultdict(list))
            for row in rows:
                results[row['company_id']][row['metric_name']].append({
                    'value': row['metric_value'],
                    'timestamp': row['recorded_at']
                })
            
            return dict(results)
    
    async def parallel_verification(self, company_id: str) -> VerificationResult:
        """Run verification tasks in parallel"""
        tasks = [
            self.verify_financial_metrics(company_id),
            self.verify_product_metrics(company_id),
            self.verify_growth_metrics(company_id),
            self.verify_team_metrics(company_id)
        ]
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Aggregate results
        verification_result = VerificationResult()
        for result in results:
            if isinstance(result, Exception):
                logger.error(f"Verification task failed: {result}")
                continue
            verification_result.merge(result)
        
        return verification_result
```

## Implementation Timeline

### Phase 1: Core Infrastructure (Weeks 1-4)

**Week 1-2:**
- Set up development environment
- Initialize database schemas
- Implement basic authentication
- Create project structure

**Week 3-4:**
- Deploy Kubernetes cluster
- Set up CI/CD pipeline
- Implement basic API endpoints
- Create monitoring infrastructure

### Phase 2: UAuth Implementation (Weeks 5-8)

**Week 5-6:**
- OAuth2 flow for Stripe and QuickBooks
- Token encryption and storage
- Connection manager service
- Basic data normalization

**Week 7-8:**
- Additional provider integrations
- Webhook listeners
- Real-time sync system
- Security hardening

### Phase 3: Badge System (Weeks 9-12)

**Week 9-10:**
- Badge generator service
- Rendering engine
- CDN distribution
- Embed code generation

**Week 11-12:**
- Real-time updates via WebSocket
- Analytics tracking
- Multiple badge types
- Performance optimization

### Phase 4: The Ark (Weeks 13-16)

**Week 13-14:**
- Neo4j setup and schema
- Basic graph operations
- Position calculation
- Simple visualization

**Week 15-16:**
- Pattern recognition
- Intelligence API
- Advanced visualization
- WebSocket updates

### Phase 5: Integration & Testing (Weeks 17-20)

**Week 17-18:**
- End-to-end testing
- Performance testing
- Security audit
- API documentation

**Week 19-20:**
- SDK development
- Developer portal
- Beta launch preparation
- Final optimizations

## Monitoring & Observability

### Metrics Collection

```python
# Prometheus Metrics
from prometheus_client import Counter, Histogram, Gauge
import time

# Define metrics
api_requests = Counter('unicorn_api_requests_total', 'Total API requests', ['method', 'endpoint', 'status'])
api_latency = Histogram('unicorn_api_latency_seconds', 'API latency', ['method', 'endpoint'])
active_connections = Gauge('unicorn_active_connections', 'Active WebSocket connections')
verification_score = Histogram('unicorn_verification_scores', 'Distribution of verification scores')

# Middleware for automatic metrics
async def metrics_middleware(request, call_next):
    start_time = time.time()
    
    # Process request
    response = await call_next(request)
    
    # Record metrics
    api_requests.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()
    
    api_latency.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(time.time() - start_time)
    
    return response
```

### Logging Strategy

```python
# Structured Logging
import structlog
from pythonjsonlogger import jsonlogger

# Configure structured logging
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.render_to_log_kwargs,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Usage
logger.info("verification_completed", 
    company_id=company_id,
    score=verification_result.score,
    duration_ms=duration_ms,
    sources=verification_result.sources
)
```

### Error Tracking

```python
# Sentry Integration
import sentry_sdk
from sentry_sdk.integrations.asgi import SentryAsgiMiddleware

sentry_sdk.init(
    dsn=SENTRY_DSN,
    environment=ENVIRONMENT,
    traces_sample_rate=0.1,
    profiles_sample_rate=0.1,
)

# Wrap FastAPI app
app = FastAPI()
app.add_middleware(SentryAsgiMiddleware)

# Custom error tracking
def track_error(error: Exception, context: dict = None):
    with sentry_sdk.push_scope() as scope:
        if context:
            for key, value in context.items():
                scope.set_context(key, value)
        sentry_sdk.capture_exception(error)
```

## Conclusion

This technical architecture provides a comprehensive foundation for building Unicorn Protocol's three pillars:

1. **UAuth**: A secure, scalable authentication system that connects to multiple data sources
2. **Badge System**: A real-time visualization engine with CDN distribution
3. **The Ark**: A graph-based intelligence platform with pattern recognition

The architecture emphasizes:
- **Scalability**: Horizontal scaling, sharding, and caching strategies
- **Security**: Encryption, authentication, and audit trails
- **Performance**: Optimized queries, parallel processing, and CDN distribution
- **Reliability**: Health checks, circuit breakers, and graceful degradation
- **Developer Experience**: Clean APIs, comprehensive SDKs, and clear documentation

By following this implementation guide, Unicorn Protocol can build the technical infrastructure needed to become the trust layer for the global startup ecosystem.# Unicorn Protocol: Technical Architecture & Implementation Guide

## Table of Contents

1. [System Overview](#system-overview)
2. [UAuth: Universal Authentication System](#uauth-universal-authentication-system)
3. [Badge System: Trust Visualization Engine](#badge-system-trust-visualization-engine)
4. [The Ark: Graph Intelligence Platform](#the-ark-graph-intelligence-platform)
5. [Core Infrastructure](#core-infrastructure)
6. [Data Architecture](#data-architecture)
7. [Security Architecture](#security-architecture)
8. [API Design](#api-design)
9. [Scalability Architecture](#scalability-architecture)
10. [Implementation Timeline](#implementation-timeline)

## System Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           CLIENT LAYER                              │
├─────────────────────────────────────────────────────────────────────┤
│   Web App    Mobile Apps    Embedded Badges    Developer SDKs      │
│   (React)    (React Native)    (iframe)       (JS/Python/Go)       │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          API GATEWAY                                │
├─────────────────────────────────────────────────────────────────────┤
│   Load Balancing    Rate Limiting    Auth    Request Routing       │
│        (ALB)         (Redis)       (JWT)      (Kong/Envoy)         │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         SERVICE LAYER                               │
├──────────────────┬──────────────────┬──────────────────────────────┤
│     UAuth        │     Badge        │        The Ark              │
│    Service       │    Service       │       Service               │
├──────────────────┼──────────────────┼──────────────────────────────┤
│  Verification    │   Analytics      │    Intelligence             │
│   Service        │    Service       │     Service                 │
└──────────────────┴──────────────────┴──────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          DATA LAYER                                 │
├─────────────────────────────────────────────────────────────────────┤
│  PostgreSQL    Neo4j Graph    Redis Cache    S3 Storage    Kafka   │
│  (Primary DB)  (Relationships) (Real-time)   (Documents)  (Events) │
└─────────────────────────────────────────────────────────────────────┘
```

### Technology Stack

- **Languages**: Python (FastAPI), TypeScript (Node.js), Go (performance-critical services)
- **Databases**: PostgreSQL (ACID compliance), Neo4j (graph relationships), Redis (caching)
- **Message Queue**: Apache Kafka (event streaming)
- **Container Orchestration**: Kubernetes (EKS/GKE)
- **Monitoring**: Prometheus + Grafana, Datadog
- **CI/CD**: GitHub Actions, ArgoCD
- **Infrastructure as Code**: Terraform

## UAuth: Universal Authentication System

### Technical Overview

UAuth is a OAuth2-based authentication system that securely connects startup data sources to create a unified verification layer.

### Architecture Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                         UAuth Architecture                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────┐     ┌──────────────┐     ┌──────────────────┐  │
│  │               │     │              │     │                  │  │
│  │  OAuth2       │────▶│  Connection  │────▶│  Data           │  │
│  │  Handler      │     │  Manager     │     │  Normalizer     │  │
│  │               │     │              │     │                  │  │
│  └───────────────┘     └──────────────┘     └──────────────────┘  │
│          │                     │                      │             │
│          ▼                     ▼                      ▼             │
│  ┌───────────────┐     ┌──────────────┐     ┌──────────────────┐  │
│  │               │     │              │     │                  │  │
│  │  Token        │     │  Webhook     │     │  Verification   │  │
│  │  Storage      │     │  Listener    │     │  Engine         │  │
│  │               │     │              │     │                  │  │
│  └───────────────┘     └──────────────┘     └──────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementation Details

#### 1. OAuth2 Flow Implementation

```python
# FastAPI OAuth2 Handler
from fastapi import FastAPI, HTTPException
from authlib.integrations.starlette_client import OAuth
import jwt
from datetime import datetime, timedelta

class UAuthHandler:
    def __init__(self):
        self.oauth = OAuth()
        self.providers = self._initialize_providers()
    
    def _initialize_providers(self):
        return {
            'stripe': self.oauth.register(
                name='stripe',
                client_id=STRIPE_CLIENT_ID,
                client_secret=STRIPE_CLIENT_SECRET,
                authorize_url='https://connect.stripe.com/oauth/authorize',
                access_token_url='https://connect.stripe.com/oauth/token',
                client_kwargs={'scope': 'read_only'}
            ),
            'quickbooks': self.oauth.register(
                name='quickbooks',
                client_id=QB_CLIENT_ID,
                client_secret=QB_CLIENT_SECRET,
                authorize_url='https://appcenter.intuit.com/connect/oauth2',
                access_token_url='https://oauth.platform.intuit.com/oauth2/v1/tokens/bearer'
            ),
            # Additional providers...
        }
    
    async def initiate_connection(self, provider: str, company_id: str):
        """Initiate OAuth2 flow for a provider"""
        state = self.generate_state(company_id, provider)
        redirect_uri = f"{BASE_URL}/auth/callback/{provider}"
        
        return await self.providers[provider].authorize_redirect(
            redirect_uri=redirect_uri,
            state=state
        )
    
    def generate_state(self, company_id: str, provider: str):
        """Generate secure state parameter with embedded metadata"""
        payload = {
            'company_id': company_id,
            'provider': provider,
            'timestamp': datetime.utcnow().isoformat(),
            'exp': datetime.utcnow() + timedelta(minutes=10)
        }
        return jwt.encode(payload, SECRET_KEY, algorithm='HS256')
```

#### 2. Connection Manager

```python
class ConnectionManager:
    def __init__(self, db_pool, redis_client):
        self.db = db_pool
        self.redis = redis_client
        self.encryptor = Fernet(ENCRYPTION_KEY)
    
    async def store_connection(self, company_id: str, provider: str, tokens: dict):
        """Securely store OAuth tokens and connection metadata"""
        encrypted_tokens = self.encryptor.encrypt(json.dumps(tokens).encode())
        
        connection = {
            'company_id': company_id,
            'provider': provider,
            'encrypted_tokens': encrypted_tokens,
            'connected_at': datetime.utcnow(),
            'status': 'active',
            'last_sync': None,
            'sync_frequency': self._determine_sync_frequency(provider)
        }
        
        # Store in PostgreSQL
        await self.db.execute(
            """
            INSERT INTO connections (company_id, provider, encrypted_tokens, 
                                   connected_at, status, sync_frequency)
            VALUES ($1, $2, $3, $4, $5, $6)
            ON CONFLICT (company_id, provider) 
            DO UPDATE SET encrypted_tokens = $3, status = $5
            """,
            connection['company_id'],
            connection['provider'],
            connection['encrypted_tokens'],
            connection['connected_at'],
            connection['status'],
            connection['sync_frequency']
        )
        
        # Cache active connection in Redis
        await self.redis.setex(
            f"connection:{company_id}:{provider}",
            3600,  # 1 hour TTL
            json.dumps({'status': 'active', 'connected_at': connection['connected_at'].isoformat()})
        )
    
    def _determine_sync_frequency(self, provider: str) -> int:
        """Determine optimal sync frequency based on provider type"""
        frequencies = {
            'stripe': 300,      # 5 minutes - financial data is critical
            'quickbooks': 900,  # 15 minutes
            'google_analytics': 3600,  # 1 hour
            'github': 7200,     # 2 hours
            'mixpanel': 1800    # 30 minutes
        }
        return frequencies.get(provider, 3600)
```

#### 3. Data Normalizer

```python
class DataNormalizer:
    def __init__(self):
        self.normalizers = {
            'stripe': StripeNormalizer(),
            'quickbooks': QuickBooksNormalizer(),
            'google_analytics': GoogleAnalyticsNormalizer(),
            'mixpanel': MixpanelNormalizer()
        }
    
    async def normalize(self, provider: str, raw_data: dict) -> dict:
        """Normalize data from different providers into standard schema"""
        normalizer = self.normalizers.get(provider)
        if not normalizer:
            raise ValueError(f"No normalizer found for provider: {provider}")
        
        return await normalizer.normalize(raw_data)

class StripeNormalizer:
    async def normalize(self, raw_data: dict) -> dict:
        """Convert Stripe data to standard metrics schema"""
        subscriptions = raw_data.get('subscriptions', {}).get('data', [])
        customers = raw_data.get('customers', {}).get('data', [])
        charges = raw_data.get('charges', {}).get('data', [])
        
        # Calculate MRR
        mrr = sum(
            sub['plan']['amount'] / 100 
            for sub in subscriptions 
            if sub['status'] == 'active'
        )
        
        # Calculate churn
        total_customers = len(customers)
        churned_customers = len([c for c in customers if c['subscriptions']['total_count'] == 0])
        churn_rate = (churned_customers / total_customers * 100) if total_customers > 0 else 0
        
        # Calculate growth rate
        current_month_revenue = self._calculate_period_revenue(charges, 0)
        previous_month_revenue = self._calculate_period_revenue(charges, 1)
        growth_rate = ((current_month_revenue - previous_month_revenue) / previous_month_revenue * 100) if previous_month_revenue > 0 else 0
        
        return {
            'financial': {
                'mrr': mrr,
                'arr': mrr * 12,
                'revenue_growth_rate': growth_rate,
                'average_revenue_per_customer': mrr / len(customers) if customers else 0
            },
            'customers': {
                'total': total_customers,
                'active': len([c for c in customers if c['subscriptions']['total_count'] > 0]),
                'churn_rate': churn_rate
            },
            'metadata': {
                'source': 'stripe',
                'normalized_at': datetime.utcnow().isoformat(),
                'data_quality_score': self._calculate_data_quality(raw_data)
            }
        }
```

#### 4. Webhook Listener

```python
class WebhookListener:
    def __init__(self, verification_engine):
        self.verification_engine = verification_engine
        self.handlers = {
            'stripe': self._handle_stripe_webhook,
            'quickbooks': self._handle_quickbooks_webhook,
            'mixpanel': self._handle_mixpanel_webhook
        }
    
    async def handle_webhook(self, provider: str, headers: dict, body: bytes):
        """Process incoming webhooks from data providers"""
        # Verify webhook signature
        if not await self._verify_webhook_signature(provider, headers, body):
            raise HTTPException(status_code=401, detail="Invalid webhook signature")
        
        # Parse and handle webhook
        handler = self.handlers.get(provider)
        if not handler:
            raise HTTPException(status_code=400, detail=f"No handler for provider: {provider}")
        
        return await handler(json.loads(body))
    
    async def _handle_stripe_webhook(self, event: dict):
        """Handle Stripe webhook events"""
        event_type = event.get('type')
        company_id = event.get('account')  # Stripe Connect account ID
        
        if event_type in ['customer.subscription.created', 'customer.subscription.updated', 'customer.subscription.deleted']:
            # Trigger immediate re-verification
            await self.verification_engine.verify_company(company_id, providers=['stripe'])
        
        # Store event for audit
        await self._store_webhook_event(company_id, 'stripe', event)
```

### UAuth Security Measures

1. **Token Encryption**: All OAuth tokens encrypted at rest using Fernet
2. **State Validation**: JWT-based state parameters with expiration
3. **Webhook Verification**: Signature validation for all incoming webhooks
4. **Rate Limiting**: Per-company and per-provider rate limits
5. **Audit Logging**: Complete audit trail of all authentication events

## Badge System: Trust Visualization Engine

### Technical Overview

The Badge System generates real-time, embeddable trust artifacts that visualize verified startup metrics.

### Architecture Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Badge System Architecture                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────┐     ┌──────────────┐     ┌──────────────────┐  │
│  │               │     │              │     │                  │  │
│  │  Badge        │────▶│  Rendering   │────▶│  CDN            │  │
│  │  Generator    │     │  Engine      │     │  Distribution   │  │
│  │               │     │              │     │                  │  │
│  └───────────────┘     └──────────────┘     └──────────────────┘  │
│          │                     │                      │             │
│          ▼                     ▼                      ▼             │
│  ┌───────────────┐     ┌──────────────┐     ┌──────────────────┐  │
│  │               │     │              │     │                  │  │
│  │  Real-time    │     │  Cache       │     │  Analytics      │  │
│  │  Updates      │     │  Layer       │     │  Tracker        │  │
│  │               │     │              │     │                  │  │
│  └───────────────┘     └──────────────┘     └──────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementation Details

#### 1. Badge Generator Service

```typescript
// TypeScript/Node.js Badge Generator
import { createCanvas, loadImage, registerFont } from 'canvas';
import { Redis } from 'ioredis';
import { S3 } from 'aws-sdk';

class BadgeGenerator {
  private redis: Redis;
  private s3: S3;
  private wsConnections: Map<string, WebSocket[]>;

  constructor() {
    this.redis = new Redis(REDIS_CONFIG);
    this.s3 = new S3(S3_CONFIG);
    this.wsConnections = new Map();
    
    // Register custom fonts
    registerFont('./fonts/Inter-Bold.ttf', { family: 'Inter', weight: 'bold' });
    registerFont('./fonts/Inter-Regular.ttf', { family: 'Inter' });
  }

  async generateBadge(companyId: string, type: BadgeType): Promise<BadgeResult> {
    // Check cache first
    const cacheKey = `badge:${companyId}:${type}`;
    const cached = await this.redis.get(cacheKey);
    
    if (cached && type !== 'live') {
      return JSON.parse(cached);
    }

    // Fetch verified metrics
    const metrics = await this.fetchVerifiedMetrics(companyId);
    
    // Generate badge based on type
    let badge: BadgeResult;
    switch (type) {
      case 'minimal':
        badge = await this.generateMinimalBadge(metrics);
        break;
      case 'standard':
        badge = await this.generateStandardBadge(metrics);
        break;
      case 'detailed':
        badge = await this.generateDetailedBadge(metrics);
        break;
      case 'live':
        badge = await this.generateLiveBadge(metrics);
        break;
    }

    // Cache non-live badges
    if (type !== 'live') {
      await this.redis.setex(cacheKey, 300, JSON.stringify(badge)); // 5 min cache
    }

    return badge;
  }

  private async generateStandardBadge(metrics: VerifiedMetrics): Promise<BadgeResult> {
    const width = 320;
    const height = 180;
    const canvas = createCanvas(width, height);
    const ctx = canvas.getContext('2d');

    // Background gradient
    const gradient = ctx.createLinearGradient(0, 0, width, height);
    gradient.addColorStop(0, '#667eea');
    gradient.addColorStop(1, '#764ba2');
    ctx.fillStyle = gradient;
    ctx.roundRect(0, 0, width, height, 12);
    ctx.fill();

    // Add shimmer effect
    ctx.globalAlpha = 0.1;
    const shimmer = ctx.createRadialGradient(width * 0.7, height * 0.3, 0, width * 0.7, height * 0.3, width);
    shimmer.addColorStop(0, '#ffffff');
    shimmer.addColorStop(1, 'transparent');
    ctx.fillStyle = shimmer;
    ctx.fillRect(0, 0, width, height);
    ctx.globalAlpha = 1;

    // Header
    ctx.fillStyle = '#ffffff';
    ctx.font = 'bold 14px Inter';
    ctx.fillText('UNICORN VERIFIED', 20, 30);

    // Verification mark
    ctx.beginPath();
    ctx.arc(width - 30, 30, 12, 0, Math.PI * 2);
    ctx.fillStyle = 'rgba(255, 255, 255, 0.2)';
    ctx.fill();
    ctx.fillStyle = '#ffffff';
    ctx.font = '16px Inter';
    ctx.fillText('✓', width - 36, 36);

    // Score
    ctx.font = 'bold 36px Inter';
    ctx.fillText(metrics.score.toString(), 20, 80);
    ctx.font = '14px Inter';
    ctx.fillText(metrics.stage.toUpperCase(), 20, 100);

    // Metrics bars
    this.drawMetricBar(ctx, 'MRR', metrics.mrr_percentile, 20, 120);
    this.drawMetricBar(ctx, 'GROWTH', metrics.growth_percentile, 20, 140);
    this.drawMetricBar(ctx, 'EFFICIENCY', metrics.efficiency_percentile, 20, 160);

    // Footer
    ctx.font = '12px Inter';
    ctx.fillStyle = 'rgba(255, 255, 255, 0.8)';
    const verifiedTime = this.getRelativeTime(metrics.verified_at);
    ctx.fillText(`✓ Verified ${verifiedTime}`, 20, height - 10);

    // Convert to buffer
    const buffer = canvas.toBuffer('image/png');
    
    // Upload to S3
    const s3Key = `badges/${metrics.company_id}/standard-${Date.now()}.png`;
    await this.s3.putObject({
      Bucket: BADGE_BUCKET,
      Key: s3Key,
      Body: buffer,
      ContentType: 'image/png',
      CacheControl: 'max-age=300'
    }).promise();

    return {
      type: 'standard',
      url: `${CDN_URL}/${s3Key}`,
      embedCode: this.generateEmbedCode(metrics.company_id, 'standard'),
      metrics: {
        score: metrics.score,
        stage: metrics.stage,
        percentiles: {
          mrr: metrics.mrr_percentile,
          growth: metrics.growth_percentile,
          efficiency: metrics.efficiency_percentile
        }
      }
    };
  }

  private drawMetricBar(ctx: CanvasRenderingContext2D, label: string, percentile: number, x: number, y: number) {
    // Label
    ctx.fillStyle = 'rgba(255, 255, 255, 0.8)';
    ctx.font = '10px Inter';
    ctx.fillText(label, x, y);

    // Background bar
    ctx.fillStyle = 'rgba(255, 255, 255, 0.2)';
    ctx.fillRect(x + 60, y - 8, 200, 6);

    // Progress bar
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(x + 60, y - 8, 200 * (percentile / 100), 6);
  }

  private generateEmbedCode(companyId: string, type: string): string {
    return `<iframe src="${BASE_URL}/badge/${companyId}?type=${type}" width="320" height="180" frameborder="0" style="border-radius: 12px;"></iframe>`;
  }
}
```

#### 2. Real-time Update System

```typescript
class RealTimeUpdater {
  private kafka: Kafka;
  private consumer: Consumer;
  private wsServer: WebSocketServer;
  
  constructor() {
    this.kafka = new Kafka({
      clientId: 'badge-updater',
      brokers: KAFKA_BROKERS
    });
    
    this.consumer = this.kafka.consumer({ groupId: 'badge-update-group' });
    this.wsServer = new WebSocketServer({ port: WS_PORT });
    
    this.setupWebSocketServer();
    this.subscribeToUpdates();
  }
  
  private async subscribeToUpdates() {
    await this.consumer.connect();
    await this.consumer.subscribe({ topic: 'metric-updates', fromBeginning: false });
    
    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const update = JSON.parse(message.value.toString());
        await this.handleMetricUpdate(update);
      }
    });
  }
  
  private async handleMetricUpdate(update: MetricUpdate) {
    const { companyId, metrics, timestamp } = update;
    
    // Update cache
    await this.updateMetricCache(companyId, metrics);
    
    // Notify WebSocket clients
    this.broadcastUpdate(companyId, {
      type: 'metric_update',
      metrics,
      timestamp,
      changes: this.calculateChanges(companyId, metrics)
    });
    
    // Trigger badge regeneration
    await this.triggerBadgeRegeneration(companyId);
  }
  
  private broadcastUpdate(companyId: string, update: any) {
    const connections = this.wsConnections.get(companyId) || [];
    
    connections.forEach(ws => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify(update));
      }
    });
  }
  
  private setupWebSocketServer() {
    this.wsServer.on('connection', (ws, req) => {
      const companyId = this.extractCompanyId(req.url);
      
      if (!companyId) {
        ws.close(1008, 'Company ID required');
        return;
      }
      
      // Add to connection pool
      if (!this.wsConnections.has(companyId)) {
        this.wsConnections.set(companyId, []);
      }
      this.wsConnections.get(companyId).push(ws);
      
      // Send initial state
      this.sendInitialState(ws, companyId);
      
      // Handle disconnection
      ws.on('close', () => {
        const connections = this.wsConnections.get(companyId) || [];
        const index = connections.indexOf(ws);
        if (index > -1) {
          connections.splice(index, 1);
        }
      });
    });
  }
}
```

#### 3. Badge Analytics Tracker

```python
class BadgeAnalytics:
    def __init__(self, clickhouse_client, redis_client):
        self.clickhouse = clickhouse_client
        self.redis = redis_client
        
    async def track_impression(self, badge_data: dict):
        """Track badge impression (view)"""
        event = {
            'event_type': 'badge_impression',
            'company_id': badge_data['company_id'],
            'badge_type': badge_data['badge_type'],
            'viewer_ip': badge_data['viewer_ip'],
            'referrer': badge_data['referrer'],
            'timestamp': datetime.utcnow(),
            'user_agent': badge_data['user_agent']
        }
        
        # Stream to ClickHouse for analytics
        await self.clickhouse.execute(
            """
            INSERT INTO badge_events 
            (event_type, company_id, badge_type, viewer_ip, referrer, timestamp, user_agent)
            VALUES
            """,
            event
        )
        
        # Update real-time counters in Redis
        await self.redis.hincrby(f"badge:impressions:{badge_data['company_id']}", 'total', 1)
        await self.redis.hincrby(f"badge:impressions:{badge_data['company_id']}", badge_data['badge_type'], 1)
        
        # Detect investor interest
        if await self._is_investor_ip(badge_data['viewer_ip']):
            await self._track_investor_interest(badge_data)
    
    async def _is_investor_ip(self, ip: str) -> bool:
        """Check if IP belongs to known investor"""
        # Check against known investor IP ranges
        investor_data = await self.redis.get(f"investor:ip:{ip}")
        return investor_data is not None
    
    async def _track_investor_interest(self, badge_data: dict):
        """Track when investors view badges"""
        await self.redis.zadd(
            f"investor:interest:{badge_data['company_id']}",
            {badge_data['viewer_ip']: datetime.utcnow().timestamp()}
        )
        
        # Notify company of investor interest (if premium feature enabled)
        await self._notify_investor_interest(badge_data['company_id'], badge_data['viewer_ip'])
```

## The Ark: Graph Intelligence Platform

### Technical Overview

The Ark is a graph-based intelligence platform that maps relationships between verified entities and generates ecosystem-wide insights.

### Architecture Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                        The Ark Architecture                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────┐     ┌──────────────┐     ┌──────────────────┐  │
│  │               │     │              │     │                  │  │
│  │  Graph        │────▶│  Pattern     │────▶│  Intelligence   │  │
│  │  Engine       │     │  Recognition │     │  API            │  │
│  │  (Neo4j)      │     │              │     │                  │  │
│  └───────────────┘     └──────────────┘     └──────────────────┘  │
│          │                     │                      │             │
│          ▼                     ▼                      ▼             │
│  ┌───────────────┐     ┌──────────────┐     ┌──────────────────┐  │
│  │               │     │              │     │                  │  │
│  │  Relationship │     │  Predictive  │     │  Visualization  │  │
│  │  Inference    │     │  Modeling    │     │  Engine         │  │
│  │               │     │              │     │                  │  │
│  └───────────────┘     └──────────────┘     └──────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementation Details

#### 1. Graph Engine (Neo4j)

```python
# Python Neo4j Graph Engine
from neo4j import AsyncGraphDatabase
import numpy as np
from typing import List, Dict, Optional

class ArkGraphEngine:
    def __init__(self, uri: str, auth: tuple):
        self.driver = AsyncGraphDatabase.driver(uri, auth=auth)
        
    async def create_company_node(self, company_data: dict):
        """Create or update a company node in the graph"""
        async with self.driver.session() as session:
            await session.write_transaction(self._create_company, company_data)
    
    @staticmethod
    async def _create_company(tx, company_data):
        query = """
        MERGE (c:Company {id: $company_id})
        SET c.name = $name,
            c.stage = $stage,
            c.score = $score,
            c.mrr = $mrr,
            c.growth_rate = $growth_rate,
            c.efficiency_score = $efficiency_score,
            c.verified_at = datetime($verified_at),
            c.sector = $sector,
            c.location = point({latitude: $lat, longitude: $lon})
        """
        await tx.run(query, **company_data)
    
    async def create_relationship(self, from_id: str, to_id: str, rel_type: str, properties: dict = None):
        """Create a relationship between two entities"""
        async with self.driver.session() as session:
            await session.write_transaction(
                self._create_relationship, 
                from_id, to_id, rel_type, properties or {}
            )
    
    @staticmethod
    async def _create_relationship(tx, from_id, to_id, rel_type, properties):
        # Sanitize relationship type for Cypher
        rel_type = rel_type.upper().replace(' ', '_')
        
        query = f"""
        MATCH (a {{id: $from_id}})
        MATCH (b {{id: $to_id}})
        MERGE (a)-[r:{rel_type}]->(b)
        SET r += $properties
        SET r.created_at = datetime()
        SET r.strength = COALESCE(r.strength, 1.0)
        """
        await tx.run(query, from_id=from_id, to_id=to_id, properties=properties)
    
    async def find_network_position(self, company_id: str) -> dict:
        """Calculate a company's position in the network"""
        async with self.driver.session() as session:
            # Get centrality metrics
            centrality = await session.read_transaction(
                self._calculate_centrality, company_id
            )
            
            # Get clustering coefficient
            clustering = await session.read_transaction(
                self._calculate_clustering, company_id
            )
            
            # Get growth trajectory neighbors
            similar_companies = await session.read_transaction(
                self._find_similar_trajectory, company_id
            )
            
            return {
                'centrality': centrality,
                'clustering': clustering,
                'similar_companies': similar_companies,
                'network_rank': await self._calculate_network_rank(company_id)
            }
    
    @staticmethod
    async def _calculate_centrality(tx, company_id):
        """Calculate various centrality measures"""
        # Degree centrality
        degree_query = """
        MATCH (c:Company {id: $company_id})-[r]-()
        RETURN count(r) as degree_centrality
        """
        degree_result = await tx.run(degree_query, company_id=company_id)
        degree = await degree_result.single()
        
        # Betweenness centrality (simplified for performance)
        betweenness_query = """
        MATCH p = allShortestPaths((a:Company)-[*..3]-(b:Company))
        WHERE a <> b AND (a)-[]-(c:Company {id: $company_id})-[]-(b)
        RETURN count(p) as betweenness_centrality
        """
        betweenness_result = await tx.run(betweenness_query, company_id=company_id)
        betweenness = await betweenness_result.single()
        
        return {
            'degree': degree['degree_centrality'],
            'betweenness': betweenness['betweenness_centrality']
        }
```

#### 2. Pattern Recognition Engine

```python
class PatternRecognitionEngine:
    def __init__(self, graph_engine: ArkGraphEngine, ml_model_path: str):
        self.graph = graph_engine
        self.models = self._load_models(ml_model_path)
        
    async def identify_success_patterns(self, company_id: str) -> List[Pattern]:
        """Identify success patterns for a company based on similar companies"""
        # Get company data and trajectory
        company_data = await self.graph.get_company_data(company_id)
        
        # Find similar companies that succeeded
        similar_successful = await self._find_similar_successful_companies(company_data)
        
        # Extract patterns from their journeys
        patterns = []
        for company in similar_successful:
            journey = await self._extract_company_journey(company['id'])
            pattern = self._analyze_journey_pattern(journey, company_data)
            if pattern.confidence > 0.7:
                patterns.append(pattern)
        
        # Rank patterns by relevance and confidence
        return sorted(patterns, key=lambda p: p.relevance * p.confidence, reverse=True)[:5]
    
    async def _find_similar_successful_companies(self, company_data: dict) -> List[dict]:
        """Find companies with similar characteristics that achieved success"""
        query = """
        MATCH (c:Company)
        WHERE c.stage IN ['series-b', 'series-c', 'acquired', 'ipo']
        AND abs(c.initial_mrr - $mrr) < $mrr * 0.3
        AND c.sector = $sector
        WITH c, 
             abs(c.initial_growth_rate - $growth_rate) as growth_diff,
             abs(c.initial_efficiency - $efficiency) as eff_diff
        ORDER BY growth_diff + eff_diff
        LIMIT 20
        RETURN c
        """
        
        async with self.graph.driver.session() as session:
            result = await session.run(query, 
                mrr=company_data['mrr'],
                sector=company_data['sector'],
                growth_rate=company_data['growth_rate'],
                efficiency=company_data['efficiency_score']
            )
            return [record['c'] async for record in result]
    
    async def predict_trajectory(self, company_id: str) -> TrajectoryPrediction:
        """Predict company's future trajectory based on current metrics and patterns"""
        # Get current state
        current_state = await self._get_company_state_vector(company_id)
        
        # Get similar company trajectories
        similar_trajectories = await self._get_similar_trajectories(current_state)
        
        # Apply ML model for prediction
        prediction = self.models['trajectory_predictor'].predict(
            current_state,
            similar_trajectories
        )
        
        return TrajectoryPrediction(
            next_milestone=prediction['next_milestone'],
            probability=prediction['probability'],
            timeline_months=prediction['timeline'],
            key_factors=prediction['factors'],
            risks=prediction['risks']
        )
```

#### 3. Intelligence API

```python
# FastAPI Intelligence API
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List, Optional
import asyncio

app = FastAPI()

class IntelligenceAPI:
    def __init__(self, graph_engine, pattern_engine, prediction_engine):
        self.graph = graph_engine
        self.patterns = pattern_engine
        self.predictions = prediction_engine
        self.websocket_manager = ConnectionManager()
    
    @app.get("/ark/company/{company_id}/position")
    async def get_network_position(company_id: str):
        """Get a company's position in the Ark"""
        position = await self.graph.find_network_position(company_id)
        
        # Enhance with comparative context
        position['percentile_rank'] = await self._calculate_percentile_rank(
            company_id, 
            position['network_rank']
        )
        
        position['ecosystem_health'] = await self._assess_ecosystem_health(
            position['similar_companies']
        )
        
        return {
            'company_id': company_id,
            'position': position,
            'insights': await self._generate_position_insights(position)
        }
    
    @app.get("/ark/company/{company_id}/opportunities")
    async def find_opportunities(company_id: str):
        """Find strategic opportunities for a company"""
        # Partnership opportunities
        partnerships = await self._find_partnership_opportunities(company_id)
        
        # Investment opportunities
        investors = await self._find_relevant_investors(company_id)
        
        # Talent opportunities
        talent = await self._find_talent_connections(company_id)
        
        # Market opportunities
        market = await self._analyze_market_gaps(company_id)
        
        return {
            'partnerships': partnerships[:5],
            'investors': investors[:10],
            'talent': talent[:5],
            'market_gaps': market[:3]
        }
    
    async def _find_partnership_opportunities(self, company_id: str) -> List[dict]:
        """Find potential strategic partnerships"""
        query = """
        // Find companies with complementary characteristics
        MATCH (source:Company {id: $company_id})
        MATCH (target:Company)
        WHERE target.id <> source.id
        AND NOT (source)-[:PARTNER_WITH]-(target)
        
        // Calculate synergy score based on:
        // 1. Shared customers (indirect relationship)
        // 2. Complementary metrics
        // 3. Similar growth trajectory
        
        OPTIONAL MATCH (source)-[:SERVES]->(c:Customer)<-[:SERVES]-(target)
        WITH source, target, count(c) as shared_customers
        
        WITH target, shared_customers,
             abs(source.growth_rate - target.growth_rate) as growth_similarity,
             CASE 
                WHEN source.sector <> target.sector THEN 1.5 
                ELSE 1.0 
             END as sector_multiplier
        
        WITH target, 
             (shared_customers * 10 + (100 - growth_similarity)) * sector_multiplier as synergy_score
        
        WHERE synergy_score > 50
        ORDER BY synergy_score DESC
        LIMIT 10
        
        RETURN target, synergy_score
        """
        
        async with self.graph.driver.session() as session:
            result = await session.run(query, company_id=company_id)
            
            partnerships = []
            async for record in result:
                target = record['target']
                partnerships.append({
                    'company': {
                        'id': target['id'],
                        'name': target['name'],
                        'stage': target['stage'],
                        'score': target['score']
                    },
                    'synergy_score': record['synergy_score'],
                    'rationale': await self._generate_partnership_rationale(
                        company_id, 
                        target['id'], 
                        record['synergy_score']
                    )
                })
            
            return partnerships
    
    @app.websocket("/ark/ws/{company_id}")
    async def websocket_endpoint(websocket: WebSocket, company_id: str):
        """WebSocket for real-time Ark updates"""
        await self.websocket_manager.connect(websocket, company_id)
        
        try:
            # Send initial state
            initial_state = await self.get_network_position(company_id)
            await websocket.send_json({
                'type': 'initial_state',
                'data': initial_state
            })
            
            # Keep connection alive and send updates
            while True:
                # Wait for updates or ping
                update = await self.websocket_manager.get_update(company_id)
                if update:
                    await websocket.send_json({
                        'type': 'position_update',
                        'data': update
                    })
                
                await asyncio.sleep(1)
                
        except WebSocketDisconnect:
            self.websocket_manager.disconnect(websocket, company_id)
```

#### 4. Visualization Engine

```typescript
// D3.js-based Ark Visualization Engine
import * as d3 from 'd3';
import { ForceSimulation, SimulationNodeDatum, SimulationLinkDatum } from 'd3-force';

interface ArkNode extends SimulationNodeDatum {
  id: string;
  name: string;
  score: number;
  stage: string;
  type: 'company' | 'investor' | 'service';
  verified: boolean;
}

interface ArkLink extends SimulationLinkDatum<ArkNode> {
  source: string | ArkNode;
  target: string | ArkNode;
  strength: number;
  type: string;
}

class ArkVisualization {
  private svg: d3.Selection<SVGSVGElement, unknown, HTMLElement, any>;
  private simulation: ForceSimulation<ArkNode, ArkLink>;
  private nodes: ArkNode[] = [];
  private links: ArkLink[] = [];
  private centerCompanyId: string;
  
  constructor(container: HTMLElement, centerCompanyId: string) {
    this.centerCompanyId = centerCompanyId;
    this.initializeSVG(container);
    this.setupSimulation();
    this.loadData();
  }
  
  private initializeSVG(container: HTMLElement) {
    const { width, height } = container.getBoundingClientRect();
    
    this.svg = d3.select(container)
      .append('svg')
      .attr('width', width)
      .attr('height', height)
      .attr('viewBox', `0 0 ${width} ${height}`);
    
    // Add zoom behavior
    const zoom = d3.zoom<SVGSVGElement, unknown>()
      .scaleExtent([0.1, 10])
      .on('zoom', (event) => {
        this.svg.select('g').attr('transform', event.transform);
      });
    
    this.svg.call(zoom);
    
    // Create main group
    this.svg.append('g').attr('class', 'ark-main');
    
    // Add gradient definitions
    this.addGradients();
  }
  
  private setupSimulation() {
    const width = +this.svg.attr('width');
    const height = +this.svg.attr('height');
    
    this.simulation = d3.forceSimulation<ArkNode>()
      .force('link', d3.forceLink<ArkNode, ArkLink>()
        .id(d => d.id)
        .distance(d => 100 / (d.strength || 1))
        .strength(d => d.strength || 0.5)
      )
      .force('charge', d3.forceManyBody()
        .strength(d => d.id === this.centerCompanyId ? -500 : -200)
      )
      .force('center', d3.forceCenter(width / 2, height / 2))
      .force('collision', d3.forceCollide()
        .radius(d => this.getNodeRadius(d) + 5)
      );
  }
  
  private async loadData() {
    try {
      // Fetch network data
      const response = await fetch(`/api/ark/network/${this.centerCompanyId}?depth=2`);
      const data = await response.json();
      
      this.nodes = data.nodes;
      this.links = data.links;
      
      this.render();
      
      // Set up real-time updates
      this.setupWebSocket();
    } catch (error) {
      console.error('Failed to load Ark data:', error);
    }
  }
  
  private render() {
    const g = this.svg.select('g.ark-main');
    
    // Render links
    const link = g.selectAll('.ark-link')
      .data(this.links)
      .join('line')
      .attr('class', 'ark-link')
      .attr('stroke', d => this.getLinkColor(d))
      .attr('stroke-width', d => Math.sqrt(d.strength) * 2)
      .attr('stroke-opacity', 0.6);
    
    // Render nodes
    const node = g.selectAll('.ark-node')
      .data(this.nodes)
      .join('g')
      .attr('class', 'ark-node')
      .call(this.dragBehavior());
    
    // Node circles
    node.append('circle')
      .attr('r', d => this.getNodeRadius(d))
      .attr('fill', d => this.getNodeColor(d))
      .attr('stroke', '#fff')
      .attr('stroke-width', 2)
      .style('cursor', 'pointer');
    
    // Node labels
    node.append('text')
      .text(d => d.name)
      .attr('x', 0)
      .attr('y', d => this.getNodeRadius(d) + 15)
      .attr('text-anchor', 'middle')
      .style('font-size', '12px')
      .style('fill', '#333');
    
    // Add score badges for verified companies
    node.filter(d => d.verified && d.type === 'company')
      .append('g')
      .attr('transform', d => `translate(${this.getNodeRadius(d) - 10}, ${-this.getNodeRadius(d) + 10})`)
      .call(this.addScoreBadge);
    
    // Set up hover effects
    node.on('mouseenter', (event, d) => this.handleNodeHover(d, true))
        .on('mouseleave', (event, d) => this.handleNodeHover(d, false))
        .on('click', (event, d) => this.handleNodeClick(d));
    
    // Update simulation
    this.simulation.nodes(this.nodes);
    this.simulation.force<d3.ForceLink<ArkNode, ArkLink>>('link')!.links(this.links);
    
    this.simulation.on('tick', () => {
      link
        .attr('x1', d => (d.source as ArkNode).x!)
        .attr('y1', d => (d.source as ArkNode).y!)
        .attr('x2', d => (d.target as ArkNode).x!)
        .attr('y2', d => (d.target as ArkNode).y!);
      
      node.attr('transform', d => `translate(${d.x},${d.y})`);
    });
  }
  
  private getNodeRadius(node: ArkNode): number {
    if (node.id === this.centerCompanyId) return 30;
    if (node.type === 'company') return 20 + (node.score / 10);
    if (node.type === 'investor') return 25;
    return 15;
  }
  
  private getNodeColor(node: ArkNode): string {
    if (node.id === this.centerCompanyId) return 'url(#center-gradient)';
    if (node.type === 'company') return node.verified ? '#667eea' : '#cbd5e0';
    if (node.type === 'investor') return '#48bb78';
    if (node.type === 'service') return '#ed8936';
    return '#718096';
  }
  
  private setupWebSocket() {
    const ws = new WebSocket(`wss://${window.location.host}/ark/ws/${this.centerCompanyId}`);
    
    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      
      switch (message.type) {
        case 'position_update':
          this.handlePositionUpdate(message.data);
          break;
        case 'new_connection':
          this.handleNewConnection(message.data);
          break;
        case 'metric_update':
          this.handleMetricUpdate(message.data);
          break;
      }
    };
  }
}
```

## Core Infrastructure

### Database Schema

```sql
-- PostgreSQL Main Database Schema

-- Companies table
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    founded_date DATE,
    stage VARCHAR(50),
    sector VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Verifications table
CREATE TABLE verifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id),
    verification_score INTEGER CHECK (verification_score >= 0 AND verification_score <= 100),
    stage VARCHAR(50),
    verified_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    status VARCHAR(50) DEFAULT 'active'
);

-- Metrics table (time-series data)
CREATE TABLE metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id),
    metric_name VARCHAR(100),
    metric_value NUMERIC,
    metric_unit VARCHAR(50),
    source VARCHAR(100),
    confidence_score DECIMAL(3,2),
    recorded_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_company_metric_time (company_id, metric_name, recorded_at DESC)
);

-- Connections table
CREATE TABLE connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id),
    provider VARCHAR(100),
    encrypted_tokens TEXT,
    connected_at TIMESTAMP DEFAULT NOW(),
    last_sync TIMESTAMP,
    sync_frequency INTEGER DEFAULT 3600,
    status VARCHAR(50) DEFAULT 'active',
    UNIQUE(company_id, provider)
);

-- Badge impressions table
CREATE TABLE badge_impressions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id),
    badge_type VARCHAR(50),
    viewer_ip INET,
    viewer_location JSONB,
    referrer TEXT,
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_company_impressions (company_id, created_at DESC)
);
```

### Message Queue Architecture

```python
# Kafka Event Streaming Configuration
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer
import json

class EventStreamer:
    def __init__(self):
        self.producer = None
        self.consumers = {}
        
    async def initialize(self):
        self.producer = AIOKafkaProducer(
            bootstrap_servers=KAFKA_BROKERS,
            value_serializer=lambda v: json.dumps(v).encode(),
            compression_type='snappy'
        )
        await self.producer.start()
        
        # Define topics
        self.topics = {
            'metric-updates': self._handle_metric_update,
            'verification-completed': self._handle_verification_completed,
            'connection-established': self._handle_connection_established,
            'badge-generated': self._handle_badge_generated,
            'ark-position-changed': self._handle_position_changed
        }
        
        # Start consumers
        for topic, handler in self.topics.items():
            consumer = AIOKafkaConsumer(
                topic,
                bootstrap_servers=KAFKA_BROKERS,
                group_id=f'{topic}-consumer-group',
                value_deserializer=lambda v: json.loads(v.decode())
            )
            await consumer.start()
            self.consumers[topic] = consumer
            
            # Start consumer loop
            asyncio.create_task(self._consume_loop(consumer, handler))
    
    async def emit_event(self, topic: str, event: dict):
        """Emit an event to Kafka"""
        event['timestamp'] = datetime.utcnow().isoformat()
        event['event_id'] = str(uuid.uuid4())
        
        await self.producer.send(topic, value=event)
    
    async def _consume_loop(self, consumer, handler):
        """Consume events from a topic"""
        async for message in consumer:
            try:
                await handler(message.value)
            except Exception as e:
                logger.error(f"Error handling event: {e}", exc_info=True)
```

### Caching Strategy

```python
# Redis Caching Layer
import aioredis
import json
from typing import Optional, Any

class CacheManager:
    def __init__(self):
        self.redis = None
        
    async def initialize(self):
        self.redis = await aioredis.create_redis_pool(
            REDIS_URL,
            encoding='utf-8',
            minsize=5,
            maxsize=20
        )
    
    async def get_or_set(self, key: str, getter_func, ttl: int = 300) -> Any:
        """Get from cache or compute and set"""
        # Try cache first
        cached = await self.redis.get(key)
        if cached:
            return json.loads(cached)
        
        # Compute value
        value = await getter_func()
        
        # Cache it
        await self.redis.setex(key, ttl, json.dumps(value))
        
        return value
    
    async def invalidate_pattern(self, pattern: str):
        """Invalidate all keys matching pattern"""
        cursor = '0'
        while cursor != 0:
            cursor, keys = await self.redis.scan(
                cursor=cursor,
                match=pattern,
                count=100
            )
            if keys:
                await self.redis.delete(*keys)
    
    # Cache key strategies
    def company_key(self, company_id: str) -> str:
        return f"company:{company_id}"
    
    def verification_key(self, company_id: str) -> str:
        return f"verification:{company_id}"
    
    def badge_key(self, company_id: str, badge_type: str) -> str:
        return f"badge:{company_id}:{badge_type}"
    
    def ark_position_key(self, company_id: str) -> str:
        return f"ark:position:{company_id}"
```

## Security Architecture

### Authentication & Authorization

```python
# JWT-based Authentication
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
from datetime import datetime, timedelta

class AuthManager:
    def __init__(self):
        self.security = HTTPBearer()
        self.secret_key = SECRET_KEY
        self.algorithm = "HS256"
    
    def create_access_token(self, subject: dict, expires_delta: timedelta = None):
        if expires_delta:
            expire = datetime.utcnow() + expires_delta
        else:
            expire = datetime.utcnow() + timedelta(hours=24)
        
        to_encode = subject.copy()
        to_encode.update({"exp": expire, "iat": datetime.utcnow()})
        
        return jwt.encode(to_encode, self.secret_key, algorithm=self.algorithm)
    
    async def verify_token(self, credentials: HTTPAuthorizationCredentials = Security(HTTPBearer())):
        token = credentials.credentials
        
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            return payload
        except jwt.ExpiredSignatureError:
            raise HTTPException(status_code=401, detail="Token expired")
        except jwt.JWTError:
            raise HTTPException(status_code=401, detail="Invalid token")
    
    def create_api_key(self, company_id: str, scopes: List[str]):
        """Create API key for programmatic access"""
        api_key = secrets.token_urlsafe(32)
        
        # Store in database with scopes
        self.store_api_key(api_key, company_id, scopes)
        
        return f"uni_live_{api_key}"
```

### Encryption

```python
# Data Encryption
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
import base64

class EncryptionManager:
    def __init__(self):
        self.key = self._generate_key()
        self.cipher = Fernet(self.key)
    
    def _generate_key(self):
        """Generate encryption key from master secret"""
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=ENCRYPTION_SALT.encode(),
            iterations=100000,
        )
        key = base64.urlsafe_b64encode(kdf.derive(MASTER_SECRET.encode()))
        return key
    
    def encrypt_sensitive_data(self, data: str) -> str:
        """Encrypt sensitive data like OAuth tokens"""
        return self.cipher.encrypt(data.encode()).decode()
    
    def decrypt_sensitive_data(self, encrypted_data: str) -> str:
        """Decrypt sensitive data"""
        return self.cipher.decrypt(encrypted_data.encode()).decode()
```

## API Design

### RESTful API Structure

```yaml
openapi: 3.0.0
info:
  title: Unicorn Protocol API
  version: 1.0.0
  description: Trust infrastructure for the global startup ecosystem

paths:
  /v1/auth/connect:
    post:
      summary: Initialize UAuth connection
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                provider:
                  type: string
                  enum: [stripe, quickbooks, google_analytics, mixpanel]
                company_id:
                  type: string
      responses:
        200:
          description: OAuth redirect URL
          content:
            application/json:
              schema:
                type: object
                properties:
                  redirect_url:
                    type: string
                  state:
                    type: string

  /v1/verify/{company_id}:
    post:
      summary: Trigger verification for a company
      parameters:
        - name: company_id
          in: path
          required: true
          schema:
            type: string
      responses:
        200:
          description: Verification result
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/VerificationResult'

  /v1/badge/{company_id}:
    get:
      summary: Get badge for a company
      parameters:
        - name: company_id
          in: path
          required: true
          schema:
            type: string
        - name: type
          in: query
          schema:
            type: string
            enum: [minimal, standard, detailed, live]
      responses:
        200:
          description: Badge data
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BadgeData'

  /v1/ark/company/{company_id}/position:
    get:
      summary: Get company's position in the Ark
      parameters:
        - name: company_id
          in: path
          required: true
          schema:
            type: string
      responses:
        200:
          description: Ark position data
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ArkPosition'

components:
  schemas:
    VerificationResult:
      type: object
      properties:
        company_id:
          type: string
        score:
          type: integer
          minimum: 0
          maximum: 100
        stage:
          type: string
        metrics:
          type: object
        verified_at:
          type: string
          format: date-time
```

### GraphQL API

```graphql
type Query {
  # Get company verification
  company(id: ID!): Company
  
  # Search verified companies
  searchCompanies(
    filter: CompanyFilter
    sort: CompanySort
    limit: Int = 20
    offset: Int = 0
  ): CompanyConnection!
  
  # Get Ark intelligence
  arkPosition(companyId: ID!): ArkPosition!
  arkOpportunities(companyId: ID!): ArkOpportunities!
}

type Mutation {
  # Initialize UAuth connection
  connectDataSource(
    provider: DataProvider!
    companyId: ID!
  ): ConnectionResult!
  
  # Trigger verification
  verifyCompany(companyId: ID!): VerificationResult!
}

type Subscription {
  # Real-time metric updates
  metricUpdates(companyId: ID!): MetricUpdate!
  
  # Ark position changes
  arkPositionUpdates(companyId: ID!): ArkPositionUpdate!
}

type Company {
  id: ID!
  name: String!
  stage: Stage!
  score: Int!
  verification: Verification!
  metrics: Metrics!
  arkPosition: ArkPosition!
}

type Verification {
  score: Int!
  verifiedAt: DateTime!
  sources: [DataSource!]!
  badges: [Badge!]!
}

type ArkPosition {
  rank: Int!
  percentile: Float!
  centrality: Float!
  connections: Int!
  similarCompanies: [Company!]!
  trajectory: Trajectory!
  opportunities: [Opportunity!]!
}

type Trajectory {
  predictedMilestone: String!
  probability: Float!
  timelineMonths: Int!
  keyFactors: [String!]!
  risks: [String!]!
}