# Context Window Management Strategies for Multi-Channel AI Assistants

**Research Date:** 2026-01-30

A comprehensive guide to managing context windows effectively in LLM-powered chatbot applications, with specific recommendations for multi-channel messaging systems.

---

## Table of Contents

1. [Truncation Strategies](#1-truncation-strategies)
2. [Summarization Techniques](#2-summarization-techniques)
3. [Memory Systems](#3-memory-systems)
4. [Optimal Context Lengths](#4-optimal-context-lengths)
5. [Chunking and Pagination](#5-chunking-and-pagination)
6. [Context Compression Techniques](#6-context-compression-techniques)
7. [Prompt Caching](#7-prompt-caching)
8. [Multi-Channel Specific Recommendations](#8-multi-channel-specific-recommendations)
9. [Code Examples](#9-code-examples)
10. [Benchmarks and Performance Data](#10-benchmarks-and-performance-data)

---

## 1. Truncation Strategies

### Sliding Window (Token-Based)

The simplest approach: keep only the N most recent tokens/messages, discarding older ones.

```typescript
function slidingWindow(messages: Message[], maxTokens: number): Message[] {
  let tokenCount = 0;
  const result: Message[] = [];

  // Work backwards from most recent
  for (let i = messages.length - 1; i >= 0; i--) {
    const msgTokens = countTokens(messages[i].content);
    if (tokenCount + msgTokens > maxTokens) break;
    result.unshift(messages[i]);
    tokenCount += msgTokens;
  }

  return result;
}
```

**Trade-offs:**
- **Pros:** Simple, predictable, fast
- **Cons:** Loses early context that may be crucial (e.g., initial instructions, user preferences)

### Token Budget Allocation

A more sophisticated approach allocating token budgets by importance:

```typescript
interface TokenBudget {
  systemPrompt: number;      // 15-20% - never truncated
  summary: number;           // 10-15% - compressed history
  toolResults: number;       // 20-25% - recent tool outputs
  recentMessages: number;    // 40-50% - conversation history
}

const defaultBudget: TokenBudget = {
  systemPrompt: 2000,
  summary: 1500,
  toolResults: 2500,
  recentMessages: 4000,
};
```

### Importance-Based Selection

Anthropic's recommendation for agentic applications: weight messages by importance and recency.

```typescript
interface WeightedMessage {
  message: Message;
  importance: number;  // 0-1, higher = more important
  timestamp: number;
}

function selectByImportance(
  messages: WeightedMessage[],
  maxTokens: number
): Message[] {
  // Always keep system prompt and current user message
  const mustKeep = messages.filter(m =>
    m.message.role === 'system' ||
    m.timestamp === Math.max(...messages.map(x => x.timestamp))
  );

  // Sort remaining by combined score (importance + recency)
  const candidates = messages
    .filter(m => !mustKeep.includes(m))
    .sort((a, b) => {
      const recencyA = a.timestamp / Date.now();
      const recencyB = b.timestamp / Date.now();
      return (b.importance + recencyB) - (a.importance + recencyA);
    });

  // Add candidates by importance until budget exhausted
  const result = [...mustKeep];
  let tokenCount = result.reduce((acc, m) => acc + countTokens(m.message.content), 0);

  for (const candidate of candidates) {
    const tokens = countTokens(candidate.message.content);
    if (tokenCount + tokens > maxTokens) continue;
    result.push(candidate);
    tokenCount += tokens;
  }

  return result.map(w => w.message).sort((a, b) => a.timestamp - b.timestamp);
}
```

### Observation Masking

Target environment observations specifically while preserving action and reasoning history in full. This approach is particularly effective for agentic applications.

---

## 2. Summarization Techniques

### Recursive Summarization

Messages undergo recursive summarization—they're summarized along with existing summaries from previously summarized messages. As conversations grow longer, older messages have progressively less influence on the summary.

```typescript
interface ConversationState {
  summary: string;
  recentMessages: Message[];
  summaryTokenLimit: number;
  recentWindowSize: number;
}

async function recursiveSummarize(
  state: ConversationState,
  newMessage: Message,
  summarizer: (messages: Message[], existingSummary: string) => Promise<string>
): Promise<ConversationState> {
  const updatedRecent = [...state.recentMessages, newMessage];

  if (updatedRecent.length <= state.recentWindowSize) {
    return { ...state, recentMessages: updatedRecent };
  }

  // Messages to be summarized
  const toSummarize = updatedRecent.slice(0, -state.recentWindowSize);
  const newSummary = await summarizer(toSummarize, state.summary);

  return {
    ...state,
    summary: newSummary,
    recentMessages: updatedRecent.slice(-state.recentWindowSize)
  };
}
```

### ConversationSummaryBufferMemory Pattern

A hybrid approach that summarizes the earliest interactions while maintaining the most recent tokens in full.

```typescript
class SummaryBufferMemory {
  private summary: string = '';
  private buffer: Message[] = [];
  private maxBufferTokens: number;

  constructor(maxBufferTokens: number = 2000) {
    this.maxBufferTokens = maxBufferTokens;
  }

  async addMessage(message: Message, summarizer: LLMSummarizer): Promise<void> {
    this.buffer.push(message);

    const bufferTokens = this.countBufferTokens();
    if (bufferTokens > this.maxBufferTokens) {
      await this.compressBuffer(summarizer);
    }
  }

  private async compressBuffer(summarizer: LLMSummarizer): Promise<void> {
    const halfPoint = Math.floor(this.buffer.length / 2);
    const toSummarize = this.buffer.slice(0, halfPoint);

    this.summary = await summarizer.summarize(
      this.summary,
      toSummarize
    );

    this.buffer = this.buffer.slice(halfPoint);
  }

  getContext(): { summary: string; recentMessages: Message[] } {
    return {
      summary: this.summary,
      recentMessages: this.buffer
    };
  }
}
```

### Topic-Based Summarization

AWS AgentCore's approach creates running narratives of conversations under different topics, scoped to sessions, preserving key information in structured XML format.

```typescript
interface TopicSummary {
  topic: string;
  summary: string;
  lastUpdated: Date;
  messageCount: number;
}

class TopicBasedMemory {
  private topics: Map<string, TopicSummary> = new Map();

  async addMessage(
    message: Message,
    topicClassifier: (msg: Message) => Promise<string>,
    summarizer: LLMSummarizer
  ): Promise<void> {
    const topic = await topicClassifier(message);
    const existing = this.topics.get(topic);

    if (existing) {
      const newSummary = await summarizer.updateSummary(
        existing.summary,
        message
      );
      this.topics.set(topic, {
        ...existing,
        summary: newSummary,
        lastUpdated: new Date(),
        messageCount: existing.messageCount + 1
      });
    } else {
      this.topics.set(topic, {
        topic,
        summary: message.content,
        lastUpdated: new Date(),
        messageCount: 1
      });
    }
  }

  getRelevantSummaries(query: string, limit: number = 3): TopicSummary[] {
    return Array.from(this.topics.values())
      .sort((a, b) => b.lastUpdated.getTime() - a.lastUpdated.getTime())
      .slice(0, limit);
  }
}
```

---

## 3. Memory Systems

### The memory.md Pattern

The memory.md pattern is a file-based approach providing AI agents with persistent memory across sessions. Instead of complex vector databases, it uses transparent Markdown files.

**Key principles:**
- **MEMORY.md** = curated, distilled knowledge (preferences, decisions, lessons learned)
- **Daily notes** = raw logs (everything captured)
- Keep memory lean: only information essential for every session
- Cap at ~30 items, remove outdated entries, merge duplicates

### Episodic vs. Semantic Memory

Humans use three types of memory, and AI agents can mirror this:

| Memory Type | Human Analog | AI Implementation |
|-------------|--------------|-------------------|
| **Semantic** | Facts learned in school | Vector database + embeddings; facts extracted from conversations |
| **Episodic** | Specific experiences | Logged events, actions, and outcomes in structured format |
| **Procedural** | Skills and rules | System prompts, few-shot examples, optimized with feedback |

```typescript
interface SemanticMemory {
  fact: string;
  source: string; // conversation ID or document
  confidence: number;
  lastAccessed: Date;
}

interface EpisodicMemory {
  event: string;
  context: string;
  outcome: string;
  reasoning: string; // chain of thought that led to success
  timestamp: Date;
}

interface MemorySystem {
  semantic: SemanticMemory[];
  episodic: EpisodicMemory[];
  procedural: string; // system prompt with learned rules
}
```

### MemGPT: Virtual Context Management

MemGPT treats context management like an operating system's virtual memory, with paging between "RAM" (in-context) and "disk" (out-of-context).

**Key characteristics:**
1. **Two-tier memory architecture:** main context (in-context) and external context (out-of-context)
2. **Self-editing memory:** LLM uses function calls to manage its own memory

```typescript
interface MemGPTContext {
  // Main context (in the prompt)
  systemPrompt: string;
  workingMemory: string; // editable by the agent
  recentMessages: Message[];

  // External context (stored, retrievable)
  archivalMemory: VectorStore;
  recallMemory: Message[]; // full conversation history
}

const memoryTools = [
  {
    name: 'core_memory_append',
    description: 'Append to working memory',
    execute: (content: string) => { /* add to workingMemory */ }
  },
  {
    name: 'core_memory_replace',
    description: 'Replace content in working memory',
    execute: (old: string, new_: string) => { /* replace in workingMemory */ }
  },
  {
    name: 'archival_memory_insert',
    description: 'Store in archival memory',
    execute: (content: string) => { /* add to vector store */ }
  },
  {
    name: 'archival_memory_search',
    description: 'Search archival memory',
    execute: (query: string) => { /* semantic search */ }
  },
  {
    name: 'conversation_search',
    description: 'Search past conversations',
    execute: (query: string, range: DateRange) => { /* search recallMemory */ }
  }
];
```

### Multi-Level Memory Hierarchies

The most sophisticated approach maintains different memory types at different time scales:

```typescript
interface HierarchicalMemory {
  // Immediate: current session context
  workingMemory: {
    currentGoal: string;
    recentExchanges: Message[];
    scratchpad: string;
  };

  // Short-term: this session's important moments
  sessionMemory: {
    sessionId: string;
    keyEvents: EpisodicMemory[];
    learnedFacts: SemanticMemory[];
  };

  // Long-term: persisted across sessions
  persistentMemory: {
    userPreferences: Record<string, unknown>;
    projectKnowledge: SemanticMemory[];
    successfulPatterns: EpisodicMemory[];
  };
}
```

---

## 4. Optimal Context Lengths

### Benchmark Data

| Context Length | Performance Characteristics |
|----------------|----------------------------|
| **4K-8K** | Optimal for most tasks; throughput 37-41 tok/s with Flash Attention |
| **16K** | Performance saturation point for many models (GPT-4-turbo, Claude-3-sonnet); throughput drops to 20-31 tok/s |
| **32K** | Most open-source models fail RULER benchmark at this length; only GPT-4, Claude, Yi-34B maintain strong performance |
| **64K+** | Only latest models (GPT-4o, Claude-3.5-sonnet, GPT-4o-mini) show no performance deterioration |

### RAG Performance at Different Lengths

Research from Databricks shows:
- Performance increases from 2K to 4K across all models
- Saturation points vary: 16K for GPT-4-turbo, 4K for Mixtral, 8K for DBRX
- Recent models (GPT-4o, Claude-3.5-sonnet) show little deterioration at higher lengths

### Memory Requirements

For an 8B parameter model with 4-bit quantization:

| Context Length | Memory Required |
|----------------|-----------------|
| 8K | ~6 GB |
| 16K | ~8 GB |
| 32K | ~10-11 GB |

### Recommendations by Use Case

```typescript
const contextLengthRecommendations = {
  chatbot: {
    recommended: 4_000,
    max: 8_000,
    rationale: 'Quick, relevant replies; most conversational context fits'
  },
  documentAnalysis: {
    recommended: 16_000,
    max: 32_000,
    rationale: 'Sufficient for most documents; beyond this, use chunking'
  },
  codingAssistant: {
    recommended: 8_000,
    max: 16_000,
    rationale: 'Balance between file context and response quality'
  },
  agenticWorkflows: {
    recommended: 16_000,
    max: 32_000,
    rationale: 'Preserve action/reasoning history; use compression for observations'
  }
};
```

---

## 5. Chunking and Pagination

### Chunking Strategies

| Strategy | Best For | Trade-offs |
|----------|----------|------------|
| **Fixed-size** | Uniform content | May break semantic units |
| **Semantic** | Coherent retrieval | Computationally expensive |
| **Sliding window with overlap** | Context continuity | Redundant embeddings |
| **Agentic/Adaptive** | Mixed content types | Requires LLM calls |
| **On-the-fly** | Live data (chat, transcripts) | Higher processing power |

### Sliding Window with Overlap

```typescript
function slidingWindowChunk(
  text: string,
  chunkSize: number = 512,
  overlap: number = 128
): string[] {
  const tokens = tokenize(text);
  const chunks: string[] = [];

  for (let i = 0; i < tokens.length; i += (chunkSize - overlap)) {
    const chunk = tokens.slice(i, i + chunkSize);
    chunks.push(detokenize(chunk));

    if (i + chunkSize >= tokens.length) break;
  }

  return chunks;
}
```

### Chunk Expansion for Context

Retrieve neighboring chunks within a window for each retrieved chunk:

```typescript
async function expandedRetrieval(
  query: string,
  vectorStore: VectorStore,
  expansionWindow: number = 1
): Promise<string[]> {
  const results = await vectorStore.similaritySearch(query, 3);

  const expanded = results.flatMap(result => {
    const chunks: string[] = [];
    for (let i = -expansionWindow; i <= expansionWindow; i++) {
      const neighborIndex = result.chunkIndex + i;
      const neighbor = vectorStore.getChunkByIndex(neighborIndex);
      if (neighbor) chunks.push(neighbor);
    }
    return chunks;
  });

  return [...new Set(expanded)]; // deduplicate
}
```

---

## 6. Context Compression Techniques

### LLMLingua-Style Compression

Microsoft Research's LLMLingua achieves up to **20x compression with only 1.5% performance loss**.

The technique:
1. Use a small LLM to calculate token perplexity
2. Remove tokens with lower perplexity (less information entropy)
3. Apply differential compression ratios:
   - Instructions: 10-20% compression
   - Examples: 60-80% compression
   - Questions: 0-10% compression

```typescript
interface CompressionConfig {
  instructionRatio: number; // 0.1-0.2
  exampleRatio: number;     // 0.6-0.8
  questionRatio: number;    // 0-0.1
}

async function compressPrompt(
  prompt: StructuredPrompt,
  compressor: PerplexityCompressor,
  config: CompressionConfig
): Promise<string> {
  const compressedInstruction = await compressor.compress(
    prompt.instruction,
    config.instructionRatio
  );

  const compressedExamples = await Promise.all(
    prompt.examples.map(ex => compressor.compress(ex, config.exampleRatio))
  );

  const compressedQuestion = await compressor.compress(
    prompt.question,
    config.questionRatio
  );

  return [
    compressedInstruction,
    ...compressedExamples,
    compressedQuestion
  ].join('\n\n');
}
```

### Relevance Filtering

Include only context pieces truly relevant to the target request:

```typescript
async function relevanceFilter(
  context: string[],
  query: string,
  relevanceThreshold: number = 0.7,
  embedder: Embedder
): Promise<string[]> {
  const queryEmbedding = await embedder.embed(query);
  const contextEmbeddings = await Promise.all(
    context.map(c => embedder.embed(c))
  );

  return context.filter((_, i) => {
    const similarity = cosineSimilarity(queryEmbedding, contextEmbeddings[i]);
    return similarity >= relevanceThreshold;
  });
}
```

### Compression Performance

| Compression Level | Cost Reduction | Accuracy Trade-off |
|-------------------|----------------|-------------------|
| Moderate (5-7x) | 85-90% | 5-15% (acceptable for most apps) |
| Aggressive (10-20x) | 90-95% | Requires careful validation |

---

## 7. Prompt Caching

### Claude/Anthropic Prompt Caching

Anthropic's prompt caching reduces costs by up to 90% and latency by up to 85% for long prompts.

**Pricing:**
- Cache write: 25% more than base input token price
- Cache read: 10% of base input token price
- Default TTL: 5 minutes (refreshed on use; 1-hour available at additional cost)

**Implementation:**

```typescript
const response = await anthropic.messages.create({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 1024,
  system: [
    {
      type: 'text',
      text: 'You are a helpful assistant with deep knowledge of...',
      cache_control: { type: 'ephemeral' } // Mark for caching
    }
  ],
  messages: [
    {
      role: 'user',
      content: [
        {
          type: 'text',
          text: largeDocumentContent,
          cache_control: { type: 'ephemeral' } // Cache the document
        }
      ]
    },
    {
      role: 'user',
      content: 'What are the key points?'
    }
  ]
});
```

**Best practices:**
- Place static content (system instructions, documents, examples) at the beginning
- Use cache breakpoints strategically to separate cacheable sections
- Minimum 1,024 tokens per cache checkpoint (Claude 3.7+)
- Set explicit cache breakpoint at conversation end for maximum hits

---

## 8. Multi-Channel Specific Recommendations

### Architecture for Multi-Channel Chat Assistants

```typescript
interface ChannelConfig {
  channelId: string;
  type: 'whatsapp' | 'telegram' | 'discord' | 'slack' | 'signal' | 'imessage';
  contextConfig: {
    maxTokens: number;
    summaryThreshold: number;
    persistMemory: boolean;
  };
}

const channelDefaults: Record<string, Partial<ChannelConfig['contextConfig']>> = {
  whatsapp: {
    maxTokens: 4000,      // Quick responses preferred
    summaryThreshold: 10, // Summarize after 10 messages
    persistMemory: true
  },
  telegram: {
    maxTokens: 8000,      // More flexibility
    summaryThreshold: 15,
    persistMemory: true
  },
  discord: {
    maxTokens: 8000,      // Thread-based, can handle more
    summaryThreshold: 20,
    persistMemory: true
  },
  slack: {
    maxTokens: 8000,      // Thread-based
    summaryThreshold: 20,
    persistMemory: true
  }
};
```

### Cross-Channel Context Continuity

40% of consumers abandon if experience is not seamless across channels. Implement unified context:

```typescript
interface UserContext {
  userId: string;
  channels: Map<string, ChannelContext>;
  sharedMemory: {
    preferences: Record<string, unknown>;
    facts: SemanticMemory[];
    recentTopics: string[];
  };
}

class CrossChannelMemory {
  private users: Map<string, UserContext> = new Map();

  async switchChannel(
    userId: string,
    fromChannel: string,
    toChannel: string
  ): Promise<string> {
    const user = this.users.get(userId);
    if (!user) return '';

    // Create handoff summary
    const fromContext = user.channels.get(fromChannel);
    if (!fromContext) return '';

    return this.createHandoffSummary(fromContext, user.sharedMemory);
  }

  private createHandoffSummary(
    channelContext: ChannelContext,
    sharedMemory: UserContext['sharedMemory']
  ): string {
    return `
Continuing conversation from previous channel.
Recent topics: ${sharedMemory.recentTopics.slice(0, 3).join(', ')}
Key context: ${channelContext.summary}
User preferences: ${JSON.stringify(sharedMemory.preferences)}
    `.trim();
  }
}
```

### Per-Channel Optimization Strategy

```typescript
class MultiChannelContextManager {
  private configs: Map<string, ChannelConfig> = new Map();

  async processMessage(
    channelId: string,
    channelType: string,
    message: Message,
    history: Message[]
  ): Promise<{ context: Message[]; summary: string | null }> {
    const config = this.getConfig(channelId, channelType);

    // 1. Apply channel-specific token limit
    let context = this.tokenTrim(history, config.maxTokens);

    // 2. Check if summarization needed
    let summary: string | null = null;
    if (history.length > config.summaryThreshold) {
      const oldMessages = history.slice(0, -config.summaryThreshold);
      summary = await this.summarize(oldMessages);
      context = [
        { role: 'system', content: `Previous context summary: ${summary}` },
        ...context
      ];
    }

    // 3. Apply compression if still over budget
    if (this.countTokens(context) > config.maxTokens * 0.9) {
      context = await this.compressContext(context, config.maxTokens);
    }

    return { context, summary };
  }
}
```

---

## 9. Code Examples

### Complete Context Manager Implementation

```typescript
import { encode } from 'gpt-tokenizer';

interface Message {
  role: 'system' | 'user' | 'assistant';
  content: string;
  timestamp: number;
  metadata?: {
    channelId?: string;
    importance?: number;
  };
}

interface ContextManagerConfig {
  maxTokens: number;
  summaryThreshold: number;
  compressionEnabled: boolean;
  cachingEnabled: boolean;
}

class ContextManager {
  private config: ContextManagerConfig;
  private summary: string = '';
  private buffer: Message[] = [];
  private semanticMemory: Map<string, string> = new Map();

  constructor(config: ContextManagerConfig) {
    this.config = config;
  }

  countTokens(text: string): number {
    return encode(text).length;
  }

  async addMessage(message: Message): Promise<void> {
    this.buffer.push(message);

    // Extract semantic memories
    await this.extractFacts(message);

    // Check if we need to compress
    if (this.buffer.length > this.config.summaryThreshold) {
      await this.compressHistory();
    }
  }

  private async extractFacts(message: Message): Promise<void> {
    // Use LLM to extract facts worth remembering
    const facts = await this.llmExtractFacts(message.content);
    facts.forEach(fact => {
      this.semanticMemory.set(fact.key, fact.value);
    });
  }

  private async compressHistory(): Promise<void> {
    const halfPoint = Math.floor(this.buffer.length / 2);
    const toSummarize = this.buffer.slice(0, halfPoint);

    // Recursive summarization
    this.summary = await this.llmSummarize(this.summary, toSummarize);
    this.buffer = this.buffer.slice(halfPoint);
  }

  async getContext(currentMessage: string): Promise<Message[]> {
    const context: Message[] = [];
    let tokenCount = 0;
    const budget = this.config.maxTokens;

    // 1. Always include system prompt
    const systemPrompt = this.buildSystemPrompt();
    context.push({ role: 'system', content: systemPrompt, timestamp: 0 });
    tokenCount += this.countTokens(systemPrompt);

    // 2. Add summary if exists
    if (this.summary) {
      const summaryMsg = `[Previous conversation summary]\n${this.summary}`;
      context.push({ role: 'system', content: summaryMsg, timestamp: 0 });
      tokenCount += this.countTokens(summaryMsg);
    }

    // 3. Add relevant semantic memories
    const relevantMemories = await this.getRelevantMemories(currentMessage);
    if (relevantMemories.length > 0) {
      const memoryContent = `[Relevant context]\n${relevantMemories.join('\n')}`;
      context.push({ role: 'system', content: memoryContent, timestamp: 0 });
      tokenCount += this.countTokens(memoryContent);
    }

    // 4. Add recent messages (most recent first, then trim)
    const remainingBudget = budget - tokenCount;
    const recentMessages = this.selectRecentMessages(remainingBudget);
    context.push(...recentMessages);

    return context;
  }

  private selectRecentMessages(budget: number): Message[] {
    const result: Message[] = [];
    let tokenCount = 0;

    // Work backwards from most recent
    for (let i = this.buffer.length - 1; i >= 0; i--) {
      const msg = this.buffer[i];
      const msgTokens = this.countTokens(msg.content);

      if (tokenCount + msgTokens > budget) break;

      result.unshift(msg);
      tokenCount += msgTokens;
    }

    return result;
  }

  private buildSystemPrompt(): string {
    const memories = Array.from(this.semanticMemory.entries())
      .slice(0, 10)
      .map(([k, v]) => `- ${k}: ${v}`)
      .join('\n');

    return `You are a helpful assistant.

Known facts about the user:
${memories || 'None yet.'}`;
  }

  private async getRelevantMemories(query: string): Promise<string[]> {
    // Simple keyword matching; in production, use embeddings
    const queryWords = query.toLowerCase().split(/\s+/);
    return Array.from(this.semanticMemory.entries())
      .filter(([key]) => queryWords.some(w => key.toLowerCase().includes(w)))
      .map(([k, v]) => `${k}: ${v}`);
  }

  // Placeholder for LLM calls
  private async llmExtractFacts(content: string): Promise<Array<{key: string; value: string}>> {
    // Implement with your LLM provider
    return [];
  }

  private async llmSummarize(existingSummary: string, messages: Message[]): Promise<string> {
    // Implement with your LLM provider
    return existingSummary;
  }
}
```

---

## 10. Benchmarks and Performance Data

### Context Length vs. Performance

| Model | Claimed Length | Actual Effective Length | Performance at Max |
|-------|----------------|------------------------|-------------------|
| GPT-4o | 128K | 128K | Minimal degradation |
| Claude 3.5 Sonnet | 200K | 200K | Minimal degradation |
| GPT-4-turbo | 128K | 16K optimal | Saturation at 16K |
| Mistral | 32K | 8K-16K optimal | Significant drop at 32K |
| Open-source 7B models | 32K+ | Often <8K | Many fail at claimed lengths |

### Memory System Comparison

| Approach | Token Reduction | Quality Impact | Implementation Complexity |
|----------|-----------------|----------------|--------------------------|
| Sliding Window | 50-80% | Loses early context | Low |
| Summarization | 70-90% | Lossy, may miss details | Medium |
| Hybrid (Summary + Buffer) | 80-90% | Good balance | Medium |
| MemGPT-style | 90%+ | Excellent, agent-controlled | High |
| RAG + Compression | 85-95% | Excellent for retrieval tasks | High |

### Mem0 Benchmark Results

Intelligent memory systems like Mem0 deliver:
- **26% relative improvement** in response quality
- **90%+ reduction** in token usage

### Cost Optimization Summary

| Technique | Cost Reduction | When to Use |
|-----------|----------------|-------------|
| Prompt Caching (Anthropic) | Up to 90% | Repeated system prompts, documents |
| Context Compression | 85-95% | Long conversations, document QA |
| Summarization | 70-90% | General chat, multi-turn |
| RAG | Variable (60-80%) | Knowledge-intensive tasks |
| Sliding Window | 50-80% | Simple chatbots, quick responses |

---

## Summary: Recommended Strategy for Multi-Channel Chat Assistant

1. **Use a hybrid memory system:**
   - Short-term: Sliding window with 10-20 recent messages
   - Medium-term: Session summaries (recursive summarization)
   - Long-term: Semantic memory store (vector DB or memory.md pattern)

2. **Implement channel-specific context budgets:**
   - WhatsApp/iMessage: 4K tokens (quick responses)
   - Telegram/Discord/Slack: 8K tokens (more flexibility)
   - Document analysis: 16K+ with chunking

3. **Enable prompt caching** for system prompts and frequently-used documents

4. **Apply compression selectively:**
   - Aggressive on examples and old context
   - Minimal on instructions and current query

5. **Maintain cross-channel context continuity** with shared semantic memory

6. **Start new sessions for new tasks** to avoid context pollution

---

## Sources

- [Effective context engineering for AI agents - Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Context Window Management Strategies - GetMaxim](https://www.getmaxim.ai/articles/context-window-management-strategies-for-long-context-ai-agents-and-chatbots/)
- [LLM Context Management Guide - 16x Engineer](https://eval.16x.engineer/blog/llm-context-management-guide)
- [Top techniques to Manage Context Lengths in LLMs - Agenta](https://agenta.ai/blog/top-6-techniques-to-manage-context-length-in-llms)
- [Context Length Guide 2025 - Local AI Zone](https://local-ai-zone.github.io/guides/context-length-optimization-ultimate-guide-2025.html)
- [Cutting Through the Noise: Efficient Context Management - JetBrains Research](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)
- [LLM Chat History Summarization Guide - Mem0](https://mem0.ai/blog/llm-chat-history-summarization-guide-2025)
- [Memory for agents - LangChain Blog](https://www.blog.langchain.com/memory-for-agents/)
- [LangMem SDK for agent long-term memory - LangChain](https://www.blog.langchain.com/langmem-sdk-launch/)
- [Long-Term Memory in LLM Applications - LangMem Docs](https://langchain-ai.github.io/langmem/concepts/conceptual_guide/)
- [Building smarter AI agents: AgentCore long-term memory - AWS](https://aws.amazon.com/blogs/machine-learning/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/)
- [Conversational Memory for LLMs with Langchain - Pinecone](https://www.pinecone.io/learn/series/langchain/langchain-conversational-memory/)
- [MemGPT: Towards LLMs as Operating Systems - arXiv](https://arxiv.org/abs/2310.08560)
- [MemGPT Research](https://research.memgpt.ai/)
- [Virtual context management with MemGPT - Leonie Monigatti](https://www.leoniemonigatti.com/blog/memgpt.html)
- [Long Context RAG Performance of LLMs - Databricks](https://www.databricks.com/blog/long-context-rag-performance-llms)
- [How Long Can Open-Source LLMs Truly Promise on Context Length? - LMSYS](https://lmsys.org/blog/2023-06-29-longchat/)
- [Sliding Window Attention Training - arXiv](https://arxiv.org/html/2502.18845v1)
- [Sliding Window Attention (SWA) - Sebastian Raschka](https://sebastianraschka.com/llms-from-scratch/ch04/06_swa/)
- [A-Mem: Agentic Memory for LLM Agents - arXiv](https://arxiv.org/html/2502.12110v1)
- [Chunking Strategies for LLM Applications - Pinecone](https://www.pinecone.io/learn/chunking-strategies/)
- [LLM Chunking: How to Improve Retrieval & Accuracy - Redis](https://redis.io/blog/llm-chunking/)
- [Prompt Compression for Large Language Models: A Survey - NAACL 2025](https://github.com/ZongqianLi/Prompt-Compression-Survey)
- [Prompt Compression for LLM Generation Optimization - Machine Learning Mastery](https://machinelearningmastery.com/prompt-compression-for-llm-generation-optimization-and-cost-reduction/)
- [Prompt caching - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Prompt caching with Claude - Anthropic](https://www.anthropic.com/news/prompt-caching)
- [CLAUDE.md: Building Persistent Memory for AI Coding Agents - DEV](https://dev.to/evoleinik/claudemd-building-persistent-memory-for-ai-coding-agents-5322)
- [Mem0 - Memory Layer for AI Apps](https://mem0.ai/)
- [Build smarter AI agents with Redis Memory](https://redis.io/blog/build-smarter-ai-agents-manage-short-term-and-long-term-memory-with-redis/)
