# Vercel AI Chatbot (Chat SDK)

> Open-source AI chatbot template built with Next.js 16, AI SDK 6.0, and the Vercel AI Gateway.

---

## Tech Stack

```yaml
Framework: Next.js 16.0.10 (App Router)
Runtime: React 19.0.1
Language: TypeScript 5.6.3
Package Manager: Bun 1.3.6
Styling: Tailwind CSS v4 + shadcn/ui (new-york style)
Database: PostgreSQL (Drizzle ORM)
Auth: Auth.js v5 (next-auth 5.0.0-beta.25)
AI: Vercel AI SDK 6.0 + AI Gateway
Storage: Vercel Blob
Cache: Redis (resumable streams)
```

---

## Architecture

```
apps/ai-chatbot/
├── app/
│   ├── (auth)/              # Auth routes (login, register)
│   │   └── auth.config.ts   # NextAuth configuration
│   └── (chat)/              # Chat application
│       ├── api/             # API routes
│       │   ├── chat/        # Chat streaming endpoint
│       │   ├── document/    # Document CRUD
│       │   ├── files/       # File uploads
│       │   ├── history/     # Chat history
│       │   ├── suggestions/ # AI suggestions
│       │   └── vote/        # Message voting
│       ├── chat/[id]/       # Dynamic chat pages
│       └── page.tsx         # Home page
├── components/              # 43 UI components
│   ├── ai-elements/         # AI-specific components (28)
│   ├── elements/            # Base elements (15)
│   ├── ui/                  # shadcn/ui components (25)
│   ├── chat.tsx             # Main chat component
│   ├── artifact.tsx         # Document artifact viewer
│   ├── message.tsx          # Message renderer
│   └── multimodal-input.tsx # Rich input component
├── hooks/                   # Custom React hooks (6)
├── lib/
│   ├── ai/                  # AI configuration
│   │   ├── models.ts        # Multi-provider models
│   │   ├── prompts.ts       # System prompts
│   │   ├── providers.ts     # AI Gateway setup
│   │   └── tools/           # AI tools (4)
│   └── db/                  # Database layer
│       ├── schema.ts        # Drizzle schema
│       ├── queries.ts       # Query functions
│       └── migrations/      # SQL migrations
├── artifacts/               # Document types
│   ├── code/                # Code editor
│   ├── image/               # Image editor
│   ├── sheet/               # Spreadsheet
│   └── text/                # Text editor
└── tests/                   # Playwright E2E tests
```

---

## Key Patterns

### Multi-Provider AI Gateway

```typescript
// lib/ai/models.ts - Curated model list
export const DEFAULT_CHAT_MODEL = "google/gemini-2.5-flash-lite";

export const chatModels: ChatModel[] = [
  // Anthropic
  { id: "anthropic/claude-haiku-4.5", provider: "anthropic" },
  { id: "anthropic/claude-sonnet-4.5", provider: "anthropic" },
  // OpenAI
  { id: "openai/gpt-4.1-mini", provider: "openai" },
  { id: "openai/gpt-5.2", provider: "openai" },
  // Google
  { id: "google/gemini-2.5-flash-lite", provider: "google" },
  // xAI (Grok)
  { id: "xai/grok-4.1-fast-non-reasoning", provider: "xai" },
  // Reasoning models
  { id: "anthropic/claude-3.7-sonnet-thinking", provider: "reasoning" },
];
```

### Chat with useChat Hook

```typescript
// components/chat.tsx
const {
  messages,
  setMessages,
  sendMessage,
  status,
  stop,
  regenerate,
  resumeStream,
  addToolApprovalResponse,
} = useChat<ChatMessage>({
  id,
  messages: initialMessages,
  generateId: generateUUID,
  transport: new DefaultChatTransport({
    api: "/api/chat",
    fetch: fetchWithErrorHandlers,
    prepareSendMessagesRequest(request) {
      return {
        body: {
          id: request.id,
          message: lastMessage,
          selectedChatModel,
          selectedVisibilityType,
        },
      };
    },
  }),
});
```

### API Route with AI Tools

```typescript
// app/(chat)/api/chat/route.ts
const result = streamText({
  model: getLanguageModel(selectedChatModel),
  system: systemPrompt({ selectedChatModel, requestHints }),
  messages: modelMessages,
  stopWhen: stepCountIs(5),
  experimental_activeTools: [
    "getWeather",
    "createDocument",
    "updateDocument",
    "requestSuggestions",
  ],
  providerOptions: isReasoningModel
    ? { anthropic: { thinking: { type: "enabled", budgetTokens: 10_000 } } }
    : undefined,
  tools: {
    getWeather,
    createDocument: createDocument({ session, dataStream }),
    updateDocument: updateDocument({ session, dataStream }),
    requestSuggestions: requestSuggestions({ session, dataStream }),
  },
});
```

