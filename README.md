# معماری سیستم DevOps با Redis — طراحی کامل

این سند یک طراحی عملیاتی و قابل پیاده‌سازی برای یک سیستم DevOps است که Redis را به عنوان یکی از قطعات کلیدی زیرساخت استفاده می‌کند. سند شامل معماری سطح‌بالا، مولفه‌ها، جریان‌های کلیدی، نمونه‌کانفیگ‌ها، راهکارهای پایایی و مانیتورینگ و راهنمایی برای استقرار در محیط‌های مختلف (Docker Compose / Kubernetes / Bare-metal) می‌باشد.

---

## 1. اهداف طراحی
- افزایش عملکرد (latency و throughput) با استفاده از caching و ساختارهای داده در Redis.
- افزایش تحمل خطا و دسترس‌پذیری با استفاده از Redis Cluster و Sentinel.
- ساده‌سازی هماهنگی بین سرویس‌ها (locks, leader election, distributed coordination).
- صف‌بندی و پردازش asynchronous با Redis Streams / Lists.
- فراهم کردن قابل‌مشاهده‌پذیری (observability) و مانیتورینگ برای Redis و وابستگی‌ها.

---

## 2. سناریوها و الگوهای استفاده (Use Cases)
1. **Cache layer** برای نتایج read-heavy APIها.
2. **Session store** برای وب‌سرویس‌ها (stateless app servers).
3. **Rate limiting** در لایه API Gateway با استفاده از atomic counters.
4. **Job queue / Stream processing** برای پردازش پس‌زمینه (emails, thumbnailing).
5. **Pub/Sub** برای event propagation سبک و coordination.
6. **Distributed locks & leader election** برای هماهنگی بین replica workers.

---

## 3. اجزای اصلی معماری
- **Clients / Microservices**: سرویس‌های اپلیکیشن (Node.js, Go, Python...) که از Redis برای cache, sessions, queues استفاده می‌کنند.
- **API Gateway**: (مثلاً Kong, Traefik یا AWS ALB) — پیاده‌سازی rate limiting با Redis.
- **Redis Layer**:
  - Redis Cluster برای sharding و scale افقی.
  - Redis Sentinel یا مدیریت شده (Elasticache, Memorystore) برای HA (در صورت استفاده از standalone nodes).
  - Replicaهای read-only برای read scaling.
- **Processing Workers**: مصرف‌کننده‌های queue/stream برای پردازش پس‌زمینه.
- **Persistent DB**: (Postgres/MySQL/Cassandra) که Redis فقط cache یا queue است، منبع حقیقت دیتابیس رابطه‌ای.
- **Observability Stack**: Prometheus + Grafana برای metrics، Loki/ELK برای logs، و Alertmanager برای alerting.
- **CI/CD**: GitOps یا pipelines (GitLab CI / Jenkins / ArgoCD) برای استقرار و کانفیگ.

---

## 4. دیاگرام معماری (سطح‌بالا)

```
[Clients] --> [API Gateway] --> [App Servers (stateless)]
                                    |       |
                                    |       +--> [Redis (cache/session)]
                                    |       |
                                    |       +--> [Redis Streams / Queues] --> [Workers]
                                    |
                                    +--> [Persistent DB]

Monitoring: Prometheus/Grafana <--- Redis Exporter
Logging: Fluentd/Logstash --> ELK/Loki

Redis Cluster: Master(s) + Replica(s) + Sentinel/Managed
```

---

## 5. جریان‌های کلیدی و نمونه‌ها

### 5.1. Cache (Read-through)
- Request arrives → App checks Redis key `cache:resource:<id>`
  - Hit: return cached result
  - Miss: read from DB, set key with TTL (مثلاً `EX 60` یا الگوریتم LRU)

**نکات عملی:**
- برای invalidation از pub/sub یا cache-busting با versioned keys (`resource:123:v42`).
- TTLهای منطقی و محاسباتی (adaptive TTL) استفاده کنید تا thundering herd رخ ندهد.

### 5.2. Rate Limiting (Sliding window یا fixed window)
- استفاده از atomic INCR with EX:
  - `INCR user:123:count` و `EXPIRE user:123:count 60` برای fixed window
- برای sliding-window از Sorted Sets یا Lua scripts استفاده کنید.

### 5.3. Sessions
- Store session data در `session:<session-id>` با TTL مطابق lifetime لاگین.
- اگر داده session حساس است، از encryption و signed tokens استفاده کنید.

### 5.4. Queues / Streams
- Producer: `XADD orders-stream * orderId 123 status pending`
- Consumer Group: `XGROUP CREATE orders-stream order-workers $` و سپس `XREADGROUP` برای خواندن.

**نکات عملی:**
- استفاده از consumer groups برای پردازش parallel و re-delivery handling.
- Dead-letter stream برای پیام‌های خطادار.

### 5.5. Distributed Locking
- از Redlock یا ساده‌تر: `SET resource-lock unique-token NX PX 30000` سپس RELEASE با Lua script که فقط owner را حذف کند.

---

## 6. معماری استقرار (انتخاب پلتفرم)

### 6.1 Docker Compose — برای dev و staging کوچک
- Redis cluster ممکن است در محیط dev با single node یا master-replica اجرا شود.
- نمونه `docker-compose.yml` در پیوست.

