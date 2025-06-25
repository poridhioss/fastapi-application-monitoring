# Database Performance Monitoring - PromQL Query Guide

This document provides a comprehensive explanation of the 7 database-related PromQL queries used in the FastAPI monitoring lab. These queries cover database operations, connection pool management, and query performance analysis.



## 1. SELECT Query Rate

### PromQL Query
```promql
rate(db_queries_total{operation="select"}[5m])
```

![alt text](./images/image.png)

Go to `Explain` tab to see details. 

### Query Breakdown
- **`db_queries_total`**: Counter metric tracking total database queries
- **`{operation="select"}`**: Label selector filtering only SELECT operations
- **`[5m]`**: Time range selector for the last 5 minutes
- **`rate()`**: Function calculating the per-second average rate of increase

### Mathematical Formula
```
Rate = (Current_Value - Previous_Value) / Time_Interval_Seconds
```

### Expected Output in Prometheus
```
db_queries_total{operation="select"} 1.4
```

- **Value**: 1.4 queries per second
- **Interpretation**: Database is processing 1.4 SELECT queries per second on average

### Monitoring Significance
- **Performance Indicator**: High SELECT rates indicate heavy read workloads
- **Capacity Planning**: Helps determine if read replicas are needed
- **Baseline Establishment**: Normal rate varies by application (1-100+ QPS)




## 2. INSERT Query Rate

### PromQL Query
```promql
rate(db_queries_total{operation="insert"}[5m])
```

![alt text](./images/image-1.png)

### Query Breakdown
- **`db_queries_total`**: Same counter metric as SELECT queries
- **`{operation="insert"}`**: Label selector for INSERT operations only
- **`[5m]`**: 5-minute time window for rate calculation
- **`rate()`**: Converts counter increases to per-second rates

### Mathematical Formula
```
INSERT_Rate = (INSERT_Count_Now - INSERT_Count_5min_ago) / 300_seconds
```

### Expected Output in Prometheus
```
db_queries_total{operation="insert"} 0.8
```
- **Value**: 0.8 inserts per second
- **Interpretation**: Database is handling 0.8 new records per second

### Monitoring Significance
- **Write Performance**: Critical for understanding database write load
- **Transaction Volume**: Direct correlation with business activity
- **Resource Planning**: High INSERT rates require more write-optimized storage

### Performance Implications
- **Normal Patterns**: Typically lower than SELECT rates (1:3 to 1:10 ratio)
- **Batch Operations**: Sudden spikes may indicate bulk data imports
- **Lock Contention**: Very high rates can cause table locking issues



## 3. SELECT 95th Percentile Latency

### PromQL Query
```promql
histogram_quantile(0.95, sum(rate(db_query_duration_seconds_bucket{operation="select"}[5m])) by (le))
```

![alt text](./images/image-2.png)

### Query Breakdown
- **`db_query_duration_seconds_bucket`**: Histogram metric with latency buckets
- **`{operation="select"}`**: Filter for SELECT operation latencies
- **`rate(...[5m])`**: Calculate rate of samples in each bucket over 5 minutes
- **`sum(...) by (le)`**: Aggregate across all instances, grouped by bucket upper bounds
- **`histogram_quantile(0.95, ...)`**: Calculate the 95th percentile from histogram

### Mathematical Formula
```
P95 = Bucket_Lower_Bound + (Bucket_Width × (Target_Rank - Cumulative_Count_Below) / Bucket_Count)

Where:
- Target_Rank = 0.95 × Total_Sample_Count
- Bucket_Width = Upper_Bound - Lower_Bound
```

### Expected Output in Prometheus
```
{} 0.045
```
- **Value**: 0.045 seconds (45 milliseconds)
- **Interpretation**: 95% of SELECT queries complete within 45ms

### Performance Benchmarks
- **Excellent**: < 50ms (0.05s)
- **Good**: 50-200ms (0.05-0.2s)
- **Acceptable**: 200ms-1s (0.2-1.0s)
- **Concerning**: > 1s (1.0s+)

### Optimization Indicators
- **Values > 100ms**: Consider index optimization
- **Values > 500ms**: Investigate query execution plans
- **Values > 2s**: Critical performance issues requiring immediate attention



## 4. INSERT 99th Percentile Latency

### PromQL Query
```promql
histogram_quantile(0.99, sum(rate(db_query_duration_seconds_bucket{operation="insert"}[5m])) by (le))
```

![alt text](./images/image-3.png)

### Query Breakdown
- **`histogram_quantile(0.99, ...)`**: Calculates 99th percentile instead of 95th
- **Same base components**: Uses INSERT operation filter instead of SELECT
- **Higher Precision**: 99th percentile captures more extreme latency cases

### Mathematical Formula
```
P99 = Interpolated_Value_Where_99%_Of_Samples_Fall_Below
```

### Expected Output in Prometheus
```
{} 0.125
```
- **Value**: 0.125 seconds (125 milliseconds)  
- **Interpretation**: 99% of INSERT operations complete within 125ms

