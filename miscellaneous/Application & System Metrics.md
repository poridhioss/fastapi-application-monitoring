# Application & System Monitoring - PromQL Query Guide


This document provides detailed explanations of the 8 non-database PromQL queries used in the FastAPI monitoring lab. These queries cover HTTP performance, system resources, and application-level metrics.


## 1. Global Request Rate

### PromQL Query
```promql
rate(http_requests_total[5m])
```

![alt text](./images/image-7.png)

### Query Breakdown
- **`http_requests_total`**: Counter metric tracking all HTTP requests
- **No label filters**: Aggregates requests across all endpoints and methods
- **`[5m]`**: Time range selector for 5-minute window
- **`rate()`**: Calculates per-second average rate of increase

### Mathematical Formula
```
Global_Rate = Σ(HTTP_Requests_Current - HTTP_Requests_5min_ago) / 300_seconds
```

### Expected Output in Prometheus
```
http_requests_total{endpoint="/health",method="GET"} 0.8
http_requests_total{endpoint="/data",method="GET"} 1.2  
http_requests_total{endpoint="/data",method="POST"} 0.4
```
- **Total Rate**: 2.4 requests per second across all endpoints
- **Interpretation**: Application is handling 2.4 HTTP requests per second

### Business Metrics Correlation
- **User Activity**: Direct indicator of application usage
- **Traffic Patterns**: Shows daily/weekly usage cycles
- **Capacity Planning**: Foundation for scaling decisions


## 2. POST /data Request Rate

### PromQL Query
```promql
rate(http_requests_total{endpoint="/data",method="POST"}[5m])
```

![alt text](./images/image-8.png)

### Query Breakdown
- **`http_requests_total`**: Same counter metric as global rate
- **`{endpoint="/data",method="POST"}`**: Label selectors for specific endpoint and method
- **Focused Monitoring**: Isolates data creation operations
- **Business Logic Tracking**: Monitors core application functionality

### Mathematical Formula
```
POST_Rate = (POST_Count_Now - POST_Count_5min_ago) / 300_seconds
```

### Expected Output in Prometheus
```
http_requests_total{endpoint="/data",method="POST"} 0.6
```
- **Value**: 0.6 POST requests per second
- **Interpretation**: Users are creating data at 0.6 operations per second

### Business Intelligence Applications
- **User Engagement**: Measures active content creation
- **Feature Usage**: Tracks adoption of data entry features
- **Conversion Metrics**: Can correlate with business KPIs



## 3. GET /data 95th Percentile Latency

### PromQL Query
```promql
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{endpoint="/data",method="GET"}[5m])) by (le))
```

![alt text](./images/image-9.png)

### Query Breakdown
- **`http_request_duration_seconds_bucket`**: Histogram metric with latency buckets
- **`{endpoint="/data",method="GET"}`**: Filters for specific endpoint GET requests
- **`rate(...[5m])`**: Rate of samples in each histogram bucket over 5 minutes
- **`sum(...) by (le)`**: Aggregates across instances, grouped by bucket upper bounds
- **`histogram_quantile(0.95, ...)`**: Calculates 95th percentile latency

### Mathematical Formula
```
P95_Latency = Lower_Bound + ((0.95 × Total_Count - Count_Below) / Count_In_Bucket) × Bucket_Width

Where histogram buckets might be: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, +Inf]
```

### Expected Output in Prometheus
```
{} 0.185
```
- **Value**: 0.185 seconds (185 milliseconds)
- **Interpretation**: 95% of GET /data requests complete within 185ms

### User Experience Impact
- **Excellent**: < 100ms (users perceive as instant)
- **Good**: 100-300ms (slight delay, acceptable)
- **Acceptable**: 300ms-1s (noticeable but usable)
- **Poor**: > 1s (significant user frustration)



## 4. CPU Usage Rate

### PromQL Query
```promql
rate(process_cpu_seconds_total[5m]) * 100
```

![alt text](./images/image-10.png)

### Query Breakdown
- **`process_cpu_seconds_total`**: Counter tracking total CPU time consumed
- **`rate(...[5m])`**: CPU time consumption rate over 5 minutes
- **`* 100`**: Converts decimal to percentage (0.45 → 45%)
- **Process-Specific**: Only tracks the FastAPI application process

### Mathematical Formula
```
CPU_Usage% = (CPU_Seconds_Now - CPU_Seconds_5min_ago) / 300_seconds × 100

Example: (15.8 - 12.3) / 300 × 100 = 1.17% CPU usage
```

### Expected Output in Prometheus
```
process_cpu_seconds_total 45.2
```
- **Value**: 45.2% CPU utilization
- **Interpretation**: FastAPI process is using 45.2% of available CPU

### Performance Interpretation
- **Idle**: 0-20% (application ready for more load)
- **Normal**: 20-60% (healthy utilization under normal traffic)
- **High**: 60-80% (approaching capacity limits)
- **Critical**: 80%+ (performance degradation likely)


## 5. Resident Memory Usage

### PromQL Query
```promql
process_resident_memory_bytes
```

![alt text](./images/image-11.png)

### Query Breakdown
- **`process_resident_memory_bytes`**: Gauge metric showing physical RAM usage
- **Resident Set Size (RSS)**: Physical memory currently held in RAM
- **No time range**: Instantaneous memory usage snapshot
- **Process-Specific**: Only the FastAPI application's memory footprint

