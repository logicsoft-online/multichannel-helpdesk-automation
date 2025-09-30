# n8n Workflow Diagrams and Implementation Guide

## 1. Message Ingestion Workflow

### Overview
This workflow handles incoming messages from all channels and stores them in the database.

### Flow Diagram
```
Channel Webhook
      ↓
Message Parser Node
      ↓
Channel Type Router
      ↓
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│   WhatsApp      │     Email       │   Facebook      │    Web Chat     │
│   Parser        │     Parser      │   Parser        │    Parser       │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
      ↓
Message Normalizer
      ↓
Database Storage (PostgreSQL)
      ↓
Trigger AI Processing
```

### n8n Node Configuration

#### Webhook Trigger
```json
{
  "name": "Channel Webhook",
  "type": "n8n-nodes-base.webhook",
  "parameters": {
    "path": "webhook/{{$json.channel}}",
    "httpMethod": "POST",
    "responseMode": "responseNode"
  }
}
```

#### Message Parser
```json
{
  "name": "Message Parser",
  "type": "n8n-nodes-base.function",
  "parameters": {
    "functionCode": "// Parse incoming message based on channel type\nconst channel = $input.first().json.channel;\nconst data = $input.first().json.data;\n\nlet parsedMessage = {};\n\nswitch(channel) {\n  case 'whatsapp':\n    parsedMessage = {\n      channel_id: data.from,\n      content: data.text.body,\n      sender_type: 'customer',\n      external_message_id: data.id,\n      metadata: {\n        phone: data.from,\n        timestamp: data.timestamp\n      }\n    };\n    break;\n    \n  case 'email':\n    parsedMessage = {\n      channel_id: data.from,\n      content: data.subject + '\\n\\n' + data.body,\n      sender_type: 'customer',\n      external_message_id: data.messageId,\n      metadata: {\n        email: data.from,\n        subject: data.subject,\n        timestamp: data.date\n      }\n    };\n    break;\n    \n  case 'facebook':\n    parsedMessage = {\n      channel_id: data.sender.id,\n      content: data.message.text,\n      sender_type: 'customer',\n      external_message_id: data.message.mid,\n      metadata: {\n        facebook_id: data.sender.id,\n        timestamp: data.timestamp\n      }\n    };\n    break;\n}\n\nreturn [{ json: parsedMessage }];"
  }
}
```

## 2. AI Response Generation Workflow

### Overview
This workflow processes messages through the RAG pipeline and generates AI responses.

### Flow Diagram
```
New Message Trigger
      ↓
Get Conversation Context
      ↓
Generate Query Embedding
      ↓
Vector Similarity Search (pgvector)
      ↓
Retrieve Relevant Knowledge
      ↓
Build Context for AI
      ↓
OpenAI GPT-4 Generation
      ↓
Response Validation
      ↓
Send Response to Channel
```

### n8n Node Configuration

#### RAG Query Node
```json
{
  "name": "RAG Query",
  "type": "n8n-nodes-base.function",
  "parameters": {
    "functionCode": "// Generate embedding and search knowledge base\nconst message = $input.first().json.content;\nconst conversationId = $input.first().json.conversation_id;\n\n// Call OpenAI embedding API\nconst embeddingResponse = await $http.request({\n  method: 'POST',\n  url: 'https://api.openai.com/v1/embeddings',\n  headers: {\n    'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,\n    'Content-Type': 'application/json'\n  },\n  body: {\n    input: message,\n    model: 'text-embedding-ada-002'\n  }\n});\n\nconst embedding = embeddingResponse.data.data[0].embedding;\n\n// Search knowledge base using pgvector\nconst searchQuery = `\n  SELECT id, title, content, \n         embedding <=> $1 as distance\n  FROM knowledge_base \n  WHERE is_active = true\n  ORDER BY embedding <=> $1\n  LIMIT 5\n`;\n\nconst dbResponse = await $pg.query(searchQuery, [JSON.stringify(embedding)]);\n\nreturn [{\n  json: {\n    message: message,\n    conversation_id: conversationId,\n    relevant_knowledge: dbResponse.rows,\n    embedding: embedding\n  }\n}];"
  }
}
```

#### AI Response Generation
```json
{
  "name": "AI Response Generation",
  "type": "n8n-nodes-base.openAi",
  "parameters": {
    "resource": "chat",
    "operation": "create",
    "model": "gpt-4",
    "messages": {
      "values": [
        {
          "role": "system",
          "content": "You are a helpful customer support assistant. Use the provided knowledge base to answer questions accurately and professionally."
        },
        {
          "role": "user",
          "content": "{{$json.message}}\n\nRelevant information:\n{{$json.relevant_knowledge.map(k => k.title + ': ' + k.content).join('\\n\\n')}}"
        }
      ]
    },
    "temperature": 0.7,
    "maxTokens": 500
  }
}
```

## 3. Escalation Workflow

### Overview
This workflow detects complex queries that require human intervention and escalates them to agents.