### Resumable Streams (Redis)

```typescript
// Supports resumable streaming with Redis
if (process.env.REDIS_URL) {
  const streamContext = getStreamContext();
  if (streamContext) {
    const streamId = generateId();
    await createStreamId({ streamId, chatId: id });
    await streamContext.createNewResumableStream(
      streamId,
      () => sseStream
    );
  }
}
```

---

## Database Schema

```typescript
// lib/db/schema.ts - Drizzle ORM
export const user = pgTable("User", {
  id: uuid("id").primaryKey().notNull().defaultRandom(),
  email: varchar("email", { length: 64 }).notNull(),
  password: varchar("password", { length: 64 }),
});

export const chat = pgTable("Chat", {
  id: uuid("id").primaryKey().notNull().defaultRandom(),
  createdAt: timestamp("createdAt").notNull(),
  title: text("title").notNull(),
  userId: uuid("userId").references(() => user.id),
  visibility: varchar("visibility", { enum: ["public", "private"] }).default("private"),
});

export const message = pgTable("Message_v2", {
  id: uuid("id").primaryKey().notNull().defaultRandom(),
  chatId: uuid("chatId").references(() => chat.id),
  role: varchar("role").notNull(),
  parts: json("parts").notNull(),          // Multi-part message format
  attachments: json("attachments").notNull(),
  createdAt: timestamp("createdAt").notNull(),
});

export const document = pgTable("Document", {
  id: uuid("id").notNull().defaultRandom(),
  createdAt: timestamp("createdAt").notNull(),
  title: text("title").notNull(),
  content: text("content"),
  kind: varchar("text", { enum: ["text", "code", "image", "sheet"] }).default("text"),
  userId: uuid("userId").references(() => user.id),
}, (table) => ({
  pk: primaryKey({ columns: [table.id, table.createdAt] }),
}));
```

---

## AI Tools

| Tool | Purpose |
|------|---------|
| `createDocument` | Create text/code/image/sheet artifacts |
| `updateDocument` | Modify existing documents |
| `requestSuggestions` | Get AI suggestions for content |
| `getWeather` | Weather information tool |

---

## Artifacts System

4 artifact types with dedicated editors:

| Kind | Editor | Features |
|------|--------|----------|
| `text` | ProseMirror | Rich text editing |
| `code` | CodeMirror | Python execution, syntax highlighting |
| `image` | Canvas | Basic image editing |
| `sheet` | React Data Grid | Spreadsheet with CSV |

---

## Environment Variables

```bash
AUTH_SECRET=****              # Auth.js secret
AI_GATEWAY_API_KEY=****       # Vercel AI Gateway (non-Vercel deployments)
BLOB_READ_WRITE_TOKEN=****    # Vercel Blob storage
POSTGRES_URL=****             # PostgreSQL connection
REDIS_URL=****                # Redis for resumable streams
```

---

## Development Commands

```bash
bun install          # Install dependencies
bun dev              # Start dev server (--turbo)
bun build            # Run migrations + build
bun db:migrate       # Apply database migrations
bun db:generate      # Generate migration files
bun db:studio        # Drizzle Studio GUI
bun lint             # Biome linting
bun format           # Biome formatting
bun test             # Playwright E2E tests
```

---

## Key Features

- **Multi-provider AI**: Anthropic, OpenAI, Google, xAI via Vercel AI Gateway
- **Reasoning models**: Extended thinking with `thinking` mode
- **Artifacts**: Real-time document editing (text, code, image, spreadsheet)
- **Resumable streams**: Redis-backed stream recovery
- **Geolocation**: Request hints for contextual responses
- **Rate limiting**: Per-user message limits with entitlements
- **Public/Private chats**: Visibility controls
- **Message voting**: Upvote/downvote feedback
- **Auto-title generation**: AI-generated chat titles

---

## Code Conventions

- **Biome**: Linting + formatting
- **File naming**: kebab-case for files, PascalCase for components
- **Imports**: Absolute with `@/` alias
- **Error handling**: Custom `ChatSDKError` class
- **Testing**: Playwright for E2E, Jest-compatible patterns
