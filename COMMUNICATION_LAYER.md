# Agent Communication Layer & Integration Architecture

Bidirectional communication bridge for Hermes agents across web, messaging, and webhook platforms.

## Overview

Your agents interface with external systems through three primary channels:

1. **Web Integration** — HTTP REST API + WebSocket real-time at `http://app.netintegrate.net`
2. **Telegram** — Bidirectional messaging via webhook for command input and result delivery
3. **Google Chat** — App integration + webhooks for team notifications and workflow coordination

## 1. Web Integration Layer (`app.netintegrate.net`)

### Purpose
- Real-time dashboard for agent status and metrics
- Manual task submission and workflow orchestration
- Visual demonstration of agent capabilities

### Architecture

```
┌─────────────────────────────────────┐
│  Browser (Client)                    │
│  - React Dashboard                   │
│  - Real-time WebSocket updates       │
│  - Agent status + results            │
└──────────────┬──────────────────────┘
               │
         HTTPS + WSS
               │
┌──────────────▼──────────────────────┐
│  nginx Reverse Proxy                 │
│  - SSL termination                   │
│  - Request routing                   │
│  - Rate limiting                     │
└──────────────┬──────────────────────┘
               │
        ┌──────┴──────┐
        │             │
    REST API      WebSocket
        │             │
┌───────▼────┐  ┌─────▼──────────┐
│ Express     │  │ Socket.io      │
│ Backend     │  │ Real-time feed │
│ /agents/*   │  │ /subscribe/*   │
└───────┬────┘  └─────┬──────────┘
        │             │
        └──────┬──────┘
               │
   ┌───────────▼────────────────┐
   │ Hermes Agent Coordinator   │
   │ - Task queue management    │
   │ - Result aggregation       │
   │ - Status broadcasting      │
   └───────────┬────────────────┘
               │
       ┌───────┴───────────┐
       │                   │
   Agent 1             Agent 2
   (SaaS Ops)          (VA Claims)
       │                   │
  Camofox             Browserbase
  [Browser Automation]
```

### REST API Endpoints

**Agent Status**:
```bash
GET /api/agents
# Returns: [{id, type, status, uptime, lastTask, memory_mb}]

GET /api/agents/{agentId}
# Returns: {id, type, status, currentTask, history: [...]}
```

**Submit Task**:
```bash
POST /api/agents/{agentId}/task
# Body: {
#   task: "extract_data",
#   params: {url, selector, action},
#   priority: "high",
#   timeout: 30000
# }
# Returns: {taskId, status, eta}
```

**Retrieve Results**:
```bash
GET /api/tasks/{taskId}
# Returns: {
#   id, agentId, status, result, 
#   startTime, endTime, memory_used_mb
# }
```

**Streaming Results**:
```javascript
// WebSocket subscription for live updates
const socket = io('wss://app.netintegrate.net');
socket.emit('subscribe', {agentId: 'agent-saas-1'});
socket.on('agentUpdate', (data) => {
  // {timestamp, status, memory_mb, lastTaskResult}
});
socket.on('taskComplete', (data) => {
  // {taskId, result, duration_ms}
});
```

### Dashboard Features

**Real-time Monitoring**:
- Agent uptime and health status
- Memory usage graphs (5-minute rolling)
- Task success rate and latency
- Anti-detection validation status

**Manual Task Submission**:
```
[Agent Selector] [Task Type] [Target URL] [Submit]
- Task types: extract, navigate, fill_form, screenshot
- Auto-validate: URL syntax, agent availability
- Result preview: screenshots, extracted text, structured data
```

**Webhook Log Viewer**:
- Recent Telegram/Google Chat communications
- Outbound webhook delivery status
- Two-way conversation history

---

## 2. Telegram Integration (`@hermes_agent_bot`)

### Purpose
- Low-friction task submission via chat
- Incident alerts and status notifications
- Multi-user access control via Telegram IDs

### Architecture

```
┌──────────────────┐
│ Telegram User    │
│ /task @url ...   │
└────────┬─────────┘
         │
    Telegram Bot API
         │
┌────────▼──────────────────┐
│ Webhook Receiver           │
│ POST /webhooks/telegram    │
│ Verify: bot_token + sign   │
└────────┬──────────────────┘
         │
┌────────▼──────────────────┐
│ Command Parser             │
│ - /task <params>           │
│ - /status <agent>          │
│ - /history <lines>         │
│ - /retry <taskId>          │
└────────┬──────────────────┘
         │
┌────────▼──────────────────┐
│ Hermes Coordinator         │
│ - Queue task               │
│ - Route to agent           │
│ - Store message ID         │
└────────┬──────────────────┘
         │
         ├─ Agent executes
         │
┌────────▼──────────────────┐
│ Result Handler             │
│ - Format result            │
│ - Generate screenshots     │
│ - Prepare summary          │
└────────┬──────────────────┘
         │
    Telegram Send API
         │
┌────────▼──────────────────┐
│ User receives:             │
│ - Inline result            │
│ - Screenshot (if visual)   │
│ - Extracted data (file)    │
└────────────────────────────┘
```

