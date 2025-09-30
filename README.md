# multichannel-helpdesk-automation

This POC provides automated helpdesk support for WhatsApp and a dedicated Helpdesk system. It ingests WhatsApp messages into a central database, auto-responds via AI when confident, otherwise creates a Helpdesk ticket and returns a reference ID on WhatsApp. It also ingests tickets created directly in the Helpdesk and auto-replies on the ticket when AI is confident. The design remains extensible to other channels.

## High-Level Architecture

```
┌────────────────┐     ┌──────────────┐     ┌─────────────────┐
│   WhatsApp     │────▶│     n8n      │────▶│   AI Engine     │
│  (Inbound/Out) │     │  Workflows   │     │  (RAG + LLM)    │
└────────────────┘     └──────────────┘     └─────────────────┘
         ▲                       │                     │
         │                       ▼                     │
         │             ┌─────────────────┐             │
         │             │  Helpdesk API   │◀────────────┘
         │             │ (Create/Fetch)  │
         │             └─────────────────┘
         │                       │
         └───────────────────────┼─────────────────────────────────
                                 ▼
┌─────────────────────────────────────────────────────────────┐
│                PostgreSQL + pgvector                        │
│ • Messages, Conversations, AI Decisions                     │
│ • Knowledge Base (with embeddings)                          │
│ • Vector Similarity Search                                  │
│ • User Management & Analytics                               │
└─────────────────────────────────────────────────────────────┘
```

## Technology Stack

### Core Platform
- **n8n**: Workflow automation and channel integrations
- **Docker**: Containerized deployment
- **Nginx**: Reverse proxy and load balancing

### Data Storage
- **PostgreSQL 15+**: Primary database with pgvector extension
- **Redis**: Caching and session management
- **MinIO**: File storage for attachments

### AI & RAG
- **OpenAI GPT-4**: Language model for responses
- **text-embedding-ada-002**: Embedding model
- **pgvector**: Vector similarity search
- **LangChain**: RAG pipeline orchestration

### Frontend
- **React + TypeScript**: Admin dashboard (optional in POC)
- **Tailwind CSS**: Styling
- **Chart.js**: Analytics visualization