### Flow Diagram
```
Message Analysis
      ↓
Complexity Detection
      ↓
┌─────────────────┬─────────────────┐
│   Simple Query  │  Complex Query  │
│   (Auto Reply)  │   (Escalate)    │
└─────────────────┴─────────────────┘
      ↓                    ↓
  AI Response         Agent Assignment
      ↓                    ↓
  Send Response        Send Notification
```

### n8n Node Configuration

#### Complexity Detection
```json
{
  "name": "Complexity Detection",
  "type": "n8n-nodes-base.function",
  "parameters": {
    "functionCode": "// Analyze message complexity\nconst message = $input.first().json.content;\nconst knowledgeResults = $input.first().json.relevant_knowledge;\n\n// Simple heuristics for complexity detection\nconst complexityIndicators = [\n  'urgent', 'emergency', 'complaint', 'refund', 'cancel',\n  'escalate', 'manager', 'supervisor', 'billing', 'payment'\n];\n\nconst hasComplexityIndicators = complexityIndicators.some(indicator => \n  message.toLowerCase().includes(indicator)\n);\n\n// Check if knowledge base has relevant information\nconst hasRelevantKnowledge = knowledgeResults.some(k => k.distance < 0.3);\n\n// Check message length (longer messages might be more complex)\nconst isLongMessage = message.length > 200;\n\n// Check for multiple questions\nconst questionCount = (message.match(/\\?/g) || []).length;\nconst hasMultipleQuestions = questionCount > 1;\n\nconst shouldEscalate = hasComplexityIndicators || \n                      !hasRelevantKnowledge || \n                      isLongMessage || \n                      hasMultipleQuestions;\n\nreturn [{\n  json: {\n    ...$input.first().json,\n    should_escalate: shouldEscalate,\n    complexity_score: {\n      has_indicators: hasComplexityIndicators,\n      has_knowledge: hasRelevantKnowledge,\n      is_long: isLongMessage,\n      multiple_questions: hasMultipleQuestions\n    }\n  }\n}];"
  }
}
```

## 4. Knowledge Base Management Workflow

### Overview
This workflow handles knowledge base updates and embedding generation.

### Flow Diagram
```
Document Upload
      ↓
Text Extraction
      ↓
Content Chunking
      ↓
Generate Embeddings
      ↓
Store in pgvector
      ↓
Update Search Index
```

### n8n Node Configuration

#### Document Processing
```json
{
  "name": "Document Processing",
  "type": "n8n-nodes-base.function",
  "parameters": {
    "functionCode": "// Process uploaded document\nconst document = $input.first().json;\n\n// Extract text based on file type\nlet extractedText = '';\n\nif (document.fileType === 'pdf') {\n  // Use PDF parsing library\n  extractedText = await extractPDFText(document.filePath);\n} else if (document.fileType === 'docx') {\n  // Use DOCX parsing library\n  extractedText = await extractDOCXText(document.filePath);\n} else if (document.fileType === 'txt') {\n  extractedText = document.content;\n}\n\n// Split into chunks\nconst chunkSize = 1000;\nconst chunks = [];\n\nfor (let i = 0; i < extractedText.length; i += chunkSize) {\n  chunks.push({\n    content: extractedText.slice(i, i + chunkSize),\n    chunk_index: Math.floor(i / chunkSize),\n    total_chunks: Math.ceil(extractedText.length / chunkSize)\n  });\n}\n\nreturn chunks.map(chunk => ({\n  json: {\n    ...document,\n    ...chunk,\n    processed_text: chunk.content\n  }\n}));"
  }
}
```

## 5. Analytics and Reporting Workflow

### Overview
This workflow generates analytics and reports for the dashboard.

### Flow Diagram
```
Scheduled Trigger (Daily)
      ↓
Collect Metrics
      ↓
Process Data
      ↓
Generate Reports
      ↓
Update Dashboard
```

### n8n Node Configuration

#### Metrics Collection
```json
{
  "name": "Metrics Collection",
  "type": "n8n-nodes-base.function",
  "parameters": {
    "functionCode": "// Collect daily metrics\nconst today = new Date().toISOString().split('T')[0];\n\n// Query database for metrics\nconst metrics = await $pg.query(`\n  SELECT \n    COUNT(*) as total_messages,\n    COUNT(CASE WHEN sender_type = 'customer' THEN 1 END) as customer_messages,\n    COUNT(CASE WHEN is_auto_response = true THEN 1 END) as auto_responses,\n    COUNT(CASE WHEN created_at >= $1 THEN 1 END) as today_messages,\n    AVG(CASE WHEN is_auto_response = true THEN 1 ELSE 0 END) as auto_response_rate\n  FROM messages \n  WHERE created_at >= $1\n`, [today]);\n\n// Get channel breakdown\nconst channelMetrics = await $pg.query(`\n  SELECT \n    c.name as channel_name,\n    COUNT(m.id) as message_count\n  FROM channels c\n  LEFT JOIN messages m ON c.id = m.channel_id\n  WHERE m.created_at >= $1\n  GROUP BY c.id, c.name\n`, [today]);\n\n// Get conversation metrics\nconst conversationMetrics = await $pg.query(`\n  SELECT \n    status,\n    COUNT(*) as count\n  FROM conversations\n  WHERE created_at >= $1\n  GROUP BY status\n`, [today]);\n\nreturn [{\n  json: {\n    date: today,\n    metrics: metrics.rows[0],\n    channel_breakdown: channelMetrics.rows,\n    conversation_status: conversationMetrics.rows\n  }\n}];"
  }
}
```

