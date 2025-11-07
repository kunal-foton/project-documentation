---
title: <> code changes
sidebar_position: 6
---

# Project Grace - Cloudflare Workers Migration Plan


1. [Current State Analysis](#1-current-state-analysis)
2. [Proposed Architecture](#2-proposed-architecture)
3. [Detailed Implementation Plan](#3-detailed-implementation-plan)
4. [File-by-File Change Descriptions](#4-file-by-file-change-descriptions)
5. [Testing & Deployment Strategy](#5-testing--deployment-strategy)

---

## 1. Current State Analysis

### 1.1 Architecture Overview

**Frontend (React + TypeScript)**
- Located in: `src/` directory
- Key files:
  - `src/services/openaiService.js` - AI API calls (lines 764-1674)
  - `src/hooks/useAudioRecording.ts` - Audio/VAD handling
  - `src/components/InputBar.tsx` - UI component for user input

**Backend (Express.js - Simple Server)**
- File: `simple-server.js`
- Responsibilities:
  1. **Transcription API** (`/api/transcribe`, lines 76-147) - Routes to Fireworks Whisper ⚠️ BOTTLENECK
  2. **WebSocket Proxy** (lines 558-584) - Proxies to Unreal Engine instances ✅ MUST KEEP
  3. **Session Management** (lines 212-384) - User session handling ✅ MUST KEEP
  4. **Static File Serving** (lines 590-600) - Serves built frontend ✅ MUST KEEP

### 1.2 Current API Call Patterns

#### Text Messages Flow
```
User types "hi"
    ↓
Frontend (InputBar.tsx:185)
    ↓
openaiService.callOpenAI() (openaiService.js:764)
    ↓
AI Provider Selection (line 1179):
    ├─→ Cerebras API (if TEXT_PROVIDER='cerebras')
    ├─→ Groq API (if TEXT_PROVIDER='groq')
    └─→ OpenAI API (if TEXT_PROVIDER='openai')
    ↓
Direct API call to provider
    ↓
Response returned to frontend
    ↓
Display to user

⏱️ Time: ~410ms (after warmup)
```

#### Voice Messages Flow (CURRENT BOTTLENECK)
```
User speaks "hi"
    ↓
VAD Detection (useAudioRecording.ts:186-283)
    ↓
Audio captured as Float32Array
    ↓
Convert to Blob (audioUtils.ts)
    ↓
transcribeAudio() (useAudioRecording.ts:147)
    ↓
POST /api/transcribe (Simple Server)  ⚠️ BOTTLENECK HERE
    ↓
simple-server.js:76 → Fireworks Whisper API
    ↓
Transcription returned to frontend
    ↓
Frontend calls openaiService.callOpenAI()
    ↓
Same flow as text messages above
    ↓
Response returned to user

⏱️ Time: ~2,300ms total
```


### 1.3 Identified Bottlenecks

#### 1. Transcription Routing (PRIMARY BOTTLENECK)
- **Issue**: Audio goes Frontend → Simple Server → Fireworks → Simple Server → Frontend
- **Impact**: Adds ~1,000-1,500ms of unnecessary network hops
- **Solution**: Move to edge with Cloudflare Workers

#### 2. No Response Caching
- **Issue**: Every "hi" or "hello" calls full AI API
- **Solution**: Implement KV cache for common queries

#### 3. Cold Start Penalty
- **Issue**: First API call takes 3,447ms
- **Impact**: Poor first impression for new users
- **Solution**: Edge-based warmup and connection pooling

#### 4. Geographic Latency
- **Issue**: Users in India connecting to US-based servers
- **Impact**: High baseline latency (~200-300ms round-trip)
- **Solution**: Cloudflare's global edge network (200+ locations)

### 1.4 Current Dependencies

**API Providers** (from `.env.example`):
- OpenAI (GPT-4.1-nano) - Text & Vision
- Cerebras (Llama 3.1 8B) - Text (primary)
- Groq (Llama 3.1 8B Instant) - Text (alternative)
- Fireworks (Whisper V3 Turbo) - Transcription

**Current Provider Selection** (`openaiService.js:36`):
```javascript
const TEXT_PROVIDER = 'cerebras'; // Active provider
```

**Firebase Usage**:
- Authentication
- User profiles (Firestore)
- Temp image uploads (Storage)
- User summaries/chat history

---

## 2. Proposed Architecture

### 2.1 Target Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                         USER DEVICE                          │
│  (React Frontend - served via Cloudflare Pages/CDN)         │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │
┌────────────────┴────────────────────────────────────────────┐
│               CLOUDFLARE EDGE NETWORK                        │
│                (Nearest Location - e.g., Mumbai)             │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  WORKER #1: Transcription Worker                     │   │
│  │  Route: /cf/api/transcribe                          │   │
│  │  ├─→ Try: Cloudflare AI Whisper (fastest)          │   │
│  │  └─→ Fallback: Fireworks Whisper API               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  WORKER #2: Text Chat Worker (Optional)             │   │
│  │  Route: /cf/api/chat                                │   │
│  │  ├─→ Check KV Cache                                 │   │
│  │  ├─→ Try: Cloudflare AI (Llama 3)                  │   │
│  │  ├─→ Fallback: Cerebras API                        │   │
│  │  └─→ Cache successful response                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  KV NAMESPACE: Response Cache                        │   │
│  │  Key: hash(userMessage + context)                   │   │
│  │  Value: {response, timestamp, provider}             │   │
│  │  TTL: 1 hour for dynamic, 24h for greetings         │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                               │
└────┬────────────────────────────────────────────────────────┘
     │
     │ (Worker reaches out to AI providers from edge)
     │
     ├─────────────┬────────────────┬─────────────────┐
     ▼             ▼                ▼                 ▼
┌─────────┐  ┌──────────┐   ┌────────────┐   ┌──────────────┐
│Cloudflare│  │ Cerebras │   │  Fireworks │   │    OpenAI    │
│   AI     │  │   API    │   │    API     │   │     API      │
│ (Llama3) │  │          │   │  (Whisper) │   │   (Vision)   │
└─────────┘  └──────────┘   └────────────┘   └──────────────┘

                 ┌──────────────────────────┐
                 │  SIMPLE SERVER (Express) │
                 │  ✅ REMAINS UNCHANGED    │
                 ├──────────────────────────┤
                 │ • WebSocket Proxy        │
                 │ • Session Management     │
                 │ • Instance Registration  │
                 │ • Static File Serving    │
                 └──────────────────────────┘
                          │
                          ▼
              ┌────────────────────────┐
              │  Unreal Engine Instances│
              │  (Pixel Streaming)      │
              └────────────────────────┘
```

### 2.2 Data Flow - NEW Architecture

#### Text Messages (with caching)
```
User: "hi"
    ↓
Cloudflare Worker (/cf/api/chat) [Mumbai Edge]
    ↓
Check KV Cache: hash("hi" + context)
    ├─→ HIT? Return cached response (less than10ms) ✅
    └─→ MISS? Continue...
            ↓
        Try Cloudflare AI (Llama 3) at edge
            ├─→ Success? Cache & return (~100-200ms) ✅
            └─→ Failure? Fallback to Cerebras
                    ↓
                Cerebras API from edge (~250-400ms)
                    ↓
                Cache successful response
                    ↓
                Return to user

```

#### Voice Messages (optimized)
```
User speaks
    ↓
VAD captures audio
    ↓
POST /cf/api/transcribe [Cloudflare Worker at Mumbai Edge]
    ↓
Try Cloudflare AI Whisper (at edge, ~200-400ms) ✅
    ├─→ Success? Return transcription
    └─→ Failure? Fallback to Fireworks (~500-800ms)
            ↓
        Transcription returned
            ↓
        Frontend calls /cf/api/chat (same as text flow above)
            ↓
        Response returned

```

### 2.3 Backward Compatibility Strategy

**Phase 1: Parallel Operation**
- Frontend supports BOTH endpoints:
  - `/api/transcribe` (old - Simple Server)
  - `/cf/api/transcribe` (new - Cloudflare Worker)
- Feature flag: `USE_CLOUDFLARE_WORKERS` (default: false)
- Monitor error rates and latency for both

**Phase 2: Gradual Migration**
- Enable for 10% of users
- Monitor for 48 hours
- Increase to 50%, then 100%

**Phase 3: Deprecation**
- Remove Simple Server `/api/transcribe` endpoint
- Clean up old code

---

## 3. Detailed Implementation Plan

### Phase 1: Setup & Infrastructure (Week 1)

#### Task 1.1: Cloudflare Account & Services Setup
**What**: Set up Cloudflare account and enable required services

**Actions**:
1. Create Cloudflare account (if not existing)
2. Add domain to Cloudflare (or use workers.dev subdomain for testing)
3. Enable Workers (free tier: 100,000 requests/day)
4. Enable Workers KV (free tier: 100,000 reads/day)
5. Enable Cloudflare AI (pay-as-you-go, ~$0.01 per 1K tokens)

**Configuration Needed**:
- Domain: `workers.fotonlabs.com` OR `grace-workers.workers.dev`
- KV Namespace: `RESPONSE_CACHE` (production)
- KV Namespace: `RESPONSE_CACHE_PREVIEW` (development)



---

#### Task 1.2: Create Cloudflare Workers Project Structure
**What**: Set up local development environment for Workers

**Actions**:
1. Install Wrangler CLI: `npm install -g wrangler`
2. Authenticate: `wrangler login`
3. Create new Workers project directory (outside existing repo)
4. Initialize project: `wrangler init project-grace-workers`
5. Choose TypeScript template

**Directory Structure**:
```
project-grace-workers/
├── src/
│   ├── workers/
│   │   ├── transcription-worker.ts    # Main transcription handler
│   │   ├── chat-worker.ts             # Optional: Text chat handler
│   │   └── shared/
│   │       ├── cache.ts               # KV cache utilities
│   │       ├── ai-providers.ts        # Provider clients
│   │       └── types.ts               # Shared TypeScript types
│   ├── index.ts                       # Router for all workers
│   └── bindings.d.ts                  # TypeScript bindings
├── wrangler.toml                      # Cloudflare configuration
├── package.json
├── tsconfig.json
└── README.md
```


---

#### Task 1.3: Configure Environment Variables & Secrets
**What**: Securely store API keys in Cloudflare

**Actions**:
1. Add secrets via Wrangler CLI:
   ```bash
   wrangler secret put FIREWORKS_API_KEY
   wrangler secret put CEREBRAS_API_KEY
   wrangler secret put OPENAI_API_KEY
   ```
2. Configure `wrangler.toml` with KV bindings
3. Set up environment-specific configs (dev/staging/prod)

**wrangler.toml Configuration**:
```toml
name = "project-grace-workers"
main = "src/index.ts"
compatibility_date = "2024-01-01"
node_compat = true

# KV Namespace Bindings
[[kv_namespaces]]
binding = "RESPONSE_CACHE"
id = "less thanproduction-kv-id>"
preview_id = "less thanpreview-kv-id>"

# AI Binding
[ai]
binding = "AI"

# Vars (non-secret config)
[vars]
ENVIRONMENT = "production"
CACHE_TTL_GREETINGS = 86400    # 24 hours
CACHE_TTL_DYNAMIC = 3600       # 1 hour
MAX_AUDIO_SIZE_MB = 25
```


---

### Phase 2: Transcription Worker Implementation

#### Task 2.1: Create Transcription Worker Core
**What**: Implement `/cf/api/transcribe` endpoint
**File to Create**: `src/workers/transcription-worker.ts`

**Core Functionality** (description only, not code):

1. **Request Validation**:
   - Accept `multipart/form-data` with audio file
   - Validate file size (max 25MB)
   - Validate file type (audio/wav, audio/webm, audio/mp3)
   - Extract optional parameters: language, temperature, prompt

2. **Cloudflare AI Whisper** (Primary - Fastest):
   - Use `@cloudflare/ai` binding
   - Model: `@cf/openai/whisper`
   - Convert audio to required format
   - Call Cloudflare AI with timeout (30s)
   - Parse response: `{text, segments, language}`

3. **Fireworks Whisper Fallback**:
   - If CF AI fails/unavailable
   - Use existing Fireworks API integration
   - FormData with same fields as Simple Server (lines 88-102)
   - Model: `whisper-v3-turbo`
   - Keep-alive connection for performance

4. **Response Normalization**:
   - Return consistent format:
     ```typescript
     {
       text: string,
       segments?: Arrayless than{start, end, text}>,
       detected_language?: string,
       duration?: number,
       provider: 'cloudflare' | 'fireworks',
       latency_ms: number
     }
     ```

5. **Error Handling**:
   - Proper HTTP status codes
   - Fallback chain: CF AI → Fireworks → Error
   - Logging for debugging (Cloudflare Workers Logs)

6. **Performance Monitoring**:
   - Track provider used
   - Track latency
   - Track cache hit/miss (for future caching of transcriptions)

**API Signature**:
```
POST /cf/api/transcribe
Content-Type: multipart/form-data

Request:
- audio: File (required)
- language: string (optional, e.g., "en")
- temperature: number (optional, 0-1)
- prompt: string (optional, context for transcription)

Response:
{
  "text": "transcribed text here",
  "segments": [...],
  "detected_language": "en",
  "duration": 3.5,
  "provider": "cloudflare",
  "latency_ms": 234
}
```
---

#### Task 2.2: Implement AI Provider Abstraction
**What**: Create reusable provider clients
**File to Create**: `src/workers/shared/ai-providers.ts`

**Provider Interface** (description):
```typescript
interface TranscriptionProvider {
  name: string;
  transcribe(audio: Blob, options: TranscriptionOptions): Promiseless thanTranscriptionResult>;
}

interface ChatProvider {
  name: string;
  chat(messages: Message[], tools: Tool[]): Promiseless thanChatResult>;
}
```

**Providers to Implement**:

1. **CloudflareAIProvider**:
   - Transcription: `@cf/openai/whisper`
   - Chat: `@cf/meta/llama-3-8b-instruct` or `@cf/meta/llama-3.1-8b-instruct`
   - Access via Workers AI binding

2. **FireworksProvider**:
   - Transcription: Whisper V3 Turbo
   - Endpoint: `https://audio-turbo.us-virginia-1.direct.fireworks.ai/v1/audio/transcriptions`
   - Keep-alive connections

3. **CerebrasProvider**:
   - Chat only: Llama 3.1 8B
   - Endpoint: `https://api.cerebras.ai/v1/chat/completions`
   - OpenAI-compatible format

**Error Handling Strategy**:
- Retry logic: 1 retry with exponential backoff
- Timeout: 30s for transcription, 10s for chat
- Fallback chain configurable per endpoint


---

#### Task 2.3: Add Request/Response Caching Utilities
**What**: Implement KV-based caching layer
**File to Create**: `src/workers/shared/cache.ts`

**Cache Strategy**:

1. **Cache Key Generation**:
   - For text: `hash(userMessage + agentName + contextSnippet)`
   - Use SHA-256 hash (built-in Web Crypto API)
   - Example: `chat:v1:sha256(message)`

2. **Cache Storage** (KV):
   ```typescript
   interface CachedResponse {
     response: string;
     functionName: string;
     functionArgs: any;
     provider: string;
     timestamp: number;
     ttl: number;
     hitCount: number;
   }
   ```

3. **TTL Strategy**:
   - Greetings ("hi", "hello", "hey"): 24 hours
   - Weather/news queries: 15 minutes
   - Dynamic responses: 1 hour
   - User-specific: No cache

4. **Cache Invalidation**:
   - Automatic via TTL
   - Manual via admin endpoint (future)

5. **Cache Warming** (Optional):
   - Pre-populate common greetings
   - Run on Worker Cron trigger (daily)

**Functions to Implement**:
- `getCachedResponse(key: string): Promiseless thanCachedResponse | null>`
- `setCachedResponse(key: string, value: CachedResponse, ttl: number)`
- `shouldCache(message: string): boolean`
- `getCacheTTL(message: string): number`
- `generateCacheKey(message: string, context: any): string`

---

### Phase 3: Frontend Integration 

#### Task 3.1: Create Cloudflare API Client
**What**: Add new API client for Cloudflare Workers
**File to Create**: `src/services/cloudflareService.ts`

**Functions to Implement**:

1. **`transcribeAudioCF(audioBlob: Blob): Promiseless thastring | null>`**:
   - POST to `/cf/api/transcribe`
   - FormData with audio file
   - Error handling with fallback
   - Return transcribed text

2. **`chatCF(message, agent, context): Promiseless thaChatResponse>`** (optional):
   - POST to `/cf/api/chat`
   - Same interface as `callOpenAI()`
   - Backward compatible

3. **Error Handling**:
   - Network errors → fallback to Simple Server
   - Timeout errors → fallback
   - 5xx errors → fallback
   - Log errors for monitoring

4. **Feature Flag Support**:
   - Check `localStorage.getItem('USE_CLOUDFLARE_WORKERS')`
   - Or environment variable
   - Default: false (use old endpoints)

**Example Structure**:
```typescript
// Feature flag (in config or .env)
const USE_CF_WORKERS = import.meta.env.VITE_USE_CF_WORKERS === 'true';
const CF_WORKERS_URL = import.meta.env.VITE_CF_WORKERS_URL || 'https://grace-workers.workers.dev';

export const transcribeAudioCF = async (audioBlob: Blob): Promiseless thanstring | null> => {
  // Implementation here - calls Cloudflare Worker
  // Includes timeout, retry logic, error handling
};

export const transcribeAudio = async (audioBlob: Blob): Promiseless thanstring | null> => {
  if (USE_CF_WORKERS) {
    try {
      return await transcribeAudioCF(audioBlob);
    } catch (error) {
      console.warn('CF Worker failed, falling back to Simple Server:', error);
      // Fallback to old endpoint
    }
  }

  // Existing implementation (Simple Server /api/transcribe)
  return transcribeAudioSimpleServer(audioBlob);
};
```


---

#### Task 3.2: Update useAudioRecording Hook
**What**: Switch to Cloudflare Workers endpoint
**File to Modify**: `src/hooks/useAudioRecording.ts`

**Changes Needed**:

**Line 160**: Replace `/api/transcribe` with feature-flagged endpoint

```typescript
// BEFORE (line 160):
const response = await fetch('/api/transcribe', {
  method: 'POST',
  body: formData
});

// AFTER:
import { transcribeAudio as transcribeAudioCF } from '../services/cloudflareService';

const USE_CF_WORKERS = import.meta.env.VITE_USE_CF_WORKERS === 'true';

// Inside transcribeAudio function (line 147):
if (USE_CF_WORKERS) {
  // Use Cloudflare Worker
  const result = await transcribeAudioCF(audioBlob);
  return result;
} else {
  // Use Simple Server (existing code)
  const formData = new FormData();
  formData.append('audio', audioBlob, 'recording.wav');

  const response = await fetch('/api/transcribe', {
    method: 'POST',
    body: formData
  });

  // ... rest of existing code
}
```

**Testing Checklist**:
- [ ] Manual recording works with CF endpoint
- [ ] Continuous mode works with CF endpoint
- [ ] Fallback to Simple Server works on error
- [ ] Error messages are user-friendly
- [ ] Performance metrics logged to console


---

#### Task 3.3: Add Environment Variables
**What**: Configure build-time environment variables
**Files to Modify**:
1. `.env.development`
2. `.env.production`
3. Webpack config (if needed)

**New Environment Variables**:
```bash
# Cloudflare Workers Configuration
VITE_USE_CF_WORKERS=false           # Feature flag (default off)
VITE_CF_WORKERS_URL=https://grace-workers.workers.dev
VITE_ENABLE_CF_LOGGING=true         # Extra logging for debugging
```

**For Different Environments**:
- **Development**: `USE_CF_WORKERS=false` (use local Simple Server)
- **Staging**: `USE_CF_WORKERS=true` (test Workers)
- **Production**: `USE_CF_WORKERS=true` (after validation)

**Estimated Time**: 30 minutes
**Complexity**: Very Low

---

### Phase 4: Testing & Validation 

#### Task 4.1: Unit Testing - Workers
**What**: Test Workers in isolation
**Framework**: Vitest (recommended for Cloudflare Workers)

**Tests to Write**:

1. **Transcription Worker**:
   - Valid audio file → successful transcription
   - Invalid file type → 400 error
   - File too large → 413 error
   - Cloudflare AI failure → Fireworks fallback
   - Both providers fail → proper error response

2. **Cache Utilities**:
   - Cache key generation is consistent
   - TTL respects configured values
   - Cache hit returns correct data
   - Cache miss returns null

3. **Provider Clients**:
   - Fireworks client handles responses correctly
   - Cerebras client normalizes format
   - Timeout errors handled gracefully

**Test Command**: `wrangler dev` (local testing)


---

#### Task 4.2: Integration Testing - End-to-End
**What**: Test complete user flows

**Test Scenarios**:

1. **Voice Transcription - Happy Path**:
   - User clicks mic
   - Records 3 seconds of speech
   - Audio sent to `/cf/api/transcribe`
   - Transcription returned in less than1s
   - Text sent to AI
   - Response displayed

2. **Voice Transcription - Fallback**:
   - Temporarily disable Cloudflare AI
   - Verify Fireworks fallback works
   - Check latency is acceptable

3. **Feature Flag Toggle**:
   - Disable `USE_CF_WORKERS`
   - Verify Simple Server endpoint still works
   - Re-enable flag
   - Verify CF Worker works again

4. **Error Handling**:
   - Network offline → User sees error
   - Worker timeout → Fallback works
   - Invalid audio → Clear error message

5. **Performance Validation**:
   - Measure latency for 20 voice messages
   - Compare to Simple Server baseline
   - Target: 50%+ improvement

**Testing Tools**:
- Browser DevTools Network tab
- Cloudflare Workers Logs
- Performance.now() timestamps



---

#### Task 4.3: Load Testing
**What**: Test Workers under realistic load

**Scenarios**:
1. **Sustained Load**: 100 requests/minute for 10 minutes
2. **Burst Load**: 500 requests in 30 seconds
3. **Global Load**: Requests from multiple regions

**Tools**:
- K6 (load testing tool)
- Cloudflare Workers Analytics
- Custom scripts

**Metrics to Track**:
- P50, P95, P99 latency
- Error rate
- Provider failover rate
- Cache hit rate

**Success Criteria**:
- P95 latency less than 1s for transcription
- Error rate less than 0.1%
- Cache hit rate > 30% (after warmup)

---

### Phase 5: Deployment & Rollout

#### Task 5.1: Deploy to Staging
**What**: Deploy Workers to staging environment

**Steps**:
1. Create staging Workers:
   - `project-grace-workers-staging`
   - Separate KV namespace
   - Separate secrets

2. Deploy: `wrangler deploy --env staging`

3. Update staging frontend:
   - Set `VITE_CF_WORKERS_URL=https://grace-workers-staging.workers.dev`
   - Set `VITE_USE_CF_WORKERS=true`

4. Run full regression tests

5. Get team approval

**Validation Checklist**:
- [ ] All API endpoints working
- [ ] Latency improved vs Simple Server
- [ ] Error handling works
- [ ] Logs show proper metrics
- [ ] No security issues

---

#### Task 5.2: Monitoring & Observability
**What**: Set up production monitoring

**Metrics to Track**:

1. **Cloudflare Workers Analytics** (built-in):
   - Request count
   - Error rate
   - CPU time
   - KV operations

2. **Custom Metrics** (via Workers):
   - Provider used (cloudflare vs fireworks vs cerebras)
   - Latency per provider
   - Cache hit/miss rate
   - Fallback trigger rate

3. **Frontend Metrics** (existing):
   - End-to-end latency (user perspective)
   - Error messages shown to user
   - Feature flag distribution

4. **Alerts**:
   - Error rate > 1% → Alert team
   - P99 latency > 3s → Investigate
   - Provider failover rate > 20% → Check provider status

**Logging Strategy**:
```typescript
// In Workers
console.log(JSON.stringify({
  timestamp: Date.now(),
  endpoint: '/cf/api/transcribe',
  provider: 'cloudflare',
  latency_ms: 234,
  cache_hit: false,
  user_agent: request.headers.get('user-agent')
}));
```

**Tools**:
- Cloudflare Workers Logpush (push logs to S3/BigQuery)
- Grafana/Datadog (optional, for advanced monitoring)
- Simple dashboard with Cloudflare Analytics API


---

## 4. File-by-File Change Descriptions

### 4.1 New Files to Create

#### `project-grace-workers/` (New Repository)

```
project-grace-workers/
├── src/
│   ├── index.ts
│   │   Purpose: Main router, dispatches requests to appropriate worker
│   │   Logic: Route /cf/api/transcribe → transcription-worker
│   │          Route /cf/api/chat → chat-worker (optional)
│   │   Size: ~100 lines
│   │
│   ├── workers/
│   │   ├── transcription-worker.ts
│   │   │   Purpose: Handle audio transcription
│   │   │   Logic: Parse multipart/form-data
│   │   │          Try Cloudflare AI Whisper
│   │   │          Fallback to Fireworks
│   │   │          Return normalized response
│   │   │   Size: ~200-300 lines
│   │   │   Dependencies: @cloudflare/ai, FormData API
│   │   │
│   │   ├── chat-worker.ts (OPTIONAL)
│   │   │   Purpose: Handle text chat with caching
│   │   │   Logic: Check KV cache
│   │   │          Try CF AI → Cerebras → Groq → OpenAI
│   │   │          Cache response
│   │   │   Size: ~300-400 lines
│   │   │
│   │   └── shared/
│   │       ├── cache.ts
│   │       │   Purpose: KV cache utilities
│   │       │   Functions: getCached, setCached, generateKey, shouldCache
│   │       │   Size: ~150 lines
│   │       │
│   │       ├── ai-providers.ts
│   │       │   Purpose: AI provider client implementations
│   │       │   Classes: CloudflareAIProvider, FireworksProvider, CerebrasProvider
│   │       │   Size: ~400-500 lines
│   │       │
│   │       └── types.ts
│   │           Purpose: Shared TypeScript interfaces
│   │           Types: TranscriptionRequest, ChatRequest, etc.
│   │           Size: ~100 lines
│   │
│   └── bindings.d.ts
│       Purpose: TypeScript bindings for Cloudflare bindings (KV, AI)
│       Size: ~50 lines
│
├── wrangler.toml
│   Purpose: Cloudflare Workers configuration
│   Content: KV bindings, AI binding, secrets, routes
│   Size: ~50 lines
│
├── package.json
│   Purpose: Dependencies
│   Dependencies: @cloudflare/ai, @cloudflare/workers-types, typescript
│   DevDependencies: wrangler, vitest
│
├── tsconfig.json
│   Purpose: TypeScript configuration for Workers
│
└── README.md
    Purpose: Documentation for Workers project
    Content: Setup, deployment, architecture, API docs
```

---

### 4.2 Files to Modify (Existing Project)

#### `src/services/cloudflareService.ts` (NEW FILE)

```
File: src/services/cloudflareService.ts
Status: CREATE NEW

Purpose:
- API client for Cloudflare Workers
- Wraps Worker endpoints with error handling
- Provides fallback to Simple Server

Functions to Add:
1. transcribeAudioCF(audioBlob: Blob): Promiseless thanstring | null>
   - POST to /cf/api/transcribe
   - Parse response
   - Return transcribed text

2. chatCF(message, agent, context): Promiseless thanChatResponse> (optional)
   - POST to /cf/api/chat
   - Cache-aware
   - Return AI response

3. healthCheckCF(): Promiseless thanboolean>
   - Check if Workers are reachable
   - Used for automatic fallback

Size: ~200-250 lines
Complexity: Low
Dependencies: None (uses fetch API)
```

---

#### `src/hooks/useAudioRecording.ts` (MODIFY)

```
File: src/hooks/useAudioRecording.ts
Status: MODIFY

Changes Needed:

1. Import cloudflareService:
   Line 5 (add):
   import { transcribeAudioCF } from '../services/cloudflareService';

2. Add feature flag check:
   Line 147 (transcribeAudio function):

   BEFORE:
   const response = await fetch('/api/transcribe', {
     method: 'POST',
     body: formData
   });

   AFTER:
   const USE_CF_WORKERS = import.meta.env.VITE_USE_CF_WORKERS === 'true';

   let transcriptionText;
   if (USE_CF_WORKERS) {
     try {
       transcriptionText = await transcribeAudioCF(audioBlob);
       if (!transcriptionText) throw new Error('No transcription returned');
     } catch (error) {
       console.warn('CF Worker failed, falling back to Simple Server:', error);
       // Fallback to Simple Server (existing code below)
       const formData = new FormData();
       formData.append('audio', audioBlob, 'recording.wav');
       const response = await fetch('/api/transcribe', {
         method: 'POST',
         body: formData
       });
       const result = await response.json();
       transcriptionText = result.text;
     }
   } else {
     // Use Simple Server (existing code)
     const formData = new FormData();
     formData.append('audio', audioBlob, 'recording.wav');
     const response = await fetch('/api/transcribe', {
       method: 'POST',
       body: formData
     });
     const result = await response.json();
     transcriptionText = result.text;
   }

   return transcriptionText || null;

Lines Modified: ~160-175
Impact: Medium (core transcription flow)
Risk: Low (feature flag + fallback)
Testing: Critical path
```

---

#### `src/services/openaiService.js` (OPTIONAL MODIFY)

```
File: src/services/openaiService.js
Status: OPTIONAL MODIFY (for chat caching)

Changes Needed (if implementing chat worker):

1. Add cloudflareService import:
   Line 1 (add):
   import { chatCF } from './cloudflareService';

2. Wrap callOpenAI with cache check:
   Line 764 (callOpenAI function):

   Add at beginning:
   const USE_CF_WORKERS = import.meta.env.VITE_USE_CF_WORKERS === 'true';

   if (USE_CF_WORKERS) {
     try {
       const result = await chatCF(userMessage, agentName, widgetContext, userName);
       if (result) return result;
     } catch (error) {
       console.warn('CF Worker chat failed, using direct API:', error);
       // Continue to existing implementation
     }
   }

   // Existing implementation continues...

Lines Modified: ~764-780
Impact: Medium
Risk: Low (fallback to existing)
Priority: LOW (can defer)
Alternative: Implement caching client-side
```

---

#### `.env.development` & `.env.production` (MODIFY)

```
Files:
- .env.development
- .env.production

Status: MODIFY

Add New Variables:

# Cloudflare Workers Configuration
VITE_USE_CF_WORKERS=false                              # Feature flag
VITE_CF_WORKERS_URL=https://grace-workers.workers.dev  # Worker URL
VITE_ENABLE_CF_LOGGING=true                            # Debug logging

# For different environments:
# Development: USE_CF_WORKERS=false (test locally first)
# Staging: USE_CF_WORKERS=true, URL=staging-workers
# Production: USE_CF_WORKERS=true, URL=production-workers

Impact: None (additive only)
Risk: None
```

---

#### `package.json` (MODIFY - Frontend)

```
File: package.json (frontend)
Status: MODIFY (optional)

Changes (if adding client-side caching):

devDependencies:
  "@cloudflare/workers-types": "^4.20240925.0"  // For TypeScript types

Impact: Minimal
Risk: None
Size: +1 dependency
```

---

#### `simple-server.js` (NO CHANGES INITIALLY)

```
File: simple-server.js
Status: NO CHANGES (Phase 1-3)
       DEPRECATE /api/transcribe (Phase 4, after successful migration)

Current Responsibilities (KEEP):
- WebSocket proxying (lines 558-683) ✅
- Session management (lines 212-384) ✅
- Instance registration (lines 387-491) ✅
- Static file serving (lines 590-600) ✅

To Deprecate Later (after migration):
- /api/transcribe endpoint (lines 76-147)
  Action: Comment out or remove
  Timeline: After 2 weeks of 100% Cloudflare usage
  Impact: None (all traffic already migrated)

Risk: None (Simple Server remains critical for other functions)
```

---

#### `webpack.config.js` or `webpack.dev.js` (MODIFY - if needed)

```
Files: webpack.dev.js, webpack.prod.js
Status: MODIFY (if environment variables not already supported)

Changes:
Ensure Vite/Webpack can inject environment variables:

plugins: [
  new webpack.DefinePlugin({
    'import.meta.env.VITE_USE_CF_WORKERS': JSON.stringify(process.env.VITE_USE_CF_WORKERS),
    'import.meta.env.VITE_CF_WORKERS_URL': JSON.stringify(process.env.VITE_CF_WORKERS_URL)
  })
]

Note: If using Vite, this is already supported. Check if Webpack is still used.

Impact: Low
Risk: None
```

---

## 5. Testing & Deployment Strategy

### 5.1 Testing Phases

#### Phase 1: Local Development Testing

**Setup**:
1. Run Cloudflare Workers locally:
   ```bash
   cd project-grace-workers
   wrangler dev
   ```
   - Workers run at `http://localhost:8787`

2. Update frontend `.env.development`:
   ```bash
   VITE_USE_CF_WORKERS=true
   VITE_CF_WORKERS_URL=http://localhost:8787
   ```

3. Run frontend:
   ```bash
   npm run serve
   ```

**Test Cases**:

| Test Case | Action | Expected Result | Status |
|-----------|--------|----------------|---------|
| Voice Transcription - Happy Path | Record 3s of speech, send to worker | Transcription returned in less than1s | ⬜ |
| Voice Transcription - Long Audio | Record 30s of speech | Transcription successful | ⬜ |
| Voice Transcription - Invalid File | Send .txt file | 400 Bad Request error | ⬜ |
| Voice Transcription - Fallback | Disable CF AI, send audio | Fireworks fallback works | ⬜ |
| Chat Cache - Hit | Send "hi" twice | Second response less than10ms | ⬜ |
| Chat Cache - Miss | Send unique message | Full API call, then cached | ⬜ |
| Feature Flag - Disabled | Set USE_CF_WORKERS=false | Uses Simple Server | ⬜ |
| Error Handling - Network Offline | Disconnect network | Clear error message | ⬜ |

**Tools**:
- Browser DevTools (Network tab)
- Console logs
- Wrangler logs

---

#### Phase 2: Staging Environment Testing

**Setup**:
1. Deploy to staging:
   ```bash
   wrangler deploy --env staging
   ```
   - URL: `https://grace-workers-staging.workers.dev`

2. Update staging frontend:
   ```bash
   VITE_USE_CF_WORKERS=true
   VITE_CF_WORKERS_URL=https://grace-workers-staging.workers.dev
   ```

3. Deploy staging frontend to test server

**Test Cases**:
- All local tests repeated in cloud environment
- Cross-browser testing (Chrome, Firefox, Safari, Edge)
- Mobile testing (iOS, Android)
- Different network conditions (3G, 4G, WiFi)
- Geographic testing (if possible - use VPN)

**Performance Benchmarks**:

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| Voice transcription (CF AI) | less than500ms | __ ms | ⬜ |
| Voice transcription (Fireworks) | less than800ms | __ ms | ⬜ |
| Chat (cached) | less than10ms | __ ms | ⬜ |
| Chat (CF AI) | less than200ms | __ ms | ⬜ |
| Chat (Cerebras) | less than400ms | __ ms | ⬜ |
| Error rate | less than0.1% | __% | ⬜ |
| Fallback rate | less than5% | __% | ⬜ |

**Acceptance Criteria**:
- ✅ All functional tests pass
- ✅ Performance targets met
- ✅ No critical bugs
- ✅ Error handling works correctly
- ✅ Monitoring shows healthy metrics


---

### 5.2 Rollback Plan

#### Scenario 1: Critical Bug in Workers

**Symptoms**:
- Error rate spikes >5%
- Complete failure to transcribe
- Security vulnerability discovered

**Action**:
1. **Immediate**:
   ```bash
   # Update environment variable
   VITE_USE_CF_WORKERS=false

   # Redeploy frontend
   npm run build
   npm run deploy
   ```

2. **Verify**:
   - Check error rate returns to normal
   - Test voice/text messages work
   - Monitor Simple Server load

3. **Investigate** :
   - Review Cloudflare Workers logs
   - Identify root cause
   - Fix and redeploy to staging
   - Re-test before re-enabling


---

#### Scenario 3: Cloudflare Outage

**Symptoms**:
- All Workers requests fail
- Cloudflare status page shows incident
- Global DNS/network issues

**Action**:
1. **Automatic Fallback**:
   - Frontend already has fallback logic
   - Should automatically use Simple Server
   - Verify fallback is working

2. **Communication**:
   - Notify team
   - Monitor Cloudflare status
   - Prepare statement if needed

3. **Post-Incident**:
   - Review why fallback didn't work (if applicable)
   - Improve fallback logic
   - Consider multi-provider setup


---

### 5.3 Monitoring & Alerting

**Cloudflare Workers Dashboards**:

1. **Real-time Dashboard** (Cloudflare Analytics):
   - Requests per second
   - Error rate
   - P50/P95/P99 latency
   - CPU time usage
   - KV operations/sec

2. **Custom Dashboard** (build with Cloudflare Analytics API):
   - Provider usage breakdown
   - Cache hit/miss rate
   - Fallback trigger rate
   - Geographic distribution

**Alerts** (configure in Cloudflare):

| Alert | Condition | Action |
|-------|-----------|--------|
| High Error Rate | >1% errors for 5 min | Email + Slack |
| High Latency | P95 >3s for 5 min | Slack |
| Low Cache Hit | less than20% for 1 hour | Slack |
| Provider Failures | >10% fallbacks for 10 min | Email + Slack |
| KV Quota | >80% of quota | Email |

**Logging Strategy**:

In Workers, log structured JSON:
```typescript
console.log(JSON.stringify({
  timestamp: Date.now(),
  endpoint: '/cf/api/transcribe',
  provider: 'cloudflare',
  latency_ms: 234,
  cache_hit: false,
  audio_duration_sec: 3.5,
  user_agent: request.headers.get('user-agent')?.substring(0, 50),
  request_id: crypto.randomUUID()
}));
```

**Log Destinations**:
- Cloudflare Workers Logs (real-time tail)
- Logpush to S3/BigQuery (long-term storage)
- Optional: Datadog/Grafana integration

**Weekly Review**:
- Review error logs
- Analyze latency trends
- Optimize cache strategy
- Provider cost analysis

--6