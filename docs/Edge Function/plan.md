---
title : plan
sidebar_position: 5
---


#  plan
## Current Performance

**Response Time: ~410ms** (0.4 seconds)

**Current Path:**
```
User in India
    ↓
Simple Server (your location)
    ↓
AI Provider (Cerebras in US)
    ↓
Simple Server
    ↓
User receives response
```


### Voice Mode (speaking to Grace)

**Response Time: ~2,300ms** (2.3 seconds)

**Current Path:**
```
User stops speaking
    ↓
Simple Server (your location)
    ↓
Whisper API (transcription service in US)
    ↓
Simple Server
    ↓
AI Provider (Cerebras in US)
    ↓
Simple Server
    ↓
Grace starts responding
```

---

## Why the Current System is Slow

### The Problem: Too Many Stops

Think of it like a relay race where the baton passes through multiple runners:

**Text Messages:**
- 4 stops: User → Simple Server → AI → Simple Server → User
- Each stop adds time

**Voice Messages:**
- 6 stops: User → Simple Server → Whisper → Simple Server → AI → Simple Server → User
- Twice as many stops = much slower

### Geographic Distance

Your Simple Server is located far from:
- Most users (especially in India)
- The AI providers (in the US)

Data travels slower over long distances, like sending a letter across the country versus across the street.

---

## Proposed Solution: Cloudflare Edge Network

```
USER TYPES "hi"
      ↓
┌─────────────────────────────────────────────────────────┐
│  CLOUDFLARE WORKER (Edge - Mumbai)                      │
│  └─ Receives: { message: "hi", agentName: "Grace" }     │
└─────────────────────────────────────────────────────────┘
      ↓
      ↓ Check cache first
      ↓
  ┌───┴────┐
  │ Cache? │
  └───┬────┘
      │
   ┌──┴──────────────────────────────────────┐
   │                                         │
   ↓ YES (Cache Hit)                         ↓ NO (Cache Miss)
┌──────────────────┐              ┌───────────────────────────┐
│ Return cached    │              │ Call AI:                  │
│ response         │              │                           │
│ (< 10ms)         │              │ Try Option 1:             │
└──────────────────┘              │ ┌───────────────────────┐ │
                                  │ │ Cloudflare AI         │ │
                                  │ │ (Llama 3 at edge)     │ │
                                  │ │ ~100-150ms            │ │
                                  │ └───────────────────────┘ │
                                  │         ↓ Success?        │
                                  │         ↓ Yes → Return    │
                                  │         ↓ No → Fallback   │
                                  │                           │
                                  │ Try Option 2:             │
                                  │ ┌───────────────────────┐ │
                                  │ │ Cerebras API          │ │
                                  │ │ (from edge)           │ │
                                  │ │ ~400ms                │ │
                                  │ └───────────────────────┘ │
                                  └───────────────────────────┘
                                              ↓
                                  ┌───────────────────────────┐
                                  │ Cache response for future │
                                  └───────────────────────────┘
      ↓                                       ↓
      └───────────────┬───────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────┐
│  CLOUDFLARE WORKER (Edge - Mumbai)                      │
│  └─ Returns: { response: "Good morning!", mood: ... }   │
└─────────────────────────────────────────────────────────┘
                      ↓
                   USER SEES RESPONSE
```

---

### 2️⃣ Voice Message Flow 

```
USER STOPS SPEAKING
      ↓
┌─────────────────────────────────────────────────────────┐
│  CLOUDFLARE WORKER (Edge - Mumbai)                      │
│  └─ Receives: FormData with audio file                  │
└─────────────────────────────────────────────────────────┘
      ↓
      ↓ Transcription Step
      ↓
  ┌───┴────────────────────────────────────┐
  │                                        │
  ↓ Try Option 1:                          ↓ Option 1 Failed
┌───────────────────────┐      ┌────────────────────────────┐
│ Cloudflare AI Whisper │      │ Fireworks Whisper API      │
│ (at same edge)        │      │ (from edge, faster route)  │
│ ~120-150ms            │      │ ~400-500ms                 │
└───────────────────────┘      └────────────────────────────┘
          ↓                                 ↓
          └─────────────┬───────────────────┘
                        ↓
                 TRANSCRIPTION RESULT
                 "Hello Grace!"
                        ↓
┌─────────────────────────────────────────────────────────┐
│  CLOUDFLARE WORKER (Edge - Mumbai)                      │
│  └─ Now has text, proceeds to AI response               │
└─────────────────────────────────────────────────────────┘
                        ↓
                        ↓ SAME FLOW AS TEXT MESSAGE ABOVE
                        ↓ (Check cache → Try CF AI → Fallback)
                        ↓
┌─────────────────────────────────────────────────────────┐
│  CLOUDFLARE WORKER (Edge - Mumbai)                      │
│  └─ Returns: { text: "...", response: "..." }           │
└─────────────────────────────────────────────────────────┘
                        ↓
                   USER HEARS RESPONSE
```