## 6. Real-time Dashboard Updates

### Overview
This workflow provides real-time updates to the dashboard using WebSockets.

### Flow Diagram
```
Message Event
      ↓
Process Event
      ↓
Update Cache (Redis)
      ↓
Broadcast to Dashboard
```

### n8n Node Configuration

#### Real-time Updates
```json
{
  "name": "Real-time Updates",
  "type": "n8n-nodes-base.function",
  "parameters": {
    "functionCode": "// Process real-time events\nconst event = $input.first().json;\n\n// Update Redis cache\nawait $redis.setex(`conversation:${event.conversation_id}`, 3600, JSON.stringify(event));\n\n// Broadcast to connected clients\nconst io = require('socket.io')(process.env.SOCKET_PORT);\nio.emit('message_update', {\n  type: 'new_message',\n  conversation_id: event.conversation_id,\n  message: event,\n  timestamp: new Date().toISOString()\n});\n\nreturn [{ json: { status: 'broadcasted', event: event } }];"
  }
}
```

## Implementation Checklist

### Phase 1: Core Setup
- [ ] Set up PostgreSQL with pgvector extension
- [ ] Configure n8n instance
- [ ] Create database schema
- [ ] Set up Redis for caching

### Phase 2: Channel Integrations
- [ ] WhatsApp Business API integration
- [ ] Email (SMTP/IMAP) integration
- [ ] Facebook Messenger integration
- [ ] Web chat widget

### Phase 3: AI and RAG
- [ ] OpenAI API integration
- [ ] Embedding generation pipeline
- [ ] Vector similarity search
- [ ] Knowledge base management

### Phase 4: Workflows
- [ ] Message ingestion workflow
- [ ] AI response workflow
- [ ] Escalation workflow
- [ ] Analytics workflow

### Phase 5: Dashboard
- [ ] React frontend setup
- [ ] Real-time updates
- [ ] Analytics visualization
- [ ] Configuration interface

### Phase 6: Testing and Deployment
- [ ] Unit testing
- [ ] Integration testing
- [ ] Load testing
- [ ] Production deployment

---

## 6. WhatsApp Ingestion Workflow (POC, updated)

### Overview
WhatsApp inbound messages are normalized, persisted, classified, and either auto-answered (when confident) or escalated by creating a Helpdesk ticket. A reference ID is returned to the user when a ticket is created.

### Flow Diagram
```
WhatsApp Webhook
      ↓
Normalize Payload (phone, body, msg_id)
      ↓
Persist → conversations/messages
      ↓
Classifier (rules + LLM)
      ↓
┌──────────────────────┬───────────────────────────┐
│  Confident (≥ auto)  │  Not Confident / Out-of-scope │
└──────────────────────┴───────────────────────────┘
      ↓                              ↓
RAG → AI → WhatsApp Reply     Helpdesk: Create Ticket
      ↓                              ↓
Log ai_decisions                Store ticket_id + reference_id → WhatsApp Ack
```

### Node Hints
- Webhook (Meta WhatsApp Cloud)
- Function: normalize payload
- PostgreSQL: upsert conversation, insert message
- Function/IF: classifier and thresholding
- OpenAI (chat) + pgvector query (SQL)
- HTTP Request: Helpdesk ticket create
- HTTP Request: WhatsApp send message
- PostgreSQL: insert into `ai_decisions`

---

## 7. Helpdesk Ticket Ingestion Workflow (direct tickets)

### Overview
Ingest tickets created directly in the Helpdesk. If eligible and AI confidence ≥ threshold, post an AI reply via Helpdesk API; otherwise, add a draft/internal note.

### Flow Diagram
```
Helpdesk Poll/Webhook
      ↓
Eligibility Filter (status=open, unassigned, allowed queues)
      ↓
Build Context (ticket + last N notes + KB)
      ↓
RAG → AI → Confidence
      ↓
┌──────────────────────┬────────────────────────┐
│  Confident (≥ auto)  │  Mid (≥ suggest only) │
└──────────────────────┴────────────────────────┘
      ↓                          ↓
Post public reply            Add draft/internal note
```

### Node Hints
- Scheduler/Webhook: receive ticket events
- HTTP Request: Helpdesk list/get
- IF: eligibility
- Function: assemble prompt
- OpenAI: chat completion
- HTTP Request: Helpdesk reply or note
- PostgreSQL: `ai_decisions` log

---

## 8. Confidence Policy (for all flows)
- `AUTO_REPLY_THRESHOLD` (e.g., 0.75): above this → auto reply.
- `SUGGEST_ONLY_THRESHOLD` (e.g., 0.55): above this but below auto → draft note.
- Per-queue overrides (complaints/high-priority require higher threshold).
- Never auto-close; respect opt-out flags.
