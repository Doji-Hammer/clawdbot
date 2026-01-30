# Gateway and API Performance Profiling for AI Applications

A comprehensive guide to profiling, bottleneck identification, and optimization techniques for TypeScript gateway services powering AI applications.

**Research Date:** 2026-01-30

---

## Table of Contents

1. [Profiling Tools Overview](#1-profiling-tools-overview)
2. [Bottleneck Identification Techniques](#2-bottleneck-identification-techniques)
3. [Loop and Task Handler Optimization](#3-loop-and-task-handler-optimization)
4. [Embedding Quantization and Inference Optimization](#4-embedding-quantization-and-inference-optimization)
5. [Memory Profiling and GC Optimization](#5-memory-profiling-and-gc-optimization)
6. [LLM Inference Bottlenecks](#6-llm-inference-bottlenecks)
7. [Best Practices Summary](#7-best-practices-summary)

---

## 1. Profiling Tools Overview

### 1.1 Built-in Node.js Profiling

Node.js provides several built-in profiling capabilities via the V8 engine:

```bash
# Start with inspector for Chrome DevTools profiling
node --inspect app.js

# Use built-in V8 profiler (samples stack at regular intervals)
node --prof app.js

# Process the profiler output
node --prof-process isolate-*.log > processed.txt
```

The [Node.js Performance Measurement APIs](https://nodejs.org/api/perf_hooks.html) (`perf_hooks`) provide high-resolution timing:

```typescript
import { performance, PerformanceObserver } from 'node:perf_hooks';

const obs = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`${entry.name}: ${entry.duration}ms`);
  }
});
obs.observe({ entryTypes: ['measure'] });

performance.mark('start');
// ... operation ...
performance.mark('end');
performance.measure('operation', 'start', 'end');
```

### 1.2 Datadog APM

[Datadog APM for Node.js](https://www.datadoghq.com/apm/node-apm/) provides code-level visibility into application health and performance.

**Installation:**
```bash
npm install --save dd-trace@latest
```

**TypeScript Setup:**
```typescript
// instrumentation.ts - must be imported first
import tracer from 'dd-trace';

tracer.init({
  service: 'gateway-service',
  env: process.env.NODE_ENV,
  profiling: true,  // Enable continuous profiling
  runtimeMetrics: true
});

export default tracer;
```

**Key Features:**
- Automatic instrumentation for HTTP, databases, message queues
- Distributed tracing across microservices
- Code-level profiling with flame graphs
- 100% trace ingestion with intelligent sampling

### 1.3 OpenTelemetry

[OpenTelemetry](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/) provides vendor-neutral observability with traces, metrics, and logs.

**Installation:**
```bash
npm install @opentelemetry/sdk-node \
  @opentelemetry/api \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/sdk-metrics \
  @opentelemetry/sdk-trace-node
```

**TypeScript Configuration:**
```typescript
// instrumentation.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://localhost:4318/v1/traces',
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

### 1.4 Tool Comparison

| Tool | Best For | Overhead | Production Safe |
|------|----------|----------|-----------------|
| Node.js `--prof` | One-off deep profiling | High | No |
| Chrome DevTools | Development debugging | Medium | No |
| Datadog APM | Full-stack APM | Low-Medium | Yes |
| OpenTelemetry | Vendor-neutral tracing | Low | Yes |
| Clinic.js | Local performance analysis | Medium | No |
| 0x | Quick flame graphs | Medium | No |

---

## 2. Bottleneck Identification Techniques

### 2.1 Flame Graph Analysis

**Reading Flame Graphs:**
- **X-axis (width)**: Percentage of CPU time consumed
- **Y-axis (height)**: Call stack depth
- **Hot paths**: Functions that are both wide (time-intensive) AND high (deeply nested)

**Using 0x for Quick Flame Graphs:**
```bash
npm install -g 0x

# Profile and generate flame graph
0x app.js

# With load testing
0x -- node app.js &
autocannon http://localhost:3000
```

### 2.2 Event Loop Monitoring

Monitor event loop lag to detect blocking operations:

```typescript
import { monitorEventLoopDelay } from 'node:perf_hooks';

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

setInterval(() => {
  const stats = {
    min: histogram.min / 1e6,      // Convert to ms
    max: histogram.max / 1e6,
    mean: histogram.mean / 1e6,
    p99: histogram.percentile(99) / 1e6,
  };

  if (stats.p99 > 100) {
    console.warn('Event loop lag detected:', stats);
  }

  histogram.reset();
}, 10000);
```

**Warning Signs:**
- Event loop lag > 100ms indicates blocking operations
- Consistent lag growth suggests memory pressure or GC issues

---

## 3. Loop and Task Handler Optimization

### 3.1 Avoiding Event Loop Blocking

```typescript
// BAD: Synchronous JSON parsing of large payloads (can block 150ms+)
const data = JSON.parse(largePayload);

// GOOD: Use streaming JSON parser
import { createReadStream } from 'node:fs';
import JSONStream from 'jsonstream';

createReadStream('large.json')
  .pipe(JSONStream.parse('*'))
  .on('data', (item) => processItem(item));
```

### 3.2 Task Partitioning

Break CPU-intensive tasks into smaller chunks:

```typescript
async function processLargeArray<T, R>(
  items: T[],
  processor: (item: T) => R,
  chunkSize = 100
): Promise<R[]> {
  const results: R[] = [];

  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    const chunkResults = chunk.map(processor);
    results.push(...chunkResults);

    // Yield to event loop between chunks
    await new Promise(resolve => setImmediate(resolve));
  }

  return results;
}
```

### 3.3 Worker Threads for CPU-Intensive Tasks

```typescript
import { Worker, isMainThread, parentPort, workerData } from 'node:worker_threads';

if (isMainThread) {
  async function runInWorker<T>(task: string, data: unknown): Promise<T> {
    return new Promise((resolve, reject) => {
      const worker = new Worker(__filename, {
        workerData: { task, data },
      });
      worker.on('message', resolve);
      worker.on('error', reject);
    });
  }

  // Use for embedding computation, encoding, etc.
  const embeddings = await runInWorker('compute-embeddings', textChunks);
}
```

### 3.4 Connection Pooling

```typescript
import { Pool } from 'pg';

// Database connection pooling (reduces query times by ~40% under load)
const pool = new Pool({
  host: 'localhost',
  database: 'app',
  max: 20,                    // Maximum connections
  idleTimeoutMillis: 30000,   // Close idle connections
  connectionTimeoutMillis: 2000,
});

// HTTP agent pooling for upstream services
import { Agent } from 'node:http';

const agent = new Agent({
  keepAlive: true,
  maxSockets: 50,
  maxFreeSockets: 10,
});
```

---

## 4. Embedding Quantization and Inference Optimization

### 4.1 Quantization Techniques Overview

| Precision | Storage Reduction | Use Case |
|-----------|-------------------|----------|
| FP32 → FP16 | 2x | General inference |
| FP32 → INT8 | 4x | Production retrieval |
| FP32 → INT4 | 8x | Large-scale RAG |
| Binary | 32x | Approximate search |

**Example Impact:** A 1M vector database with 1536-D embeddings:
- FP32: 6.1 GB
- INT4: 0.75 GB (8x reduction)

### 4.2 Caching Embeddings

```typescript
import { LRUCache } from 'lru-cache';

const embeddingCache = new LRUCache<string, Float32Array>({
  max: 10000,
  ttl: 1000 * 60 * 60,  // 1 hour
  sizeCalculation: (value) => value.byteLength,
  maxSize: 500 * 1024 * 1024,  // 500MB
});

async function getEmbedding(text: string): Promise<Float32Array> {
  const cacheKey = hashText(text);

  let embedding = embeddingCache.get(cacheKey);
  if (!embedding) {
    embedding = await computeEmbedding(text);
    embeddingCache.set(cacheKey, embedding);
  }

  return embedding;
}
```

### 4.3 Batching Requests

```typescript
import { createBatcher } from './batcher';

const embeddingBatcher = createBatcher({
  maxBatchSize: 32,
  maxWaitMs: 50,
  processBatch: async (texts: string[]) => {
    return await embeddingModel.embed(texts);
  },
});

// Individual calls are automatically batched
const embedding = await embeddingBatcher.add(text);
```

---

## 5. Memory Profiling and GC Optimization

### 5.1 GC Tuning Flags

```bash
# Set old space limit (default ~1.4GB on 64-bit)
node --max-old-space-size=4096 app.js

# Tune semi-space (young generation) size
node --max-semi-space-size=64 app.js

# Enable GC logging
node --trace-gc app.js
```

### 5.2 Memory Leak Patterns and Fixes

**Pattern 1: Unbounded Caches**
```typescript
// BAD: Unbounded cache
const cache = new Map();

// GOOD: LRU with size limits and TTL
const cache = new LRUCache({
  max: 1000,
  ttl: 1000 * 60 * 5,  // 5 minutes
  maxSize: 100 * 1024 * 1024,  // 100MB
});
```

**Pattern 2: Event Listener Leaks**
```typescript
// BAD: Adding listeners without cleanup
socket.on('data', handler);

// GOOD: Use AbortController
const controller = new AbortController();
socket.on('data', handler, { signal: controller.signal });
controller.abort();  // Removes listener
```

**Pattern 3: Closure Retention**
```typescript
// BAD: Large objects retained in closure
function createHandler(largeData) {
  return () => largeData.id;  // largeData retained
}

// GOOD: Extract only needed data
function createHandler(largeData) {
  const id = largeData.id;  // Extract before closure
  return () => id;
}
```

### 5.3 Production Memory Monitoring

```typescript
class MemoryMonitor extends EventEmitter {
  start(intervalMs = 10000) {
    setInterval(() => {
      const usage = process.memoryUsage();

      const metrics = {
        heapUsed: usage.heapUsed / 1024 / 1024,
        heapTotal: usage.heapTotal / 1024 / 1024,
        external: usage.external / 1024 / 1024,
        rss: usage.rss / 1024 / 1024,
      };

      if (metrics.heapUsed > 400) {
        this.emit('warning', 'High heap usage', metrics);
      }
    }, intervalMs);
  }
}
```

---

## 6. LLM Inference Bottlenecks

### 6.1 Key Bottlenecks in AI Gateways

LLM inference is fundamentally **memory-bound**, not compute-bound:

| Bottleneck | Impact | Mitigation |
|------------|--------|------------|
| KV Cache Growth | Memory scales with context length | Sliding window attention, KV cache compression |
| Memory Bandwidth | DRAM saturation during attention | Batching, Flash Attention |
| Cold Start | Model loading latency | Keep-warm strategies, model caching |
| Sequential Decoding | Token-by-token generation | Speculative decoding, parallel sampling |

### 6.2 Critical Metrics to Monitor

```typescript
interface LLMMetrics {
  ttft: number;           // Time to First Token (target: <500ms)
  tokensPerSecond: number; // Decode throughput
  e2eLatency: number;     // End-to-end request time
  coldStartTime: number;  // Model initialization time
  queueDepth: number;     // Pending requests
}
```

### 6.3 Gateway-Level Optimizations

**Request Batching:**
```typescript
class LLMRequestBatcher {
  private queue: Array<{ request: LLMRequest; resolve: Function; reject: Function }> = [];

  async add(request: LLMRequest): Promise<LLMResponse> {
    return new Promise((resolve, reject) => {
      this.queue.push({ request, resolve, reject });
      this.maybeProcess();
    });
  }

  private async maybeProcess() {
    if (this.processing || this.queue.length === 0) return;
    this.processing = true;

    const batch = this.queue.splice(0, 8);  // Max batch size

    try {
      const responses = await this.processBatch(batch.map(b => b.request));
      batch.forEach((item, i) => item.resolve(responses[i]));
    } catch (error) {
      batch.forEach(item => item.reject(error));
    }

    this.processing = false;
    setImmediate(() => this.maybeProcess());
  }
}
```

**Response Streaming:**
```typescript
app.post('/chat', async (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');

  for await (const chunk of streamLLMResponse(req.body)) {
    res.write(`data: ${JSON.stringify({ content: chunk })}\n\n`);
  }

  res.end();
});
```

---

## 7. Best Practices Summary

### 7.1 Quick Wins Checklist

| Area | Optimization | Expected Impact |
|------|--------------|-----------------|
| HTTP | Enable gzip compression | 50-70% response size reduction |
| Caching | Add Redis/LRU for hot paths | 10-100x latency reduction |
| Database | Connection pooling | 40% query time reduction |
| Async | Use Promise.all for parallel ops | Linear speedup |
| Memory | Implement bounded caches with TTL | Stable memory usage |
| Streaming | Stream large responses | 90% reduction in TTFB |

### 7.2 Architecture for AI Gateways

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
     ┌────────▼────────┐         ┌─────────▼────────┐
     │ Gateway Node 1  │         │ Gateway Node 2   │
     │  - Request Q    │         │  - Request Q     │
     │  - Rate Limit   │         │  - Rate Limit    │
     │  - Auth Cache   │         │  - Auth Cache    │
     └────────┬────────┘         └─────────┬────────┘
              │                             │
              └──────────────┬──────────────┘
                             │
              ┌──────────────┴──────────────┐
              │        Redis Cluster        │
              │  - Semantic Cache           │
              │  - Session State            │
              │  - Rate Limit Counters      │
              └──────────────┬──────────────┘
                             │
              ┌──────────────┴──────────────┐
              │      LLM Provider Pool      │
              │  - Primary (low latency)    │
              │  - Fallback (high capacity) │
              └─────────────────────────────┘
```

### 7.3 Monitoring Dashboard Metrics

Essential metrics to track:

- **Latency**: P50, P95, P99 response times
- **Throughput**: Requests/second, tokens/second
- **Error Rate**: 4xx, 5xx, timeout percentage
- **Resource**: CPU, memory, event loop lag
- **Business**: Cache hit rate, LLM token usage, cost per request

---

## Sources

- [Node.js Performance Measurement APIs](https://nodejs.org/api/perf_hooks.html)
- [Datadog Node.js APM](https://www.datadoghq.com/apm/node-apm/)
- [OpenTelemetry Node.js](https://opentelemetry.io/docs/languages/js/getting-started/nodejs/)
- [Brendan Gregg - Flame Graphs](https://www.brendangregg.com/flamegraphs.html)
- [Don't Block the Event Loop](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)
- [Node.js Memory Profiling](https://medium.com/@ahmedrao609/node-js-memory-profiling-tools-techniques-and-real-world-wins-6f04b5c39907)
- [V8 Memory Management and GC Tuning](https://blog.platformatic.dev/optimizing-nodejs-performance-v8-memory-management-and-gc-tuning)
- [LLM Inference Optimization - Clarifai](https://www.clarifai.com/blog/llm-inference-optimization/)
- [Top 7 LLM Performance Bottlenecks](https://www.getmaxim.ai/articles/top-7-performance-bottlenecks-in-llm-applications-and-how-to-overcome-them/)
- [Embedding Quantization Techniques](https://www.emergentmind.com/topics/embedding-quantization)