### Expected Output in Prometheus
```
process_resident_memory_bytes 134217728
```
- **Value**: 134,217,728 bytes (128 MB)
- **Interpretation**: FastAPI process is using 128 MB of physical RAM

### Memory Usage Analysis
```
Memory_MB = Bytes / (1024 × 1024)
Memory_GB = Bytes / (1024 × 1024 × 1024)

Example: 134,217,728 / (1024²) = 128 MB
```



## 6. Virtual Memory Usage

### PromQL Query
```promql
process_virtual_memory_bytes
```

![alt text](./images/image-12.png)

### Query Breakdown
- **`process_virtual_memory_bytes`**: Gauge showing total virtual address space
- **Virtual Memory**: All memory allocated to process (RAM + swap + unused)
- **Address Space**: Includes memory-mapped files and shared libraries
- **Always Larger**: Virtual memory ≥ resident memory

### Expected Output in Prometheus
```
process_virtual_memory_bytes 1073741824
```
- **Value**: 1,073,741,824 bytes (1 GB)
- **Interpretation**: Process has allocated 1 GB of virtual address space

### Virtual vs Resident Memory
```
Virtual_Memory = Physical_RAM + Swap_Space + Memory_Mapped_Files + Unused_Allocations
Resident_Memory = Only_Physical_RAM_Currently_Used

Typical_Ratio = Virtual_Memory / Resident_Memory ≈ 3:1 to 10:1
```

### System Health Indicators
- **Normal Growth**: Virtual memory grows with application features
- **Excessive Growth**: May indicate memory fragmentation or leaks
- **Swap Usage**: Virtual >> Physical may cause performance issues
- **32-bit Limits**: Virtual memory capped at ~3-4 GB on 32-bit systems



## 7. GC Collections Total

### PromQL Query
```promql
python_gc_collections_total
```

![alt text](./images/image-13.png)

### Query Breakdown
- **`python_gc_collections_total`**: Counter tracking Python garbage collection events
- **Cumulative Count**: Total GC cycles since process start
- **Python-Specific**: Monitors Python's automatic memory management
- **Performance Indicator**: Frequent GC can impact response times

### Expected Output in Prometheus
```
python_gc_collections_total{generation="0"} 1247
python_gc_collections_total{generation="1"} 113  
python_gc_collections_total{generation="2"} 8
```
- **Generation 0**: 1,247 young object collections (frequent, fast)
- **Generation 1**: 113 middle-aged object collections (less frequent)
- **Generation 2**: 8 old object collections (rare, expensive)

### Python GC Mathematics
```
Collection_Rate = Collections_Per_Hour
Generation_0_Rate = Δ(gen0_collections) / Time_Hours (typically 100-1000/hour)
Generation_2_Rate = Δ(gen2_collections) / Time_Hours (typically 1-10/hour)
```

### Performance Impact Analysis
- **Gen 0 Collections**: Minimal impact (~1-5ms each)
- **Gen 1 Collections**: Moderate impact (~5-20ms each)
- **Gen 2 Collections**: Significant impact (~20-100ms each)
- **Memory Pressure**: High rates indicate memory churn

### Optimization Strategies
- **Object Pooling**: Reduce temporary object creation
- **Memory Efficiency**: Use generators and iterators
- **GC Tuning**: Adjust collection thresholds if needed
- **Profiling**: Identify code creating excessive temporary objects


## 8. GC Objects Collected Total

### PromQL Query
```promql
python_gc_objects_collected_total
```

![alt text](./images/image-14.png)

### Query Breakdown
- **`python_gc_objects_collected_total`**: Counter of objects reclaimed by GC
- **Memory Reclamation**: Shows how much memory is being freed
- **Efficiency Metric**: Objects collected per GC cycle
- **Memory Health**: Indicates application memory usage patterns

### Expected Output in Prometheus
```
python_gc_objects_collected_total{generation="0"} 89234
python_gc_objects_collected_total{generation="1"} 12876
python_gc_objects_collected_total{generation="2"} 1543
```
- **Total Objects**: 103,653 objects reclaimed by garbage collection
- **Generation Distribution**: Most objects collected in gen 0 (short-lived)

### Memory Efficiency Calculations
```
Collection_Efficiency = Objects_Collected / GC_Collections
Average_Objects_Per_Collection = Total_Objects_Collected / Total_Collections

Example: 89,234 objects / 1,247 collections = 71.6 objects per gen-0 collection
```

### Application Health Indicators
- **Healthy Pattern**: Steady collection rates matching allocation patterns
- **Memory Leak Signs**: Objects allocated faster than collected
- **Optimization Success**: Decreasing collection rates with same functionality
- **Performance Impact**: Very high collection rates may cause latency spikes

### Correlation with Other Metrics
```promql
# Memory allocation efficiency
rate(python_gc_objects_collected_total[5m]) / rate(python_gc_collections_total[5m])

# GC impact on response times
rate(python_gc_collections_total{generation="2"}[5m]) vs histogram_quantile(0.95, http_request_duration_seconds_bucket)
```

---

This comprehensive analysis provides the foundation for monitoring application performance, system health, and optimization opportunities in production FastAPI deployments.