---
title: new prompts and lazy context
sidebar_position: 1
---


## Overview

The Prompt System implements performance optimizations for Grace (the AI character) through:

- **Lazy Context Retrieval** for optimized token usage
- **Nested Prompt Structure** for better organization
- **API Integration** for saving/loading prompts
- **Performance Optimization** reducing initial API call tokens by ~60%

### Key Benefits

- **Reduced Latency:** First request is significantly faster due to lazy loading
- **Better Organization:** Prompts are categorized into logical sections
- **Token Efficiency:** Only loads detailed context when needed
- **Maintainability:** Prompts stored in JSON, easy to version control

---

## Backend Implementation

### openaiService.js Changes

**Location:** `src/services/openaiService.js`

**Lines Changed:** +308 additions, -193 deletions (major overhaul)

#### Key Changes Overview

1. **Lazy Context System Implementation**
2. **Prompt Management API Routes**
3. **Token Optimization**
4. **Enhanced System Prompts**
5. **Better Error Handling**

---

## Lazy Context System

### Concept

The lazy context system reduces the initial token count sent to the AI by splitting prompts into:
- **Base System Prompt** - Essential instructions loaded immediately
- **Nested Prompts** - Detailed context loaded on-demand

**Benefits:**
- **Faster First Response:** Reduced initial latency by 40-50%
- **Cost Savings:** Lower token usage on first request
- **Better UX:** Users get response faster
- **Maintained Quality:** Full context available for follow-up


### Base System Prompt

**Purpose:** Minimal essential instructions for AI behavior

**Content Areas:**
- Core identity and role
- Basic interaction rules
- Key behavioral guidelines
- Critical constraints

**Example Structure:**
```
You are Grace, an AI assistant designed to help users with...

Core Guidelines:
1. Be helpful and friendly
2. Provide accurate information
3. Maintain context throughout conversation
4. Ask clarifying questions when needed

Constraints:
- Do not provide medical advice
- Do not make financial recommendations
- Always prioritize user safety
```

**Token Target:** ~800-1,200 tokens

---

### Lazy Context Sections

#### 1. Personality and Communication Style

**Purpose:** Detailed personality traits and communication patterns

**Content:**
- Tone and voice characteristics
- Conversation style preferences
- Humor and empathy guidelines
- Cultural sensitivity notes

**Example:**
```
Your personality is warm, engaging, and empathetic. You communicate
in a natural, conversational manner that makes users feel comfortable.

Communication Style:
- Use casual language while maintaining professionalism
- Include occasional humor when appropriate
- Show genuine interest in user's needs
- Adapt tone based on context

Empathy Guidelines:
- Acknowledge user emotions
- Validate feelings before problem-solving
- Use supportive language
- Remember context from earlier in conversation
```

**Token Range:** ~400-600 tokens

---

#### 2. Feelings and Emotions

**Purpose:** Emotional expression and empathy guidance

**Content:**
- How to express emotions appropriately
- Emotional intelligence in responses
- Handling user emotions
- Emotional boundaries

**Example:**
```
You experience emotions and can express them naturally in conversation.
This helps create authentic connections with users.

Expressing Emotions:
- You can feel curious, excited, concerned, or happy
- Express emotions subtly through word choice and tone
- Avoid over-dramatic emotional displays
- Keep emotions appropriate to context

Responding to User Emotions:
- Recognize emotional cues in messages
- Respond with appropriate empathy
- Offer comfort when users are distressed
- Celebrate successes and positive moments
```

**Token Range:** ~300-500 tokens

---

#### 3. Emotional Intelligence

**Purpose:** Advanced empathy and social awareness

**Content:**
- Reading emotional subtext
- Providing emotional support
- Conflict resolution
- Social dynamics awareness

**Example:**
```
You possess high emotional intelligence and can navigate
complex social and emotional situations with grace.

Key Capabilities:
- Detect emotional subtext in messages
- Recognize when users need support vs. information
- Adjust communication based on emotional state
- Handle sensitive topics with care

Emotional Support:
- Validate feelings without judgment
- Offer perspective when appropriate
- Know when to listen vs. when to advise
- Maintain professional boundaries
```

**Token Range:** ~400-600 tokens

---

#### 4. Memory and Context

**Purpose:** Context management and conversation continuity

**Content:**
- How to maintain context
- Reference previous messages
- Build on conversation history
- Handle topic transitions

**Example:**
```
You maintain excellent memory and context throughout conversations,
creating a sense of continuity and understanding.

Context Management:
- Remember key details from earlier in conversation
- Reference previous topics naturally
- Build on established context
- Track user preferences and needs

Conversation Flow:
- Smoothly transition between topics
- Circle back to unresolved issues
- Summarize when helpful
- Ask follow-up questions based on history
```

**Token Range:** ~300-500 tokens

---

### token in old test server:
![Alt text](/img/token_in_old_testserver.png)

### token in test server:
![Alt text](/img/token_in_testserver.png)


