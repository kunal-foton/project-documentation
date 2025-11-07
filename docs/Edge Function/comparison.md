---
title: 💻 comparison
sidebar_position: 4 
---

# 💻 My Analysis

- **Cloudflare** is best for real-time AI and streaming.  
- **AWS Lambda@Edge:** 100–150ms cold start severely impacts UX for real-time apps.  
- **Cloudflare and Vercel** have a 128MB memory limit, while **AWS** and **Fastly Compute** are configurable.  
- **Fastly Compute:** 5–15ms latency — strong for streaming.  
- **Vercel Edge** is too expensive — almost 4× the price of Fastly Compute.  
- **Fastly Compute** is the best among all, with decent speed (a bit slower than Cloudflare), good memory capacity, and a fair price.  
- **AWS** provides a much better ecosystem when integrated with **S3**.  
- However, **Fastly Compute** doesn’t support Python yet — so we’d need to rewrite the whole pipeline in Rust if chosen.

---

# 🏅 My Rankings for Project Grace

## 🥇 Cloudflare Workers
- Matches our **language stack (TypeScript + JS)**  
- Natively supports **WebSockets** and **edge state**  
- Best balance between **speed**, **maintainability**, and **cost**

---

## 🥈 Fastly Compute@Edge
- **Wasm performance:** microsecond startup, deterministic isolation  
- Ranked second because we’d need to **rewrite critical parts in Rust (Wasm)**

---

## 🥉 AWS Lambda@Edge
- Provides the **best ecosystem** with AWS services  
- **Slower iteration** and deployment pipeline compared to Cloudflare  
- Cannot **host or maintain WebSocket sessions**


# ⚡ Edge Platform Comparison — Project Grace

| Platform              | Average Latency (Speed) | Global PoPs (Approx.) | Relative Cost 💰 | Notes |
|------------------------|------------------------:|----------------------:|----------------:|-------|
| **Cloudflare Workers** | **5–10 ms**            | **310+**             | 💲💲 (Moderate)  | Excellent real-time performance, huge PoP network |
| **Fastly Compute@Edge**| 10–15 ms               | ~85+                 | 💲 (Affordable) | Slightly slower but predictable latency; supports Wasm |
| **Vercel Edge Functions** | 20–30 ms           | ~35+                 | 💲💲💲 (Expensive) | Premium pricing; limited PoP reach |
| **AWS Lambda@Edge**    | 100–150 ms (cold start) | ~410+ (via CloudFront) | 💲💲 (Moderate) | Best ecosystem; slower cold starts affect UX |

---

✅ **Verdict:**  
For **Project Grace**, **Cloudflare Workers** offers the best combination of **speed, global coverage, and developer simplicity**, while **Fastly Compute@Edge** is a strong second if future Rust/Wasm migration is considered.
