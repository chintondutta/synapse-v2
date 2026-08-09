# Synapse

Synapse is a node-based workflow automation platform. Build automations visually as a directed graph — triggers, AI model calls (Gemini, OpenAI, Anthropic), HTTP requests, and more — and Synapse resolves the execution order, runs each node as a durable background job, and streams live status back to the canvas in real time.

> **Status:** early development. Auth, database, and the tRPC/API foundation are in place; the workflow execution engine (topological sorting, executor registry, real-time status propagation) described below is being built out next.

---

## Architecture

Synapse decouples workflow execution from the request/response cycle so that long-running, high-latency steps (AI generations, third-party API calls) never block the UI or hit HTTP timeouts.

- **Durable execution with Inngest** — a tRPC mutation dispatches a `workflows/execute.workflow` event; the actual run happens in a background Inngest function, giving atomicity, retries, and out-of-band side-effect handling.
- **Prisma as ground truth** — each run reconstructs its graph (`Workflow`, `Node`, `Connection`) directly from Postgres rather than trusting volatile client state, then validates it before execution (`NonRetryableError` on missing/malformed data, so broken configs fail fast instead of burning retries).
- **Topological sort over a DAG** — connections are converted into an adjacency matrix to compute execution order, support parallel branches, attach unconnected nodes as self-edges so nothing is silently dropped, and detect cycles (`workflow contains a cycle`) before anything runs.
- **Executor registry pattern** — `Prisma.NodeType` (`manualTrigger`, `httpRequest`, `gemini`, `openAi`, `anthropic`, …) maps to a modular executor function. Every executor implements a shared `nodeExecutorParams` interface (`data`, `nodeId`, `context`, `step`), so new integrations can be added under `features/<type>/executor.ts` without touching the core loop.
- **Shared execution context + Handlebars templating** — node outputs accumulate into a master context object (`{{myNodeName.httpResponse.data.id}}`), keyed by a user-defined `variableName` to prevent collisions. JSON payloads are wrapped via a custom Handlebars `SafeString` helper to avoid HTML-escaping corrupting prompts and API bodies.
- **Real-time status propagation** — `@inngest/realtime` middleware broadcasts each node's `Initial → Loading → Success/Error` lifecycle on a scoped channel; the client subscribes with a `useNodeStatus` hook so users watch their automation run live.

Full design rationale lives in [`Synapse - Technical Document.pdf`](./Synapse%20-%20Technical%20Document.pdf).

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 16 (App Router, React 19, TypeScript) |
| Background jobs | Inngest (durable, event-driven execution) + `@inngest/realtime` |
| Database | PostgreSQL via Prisma ORM |
| API layer | tRPC + TanStack Query |
| Auth | better-auth |
| Templating | Handlebars |
| UI | Radix UI / base-ui, shadcn-generated components, Tailwind CSS, Lucide React |
| Forms | react-hook-form + zod |

## Project Structure

```
src/
├── app/            # Next.js App Router pages, route groups, and API routes
│   ├── (auth)/     # Auth-related pages (login, ...)
│   └── api/trpc/   # tRPC HTTP handler
├── components/ui/  # Shared shadcn/Radix UI primitives
├── features/       # Feature modules (auth, and eventually per-node-type executors)
├── trpc/           # tRPC router, client, and server setup
├── hooks/          # React hooks
└── lib/            # Shared utilities (db client, auth config)
prisma/             # Database schema and migrations
```

## Getting Started

### Prerequisites

- Node.js 20+
- A PostgreSQL database

### Setup

```bash
git clone https://github.com/chintondutta/synapse-v2.git
cd synapse-v2
npm install
# create a .env file with the variables listed below
npx prisma migrate dev
npm run dev
```

The app will be available at `http://localhost:3000`.

### Environment Variables

| Variable | Description |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string |
| `BETTER_AUTH_SECRET` | Secret used to sign/encrypt auth sessions |
| `BETTER_AUTH_URL` | Base URL better-auth uses for callbacks (e.g. `http://localhost:3000`) |

## Scripts

```bash
npm run dev     # start the dev server
npm run build   # production build
npm run start   # run the production build
npm run lint    # eslint
```

## License

MIT
