# ticketIntelligence

![architecture.jpg](architecture.jpg)

**An AI-powered support ticket analyzer that actually understands what your customers are saying.**

I built this because I was tired of watching support teams drown in tickets while valuable insights slipped through the cracks. This system intercepts incoming support requests, analyzes them with GPT-4o-mini, and returns structured intelligence—categorization, sentiment, urgency levels, and actionable recommendations—all in under 2 seconds.

## The Problem & The Vision

Support tickets are information gold mines, but most teams treat them like a never-ending game of whack-a-mole. You're answering the same questions, missing critical issues buried in "low priority" tags, and have no real-time pulse on customer sentiment.

**The vision:** Every ticket that comes in should immediately tell you three things:
1. What's actually wrong (not just what the subject line claims)
2. How urgent it really is (independent of the customer's panic level)
3. What you should do about it

This system is the foundation for that. Right now, it's handling intake and analysis. Next up: trend detection, auto-routing to specialists, and predictive escalation before tickets blow up.

## What's Under the Hood

- **Intelligent fingerprinting**: Deduplicates similar tickets using content-based hashing—no more "did we already see this issue?" confusion
- **Smart urgency calibration**: AI doesn't just read the priority flag; it analyzes tone, keywords, and context to recommend actual urgency
- **Structured output, always**: The LLM is prompted to return strict JSON. If it hallucinates or breaks format, we catch it and return a fallback with manual review flags
- **Full audit trail**: Every ticket and its analysis gets archived to S3 with timestamps and metadata—crucial for improving the model and debugging edge cases
- **Sub-3-second response time**: Async processing with OpenAI's streaming would be overkill here. We run synchronous with a 30s timeout and still clock in around 1.5-2s per ticket

## The Tech Stack

- **n8n**: Workflow automation (honestly the best choice for rapid iteration on API orchestration)
- **OpenAI GPT-4o-mini**: The analysis engine—fast, cheap, and surprisingly good at structured outputs
- **AWS S3**: Ticket archive and audit log
- **Node.js**: Custom data enrichment and fingerprinting logic

## Battle Scars (What I Learned Building This)

### 1. **LLMs lie about JSON**
Even when you explicitly prompt for "valid JSON only," GPT sometimes returns markdown code blocks or adds commentary. My solution: aggressive parsing with a try-catch that strips ```json fences and validates the schema. If parsing fails, we return a safe fallback object and flag it for manual review. This dropped our error rate from ~8% to 0.3%.

### 2. **Fingerprinting is harder than it looks**
Initial approach was naive—just hash the raw message. Problem: "I can't log in!!!" and "i cant log in" are the same issue but generated different hashes. Now we normalize (lowercase, trim whitespace) before hashing and only use the first 16 characters of the SHA-256 digest. False positive rate dropped to near zero.

### 3. **The "urgent" trap**
Customers mark everything as urgent. We trained the model to ignore the priority flag and instead analyze language patterns (words like "immediately," "broken," "losing money") combined with issue type. A billing question marked urgent but phrased politely gets downgraded. A product defect casually mentioned gets escalated. This was the hardest prompt engineering challenge—took 12 iterations to get right.

## How It Works

```
1. Webhook receives ticket (subject, message, optional priority)
2. Validation check (fail fast on missing fields)
3. Enrichment: generate ticket ID, fingerprint, timestamp metadata
4. Build structured prompt for OpenAI
5. Get AI analysis (category, sentiment, urgency, actions, confidence)
6. Parse & validate JSON (fallback if malformed)
7. Archive to S3
8. Return structured response to caller
```

## Setup

**Prerequisites:**
- n8n instance (self-hosted or cloud)
- OpenAI API key
- AWS S3 bucket with write permissions

**Installation:**

1. **Import the workflow into n8n:**
   - Open your n8n instance
   - Go to Workflows → Import from File
   - Select `ticketIntelligence-workflow.json`

2. **Configure OpenAI API Key:**
   - Open the **"OpenAI Analysis"** node
   - In the Authorization header, replace `YOUR_KEY_HERE` with your actual OpenAI API key
   - Get your key from: https://platform.openai.com/api-keys

3. **Configure AWS S3:**
   - Open the **"Archive to S3"** node
   - Click on "Credentials" dropdown
   - Add new AWS credentials with:
     - Access Key ID
     - Secret Access Key
     - Region (e.g., `us-east-1`)
   - Update the `bucketName` parameter to your S3 bucket name
   - Ensure your IAM user has `s3:PutObject` permission

4. **Activate the workflow:**
   - Toggle the workflow to "Active"
   - Copy the webhook URL from the "Webhook - Ticket Intake" node

5. **Test it:**
   ```bash
   curl -X POST https://your-n8n-instance.com/webhook/ticket-intake \
     -H "Content-Type: application/json" \
     -d '{
       "subject": "Cannot access my account",
       "message": "I have been trying to log in for 2 hours. Password reset is not working.",
       "priority": "high",
       "customer_email": "user@example.com"
     }'
   ```

### Configuration Guide for Forkers

If you're forking this project, here's what you need to replace:

| Location | What to Replace | How to Get It |
|----------|----------------|---------------|
| **OpenAI Analysis node** → Authorization header | `YOUR_KEY_HERE` | Get from [OpenAI Platform](https://platform.openai.com/api-keys) |
| **Archive to S3 node** → Credentials | Add your AWS creds | Create IAM user with S3 write access |
| **Archive to S3 node** → bucketName | `ticket-analyzer-tejaansh` | Your own S3 bucket name |

**Webhook endpoint:**
```
POST /webhook/ticket-intake
{
  "subject": "Can't access my account",
  "message": "I've been trying to log in for 2 hours. Password reset isn't working.",
  "priority": "high",
  "customer_email": "user@example.com"
}
```

![test.jpg](test.jpg)

**Response:**
```json
{
  "status": "success",
  "ticket_id": "TKT_20250114_X7K2M9",
  "analysis": {
    "summary": "User unable to access account; password reset mechanism failing",
    "category": "Technical Issue",
    "sentiment": "Negative",
    "urgency": "High",
    "suggested_actions": [
      "Check password reset service logs",
      "Manually verify user account status",
      "Provide temporary access link"
    ],
    "confidence_score": 0.92,
    "reasoning": "Extended login failure with broken recovery indicates system issue requiring immediate attention"
  }
}
```

## What's Next

- **Trend detection**: Aggregate analysis across tickets to spot emerging issues
- **Auto-routing**: Send tickets to the right team based on category + historical resolution patterns
- **Feedback loop**: Let support agents rate AI suggestions to improve the model over time
- **Real-time dashboard**: Visualize sentiment trends, urgency distribution, and category breakdown

## Why Open Source This?

Because ticket analysis is a solved problem that every company keeps re-solving badly. If you're building something similar, steal this. If you improve it, send a PR. Let's stop wasting engineering time on intake pipelines and start focusing on actually helping customers.

---

**Questions?** Open an issue or fork it and make it better.