### Command Reference

**Task Submission**:
```
/task extract https://example.com h1
→ Extract all h1 elements from example.com
→ Returns: JSON file + screenshot

/task fill_form https://example.com username=test password=secret
→ Navigate + fill + submit form
→ Returns: screenshot of final state

/task navigate https://example.com
→ Simple navigation task
→ Returns: page title + content length

/task screenshot https://example.com
→ Capture page screenshot
→ Returns: image file (inline preview)
```

**Status Queries**:
```
/status
→ Returns all agents: ✅ SaaS Ops, ✅ VA Claims, 🔴 Real Estate
→ Shows uptime % + last task result

/status saas
→ Detailed status for SaaS agent
→ Memory, current task, task history (last 5)

/history 10
→ Last 10 tasks across all agents
→ Format: taskId | agent | input | result | duration
```

**Retry/Management**:
```
/retry <taskId>
→ Re-execute previous task

/cancel <taskId>
→ Cancel in-progress task

/logs <agent> <lines>
→ Last N log lines for agent (default 20)
```

### Telegram Security

**Authentication**:
```javascript
// Verify incoming webhook
const telegramSignature = req.headers['x-telegram-signature'];
const expectedSignature = crypto
  .createHmac('sha256', BOT_TOKEN)
  .update(JSON.stringify(req.body))
  .digest('hex');

if (telegramSignature !== expectedSignature) {
  return res.status(401).send('Invalid signature');
}
```

**Access Control**:
```javascript
const AUTHORIZED_USERS = [
  12345678,  // Your Telegram ID
  87654321,  // Team member
];

if (!AUTHORIZED_USERS.includes(message.from.id)) {
  return sendMessage('Unauthorized access');
}
```

**Rate Limiting**:
```javascript
const userLimits = new Map(); // Track per-user usage

function checkRateLimit(userId) {
  const now = Date.now();
  const lastRequest = userLimits.get(userId) || 0;
  
  if (now - lastRequest < 5000) {
    return false; // Block: < 5 seconds since last request
  }
  
  userLimits.set(userId, now);
  return true;
}
```

---

## 3. Google Chat Integration

### Purpose
- Team notifications and incident alerts
- Structured workflow approvals
- Integration with existing Google Workspace

### Architecture

**Option A: Google Chat App (Recommended)**
```
┌─────────────────────────────┐
│ Google Chat Space           │
│ @hermes_agent_app           │
│ /run_task <params>          │
└─────────────┬───────────────┘
              │
        Google Chat API
              │
┌─────────────▼───────────────┐
│ Chat App Webhook            │
│ POST /webhooks/google-chat   │
│ Verify: X-Goog-Chat-Signature
└─────────────┬───────────────┘
              │
┌─────────────▼───────────────┐
│ Message Parser              │
│ - Extract @mentions         │
│ - Parse slashcommands       │
│ - Thread context            │
└─────────────┬───────────────┘
              │
         Hermes Coordinator
              │
┌─────────────▼───────────────┐
│ Result Formatter            │
│ - Rich text card            │
│ - Inline images             │
│ - Action buttons            │
└─────────────┬───────────────┘
              │
        Google Chat API
              │
┌─────────────▼───────────────┐
│ Team sees:                  │
│ 📋 Rich card with result    │
│ 🖼️  Screenshots inline      │
│ ✅/❌ Action buttons        │
└─────────────────────────────┘
```

**Option B: Outbound Webhooks (Status Alerts)**
```
Hermes Coordinator
    │
    ├─ Task starts    → POST /webhook → "🔄 SaaS agent starting..."
    ├─ Task completes → POST /webhook → "✅ Result: {data}"
    └─ Task fails     → POST /webhook → "❌ Error: {reason}"
                                ↓
                        Google Chat Space
                        (Automatic thread updates)
```

### Google Chat Commands

