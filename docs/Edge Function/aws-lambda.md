---
title: 💪 AWS Lambda@Edge
sidebar_position: 0 
---

# 💪 AWS Lambda@Edge

This works across Amazon’s data centers worldwide.  
AWS Lambda@Edge extends AWS Lambda to **CloudFront's edge network**, enabling code execution at AWS PoPs globally.

---

## ⚙️ Technical Insights

Unlike other serverless platforms that use containers or VMs, this uses **V8 isolates** for execution.

### Characteristics:
- Best suited for AWS ecosystem integration  
- Uses **AWS Firecracker microVMs** for isolation (lightweight, secure)  
- Slower than Cloudflare  

💰 **Pricing:** ~$0.6 per million requests

---

## ❌ Cons
- Slower than other providers  
- Fewer PoPs than Cloudflare (limited regional AWS edges)