### Why 99th Percentile for INSERTs?
- **Write Criticality**: INSERT failures have higher business impact
- **Transaction Integrity**: Slow INSERTs can cause connection pool exhaustion
- **User Experience**: Data creation delays are immediately visible to users

### Performance Analysis
- **Normal Range**: 2-5x higher than SELECT P95 latency
- **Concerning Signs**: > 1 second indicates serious write performance issues
- **Optimization Priority**: Focus on transaction log performance and index efficiency



## 5. Checked-out Connections

### PromQL Query
```promql
db_pool_checked_out_connections
```

![alt text](./images/image-4.png)

### Query Breakdown
- **`db_pool_checked_out_connections`**: Gauge metric (instantaneous value)
- **No time range**: Current snapshot of active connections
- **Direct measurement**: No mathematical transformation needed

### Expected Output in Prometheus
```
db_pool_checked_out_connections 8
```
- **Value**: 8 connections currently in use
- **Interpretation**: 8 database connections are actively serving requests

### Connection Pool Mathematics
```
Pool_Utilization = Checked_Out_Connections / Total_Pool_Size × 100%

Example: 8 / 20 = 40% utilization
```

### Monitoring Significance
- **Resource Utilization**: Shows how heavily the connection pool is used
- **Performance Indicator**: High utilization may indicate connection leaks
- **Capacity Planning**: Helps determine optimal pool size



## 6. Idle Connections

### PromQL Query
```promql
db_pool_idle_connections
```

![alt text](./images/image-5.png)

### Query Breakdown
- **`db_pool_idle_connections`**: Gauge showing available connections
- **Instantaneous Value**: Current count of unused connections in pool
- **Ready State**: These connections are established and ready for immediate use

### Expected Output in Prometheus
```
db_pool_idle_connections 12
```
- **Value**: 12 connections available for use
- **Interpretation**: 12 established database connections are idle and ready

### Pool Health Mathematics
```
Total_Connections = Checked_Out + Idle + Creating + Invalid

Pool_Health = (Idle + Checked_Out) / Total_Pool_Size × 100%
```

### Monitoring Applications
- **Availability Assurance**: Ensures connections are available for new requests
- **Pool Efficiency**: Balance between resource usage and response time
- **Capacity Buffer**: Idle connections provide headroom for traffic spikes

### Optimization Insights
- **Too Many Idle**: Reduce `min_pool_size` to save database resources
- **Too Few Idle**: Increase `max_pool_size` to improve response times
- **Zero Idle**: Immediate capacity constraint - requests will queue



## 7. Connection Waiters

### PromQL Query
```promql
db_pool_waiters
```

![alt text](./images/image-6.png)

### Query Breakdown
- **`db_pool_waiters`**: Gauge showing queued connection requests
- **Queue Depth**: Number of application threads waiting for database connections
- **Performance Bottleneck Indicator**: Non-zero values indicate resource constraint

### Expected Output in Prometheus
```
db_pool_waiters 0
```
- **Value**: 0 requests waiting
- **Interpretation**: No connection pool saturation - optimal state

### Critical Performance Indicator
```
Wait_Time_Estimate = Average_Connection_Usage_Time × Queue_Position

Example: 50ms × 3_waiters = 150ms additional latency
```

### Alert Thresholds
- **Warning**: > 0 waiters (pool under pressure)
- **Critical**: > 5 waiters (significant user impact)
- **Emergency**: > 20 waiters (potential service degradation)

### Resolution Strategies
1. **Immediate**: Scale up connection pool size
2. **Short-term**: Optimize slow queries to release connections faster
3. **Long-term**: Implement connection pooling at application layer
4. **Architecture**: Consider read replicas to distribute connection load



## Database Monitoring Best Practices

### Query Correlation Analysis
```promql
# Connection efficiency ratio
db_pool_idle_connections / (db_pool_checked_out_connections + db_pool_idle_connections)

# Query rate vs latency correlation
rate(db_queries_total[5m]) vs histogram_quantile(0.95, rate(db_query_duration_seconds_bucket[5m]))
```

### Performance Baseline Establishment
- **Peak Hours**: Monitor query rates and latencies during high traffic
- **Off-Peak Comparison**: Establish baseline performance expectations
- **Growth Trends**: Track monthly increases in query volume and latency

### Alerting Recommendations
```yaml
# Example alert conditions
SELECT_Latency_High: P95 > 200ms for 5 minutes
INSERT_Latency_Critical: P99 > 1s for 2 minutes
Connection_Pool_Exhausted: db_pool_waiters > 0 for 1 minute
Query_Rate_Anomaly: rate(db_queries_total[5m]) > 2x baseline
```

### Capacity Planning Metrics
- **Peak Query Rate**: Maximum sustainable QPS before latency degradation
- **Connection Pool Optimization**: Right-sizing based on actual usage patterns
- **Storage Performance**: Correlation between query patterns and disk I/O
- **Memory Usage**: Database cache hit ratios vs query performance

This comprehensive analysis of database monitoring queries provides the foundation for maintaining high-performance database operations in production FastAPI applications.