**Rich Command Card**:
```javascript
// User sends message in space
const message = {
  text: 'Extract data from https://example.com',
  thread: 'spaces/AAAAAA/threads/123'
};

// App responds with interactive card
const response = {
  cards: [{
    header: {
      title: 'Extract Task',
      subtitle: 'example.com'
    },
    sections: [{
      widgets: [{
        textParagraph: {
          text: 'Selector: .data-item | Status: PENDING'
        }
      }]
    }],
    cardActions: [{
      actionLabel: 'View Result',
      onClick: { openLink: { url: 'http://app.netintegrate.net/tasks/123' } }
    }]
  }]
};
```

**Approval Workflow**:
```
1. Agent encounters decision point
   → Sends approval card to Google Chat

2. Team member clicks "Approve" button
   → Webhook triggers approval handler

3. Coordinator resumes agent execution
   → Completes task and reports result

Example:
┌─────────────────────────────┐
│ ❓ Should I submit this form?│
│ - Content looks valid       │
│ - All fields filled         │
│                             │
│ [Approve] [Review] [Cancel] │
└─────────────────────────────┘
```

### Google Chat Security

**Verify Signature**:
```javascript
const crypto = require('crypto');

function verifyGoogleChatSignature(req) {
  const signature = req.headers['x-goog-chat-signature'];
  const body = req.rawBody; // Must be original body
  
  const hmac = crypto
    .createHmac('sha256', GOOGLE_CHAT_SECRET)
    .update(body)
    .digest('base64');
  
  return signature === hmac;
}
```

---

## 4. Webhook Communication Pattern

### Generic Webhook Handler

```javascript
// All webhooks follow this pattern
const express = require('express');
const router = express.Router();

router.post('/webhooks/:platform', async (req, res) => {
  // 1. Verify signature
  if (!verifySignature(req)) {
    return res.status(401).send('Invalid signature');
  }

  // 2. Parse message
  const message = parseMessage(req.body, req.params.platform);
  
  // 3. Extract intent
  const {action, params} = extractIntent(message);
  
  // 4. Queue task
  const taskId = await coordinatorQueue.enqueue({
    action,
    params,
    source: req.params.platform,
    sourceId: message.userId || message.chatId,
    threadId: message.threadId
  });
  
  // 5. Send immediate ACK
  res.status(202).json({taskId});
  
  // 6. Execute async (don't block response)
  executeTask(taskId)
    .then(result => deliverResult(result, req.params.platform, message))
    .catch(err => deliverError(err, req.params.platform, message));
});
```

### Task Result Delivery

**Telegram** (Reply to message):
```javascript
async function deliverResultTelegram(result, userId) {
  if (result.screenshot) {
    // Send photo with caption
    await bot.sendPhoto(userId, result.screenshot, {
      caption: `✅ Task ${result.taskId} complete\n${result.summary}`
    });
  }
  
  if (result.data) {
    // Send structured data as file
    await bot.sendDocument(userId, Buffer.from(JSON.stringify(result.data)), {
      filename: `task-${result.taskId}-results.json`
    });
  }
}
```

**Google Chat** (Update thread):
```javascript
async function deliverResultGoogleChat(result, spaceId, threadId) {
  const card = buildResultCard(result);
  
  await chat.spaces.messages.create({
    parent: `spaces/${spaceId}`,
    messageReplyOption: 'REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD',
    threadReplyOptions: {threadId},
    body: {
      cards: [card]
    }
  });
}
```

**Web** (Real-time WebSocket):
```javascript
// Hermes coordinator broadcasts to all subscribed clients
io.to(`agent-${result.agentId}`).emit('taskComplete', {
  taskId: result.taskId,
  status: 'success',
  duration: result.duration,
  result: result.data
});
```

---

## 5. End-to-End Task Flow Example

### Scenario: SaaS Operations Agent extracts customer list

```
Timeline:
─────────────────────────────────────────────

T+0s: User sends Telegram message
  @hermes_agent_bot
  /task extract https://dashboard.acme.com/customers .customer-name

T+1s: Webhook received → Signature verified → Task queued
  taskId: 5e4b8c9a
  Status: PENDING
  ↓
  Telegram: "⏳ Task queued. Checking agent availability..."

T+2s: Coordinator routes to SaaS agent
  Agent: SaaS Operations
  Status: IDLE
  ↓
  Agent accepts task → Begins execution

T+3-8s: Camofox executes
  - Browser: Open https://dashboard.acme.com/customers
  - Auth: Use stored session cookies
  - Extraction: Query .customer-name elements
  - Screenshot: Capture page for verification

T+8s: Task completes
  Result: {
    taskId: 5e4b8c9a,
    agentId: saas-1,
    status: success,
    extracted: [...],
    screenshot: [base64_image],
    duration: 6000
  }

T+9s: Results delivered to all channels
  
  Telegram:
  ┌─────────────────────────┐
  │ ✅ Task 5e4b8c9a        │
  │ Extracted 247 customers │
  │ [View in dashboard]     │
  │ [photo_of_page.jpg]     │
  └─────────────────────────┘
  + results.json file attachment
  
  Google Chat:
  ┌─────────────────────────┐
  │ ✅ Customer extraction  │
  │ 247 records found       │
  │ Duration: 6.2 seconds   │
  │ [View Full Results]     │
  └─────────────────────────┘
  
  Web Dashboard:
  Real-time update in task history
  Screenshot appears inline
  Download link generated

─────────────────────────────────────────────
Total latency: ~9 seconds (2 seconds overhead + 6 seconds execution + 1 second delivery)
```