### 6.2 Kubernetes — production
- از یک StatefulSet برای Redis cluster یا از Chartهای رسمی (bitnami/redis) استفاده کنید.
- از `PersistentVolumeClaims` برای persistence (AOF/RDB) استفاده کنید.
- Sidecars: redis-exporter برای metrics، و init containers برای bootstrap.
- استفاده از PodDisruptionBudget و PodAntiAffinity برای high availability.

### 6.3 Managed Redis
- اگر از AWS/GCP/Azure استفاده می‌کنید، سرویس‌های managed (ElastiCache, Memorystore, Azure Cache) کاهش operational burden را فراهم می‌کنند.

---

## 7. پیکربندی Redis — توصیه‌ها
- **Persistence**:
  - AOF with `appendfsync everysec` برای durability بالاتر.
  - RDB snapshots برای backup سریع.
  - اگر Redis صرفاً cache است، می‌توان persistence را غیرفعال کرد ولی آگاه باشید که on-restart داده از بین می‌رود.
- **Memory policies**:
  - `maxmemory` و `maxmemory-policy` (lru, allkeys-lru یا volatile-lru)
- **Security**:
  - استفاده از AUTH (`requirepass` یا ACLs in Redis 6+)
  - شبکه خصوصی، TLS encryption between clients and servers (Redis 6+ supports TLS)
  - IP allowlists و حداقل‌سازی attack surface
- **Clustering**:
  - حداقل 3 master برای cluster production توصیه می‌شود (برای quorum)
  - هر master حداقل یک replica
- **Replication & HA**:
  - Redis Sentinel یا managed provider برای failover

---

## 8. مانیتورینگ و Alerting
- Metrics to collect:
  - `used_memory`, `evicted_keys`, `instantaneous_ops_per_sec`, `connected_clients`, `keyspace_hits/misses`, replication lag (`master_link_status`, `master_last_io_seconds_ago`)
- Use `redis_exporter` -> Prometheus -> Grafana
- Alerts examples:
  - High memory usage (> 80%)
  - Rapid increase in evicted keys
  - Replication lag > 5s
  - Redis down / node unreachable

---

## 9. Backup و Disaster Recovery
- Periodic RDB snapshot to object storage (S3/GCS) + AOF archival
- Test restore monthly in staging
- Automate backup verification (restore + smoke tests)

---

## 10. Runbook نمونه برای حالت Failover
1. Detection: Alert from Prometheus/Alertmanager
2. If node is master and unresponsive: verify network, metrics, and logs
3. If automatic failover via Sentinel/managed: monitor that failover completes and verify application traffic
4. If manual intervention: promote replica to master, update configs or DNS if necessary
5. Rebuild failed node from snapshot or replace with new instance and resync replication
6. Postmortem: collect logs, timeline, and apply mitigation to prevent recurrence

---

## 11. Security و Compliance
- Encrypt in transit (TLS) and at rest (disk encryption for persistence files)
- Use Redis ACLs to limit commands (disable dangerous commands like `FLUSHALL` for app users)
- Audit access and integrate with centralized auth where possible

---

## 12. Testing و Validation
- Load testing (wrk, k6) برای سنجش hit/miss cache
- Chaos testing: شبیه‌سازی node failure، network partition
- Smoke tests پس از هر deploy برای اعتبارسنجی connectivity به Redis

---

## 13. نمونه کانفیگ‌ها (خلاصه)

### نمونه docker-compose برای single node (dev)

```yaml
version: '3.8'
services:
  redis:
    image: redis:7
    command: redis-server --appendonly yes
    ports:
      - "6379:6379"
    volumes:
      - ./data:/data
```

### snippet Redis config (redis.conf)

```
requirepass myStrongPass
maxmemory 4gb
maxmemory-policy allkeys-lru
appendonly yes
appendfsync everysec
```

(نسخه‌های production باید TLS, ACL و monitoring اضافه شوند.)

---

## 14. هزینه و تصمیم‌گیری (Managed vs Self-hosted)
- Managed: قیمت بالاتر ولی operational burden کمتر، automated backups & upgrades
- Self-hosted: کنترل کامل، هزینه زیرساختی، اما نیاز به تیم ops قوی

---

## 15. چک‌لیست استقرار Production
- [ ] Architecture review
- [ ] Sizing (RAM per shard, replicas)
- [ ] Backup enabled and tested
- [ ] Monitoring & Alerts configured
- [ ] Security: ACL, TLS, network controls
- [ ] Runbook and postmortem template
- [ ] Load/chaos testing passed

---

## 16. پیوست — منابع اجرایی و Snippets
- نمونه manifest برای Kubernetes (StatefulSet + Service + ConfigMap)
- Alert rules برای Prometheus
- Lua script نمونه برای rate limiting

(در صورت نیاز، می‌توانم هر یک از بخش‌های "نمونه‌ها" را به صورت فایل‌های جداگانه، manifests یا اسکریپت‌های آماده پیاده‌سازی تولید کنم.)

---

### پایان سند

اگر می‌خواهی بخش خاصی را کامل‌تر کنم (مثلاً manifests برای Kubernetes، docker-compose کامل، Terraform script، یا runbook دقیق برای failover) بگو تا همانجا اضافه کنم.

