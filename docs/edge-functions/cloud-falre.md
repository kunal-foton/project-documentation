---
title: ☁️ Cloudflare Workers
sidebar_position: 1
---


# ☁️ Cloudflare Workers

This is a very fast, lightweight JS/TypeScript function running on Cloudflare’s edge locations.  
They execute code using **V8 isolates (the same engine Chrome uses)**.

---

## ⚙️ Technical Insights

Unlike other serverless platforms that use containers and VMs, this uses **V8 isolates** for execution.

### Characteristics:
- 🚀 Faster startup  
- 💾 Better memory efficiency  
- ⚡ Thousands of isolates run in a single process, enabling massive scale  
- 💰 **Pricing:** ~$0.3 per million requests (cheapest)

---

## ❌ Cons
- **Limited Execution Environment:** limited memory (**128 MB**) per isolate  
- **No GPU/TPU access**, so ML workloads must run elsewhere