---

## 6. Error Handling & Fallback

### Communication Fallback Chain

```
Primary delivery channel fails
         ↓
Attempt secondary channel
         ↓
If both fail: Store in error queue
         ↓
Retry with exponential backoff:
  - 30 seconds
  - 5 minutes
  - 30 minutes
  - 24 hours
         ↓
Final fallback: Email + web alert
```

### Example: Telegram delivery fails

```javascript
async function deliverResultWithFallback(result, channels) {
  const outcomes = {};
  
  for (const channel of channels) {
    try {
      await deliverResult(result, channel);
      outcomes[channel] = 'success';
    } catch (err) {
      outcomes[channel] = {status: 'failed', error: err.message};
      
      // Fallback to web dashboard
      if (channel === 'telegram') {
        notifyAdminViaDashboard({
          message: `Telegram delivery failed: ${err.message}`,
          taskId: result.taskId,
          userAction: 'Check web dashboard'
        });
      }
    }
  }
  
  // Log delivery attempt
  await deliveryLog.insert({
    taskId: result.taskId,
    timestamp: Date.now(),
    channels: outcomes
  });
}
```

---

## 7. Production Deployment Checklist

- [ ] **Telegram Bot** configured with webhook URL
  ```bash
  curl "https://api.telegram.org/bot<TOKEN>/setWebhook?url=https://app.netintegrate.net/webhooks/telegram"
  ```

- [ ] **Google Chat App** created and installed in workspace
  - [ ] Webhook URL configured
  - [ ] Permissions scoped correctly
  - [ ] Test message sent and received

- [ ] **Web Dashboard** accessible and websocket working
  ```bash
  # Test real-time connection
  wscat -c wss://app.netintegrate.net/socket.io
  ```

- [ ] **Rate limiting** configured per platform
  - [ ] Telegram: 5 requests / 5 seconds per user
  - [ ] Google Chat: 10 requests / minute per space
  - [ ] Web API: 100 requests / minute per IP

- [ ] **Error notifications** configured
  - [ ] Failed tasks alert via all channels
  - [ ] Agent crashes notify admin immediately
  - [ ] Webhook timeouts logged and retried

- [ ] **Audit logging** enabled
  ```javascript
  // Log all communication
  {
    timestamp, channel, userId, action,
    params, result, latency, outcome
  }
  ```

- [ ] **Secrets management**
  - [ ] Bot tokens in environment variables (not code)
  - [ ] Webhook secrets rotated monthly
  - [ ] Signatures verified on all inbound messages

---

## 8. Future Enhancements

**Planned Additions**:
- **Slack Integration** — Same pattern as Google Chat
- **Discord** — For team notifications
- **Email** — Long-form results and batch reports
- **Mobile App** — Native iOS/Android for on-the-go task submission
- **Voice Commands** — "Hey Hermes, extract customer data"

**Advanced Features**:
- **Scheduled Tasks** — "Run this task every day at 9 AM"
- **Conditional Workflows** — "If task A succeeds, run task B"
- **Human-in-the-loop** — Approval gates with time-based escalation
- **Cost Tracking** — Per-task cost visualization in dashboard

---

## Monitoring & Observability

### Webhook Metrics to Track

```javascript
{
  inbound: {
    requests_per_minute: 45,
    avg_response_time_ms: 120,
    error_rate_percent: 0.2,
    by_platform: {
      telegram: 30,
      google_chat: 10,
      web_api: 5
    }
  },
  outbound: {
    deliveries_per_minute: 42,
    success_rate_percent: 99.8,
    avg_delivery_time_ms: 450,
    by_platform: {
      telegram: 28,
      google_chat: 9,
      web: 5
    }
  },
  errors: {
    timeouts: 1,
    auth_failures: 0,
    network_errors: 1,
    coordinator_queue_depth: 3
  }
}
```

Track these metrics in your ClickHouse telemetry system alongside Camofox and agent performance data.
