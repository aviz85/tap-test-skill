---
name: tap-test
description: "Create real API/HTTP integration tests that simulate actual client behavior. Spins up a test server, sends real HTTP requests, captures responses, verifies DB state, cleans up. Use for: api test, integration test, e2e test, tap test, real test, http test, test against real database."
---

# Tap Test - Real API Integration Test Generator

Generate production-grade HTTP-level integration/e2e tests that simulate real client behavior against a live API + real database.

## What is a Tap Test?

A "tap test" taps into your API exactly like a real client would:
- **Real HTTP requests** (not function calls)
- **Real database** (not mocks)
- **Real server** (lightweight Fastify instance)
- **Real responses** captured via EventEmitter or HTTP endpoints
- **Real cleanup** after each test

## Pattern

```
1. beforeAll: cleanup DB → seed test data → start Fastify server
2. beforeEach: reset test users + clear captured responses
3. test: POST to API → wait for responses → verify via HTTP state endpoint + DB queries
4. afterAll: cleanup DB → stop server
```

## Architecture

```
Test File
  ├── Fastify test server (port 3999)
  │   ├── POST /api/send          → simulate incoming message
  │   ├── GET  /api/state/:id     → read user/session state
  │   ├── GET  /api/chat/:id      → read conversation history
  │   └── DELETE /api/user/:id    → reset user for next test
  ├── EventEmitter capture        → collect outgoing responses
  ├── Direct DB queries           → verify transactions, records
  └── Cleanup helpers             → isolated test data (prefix/marker pattern)
```

## Step-by-Step Process

### 1. Explore the project
- Find the main message processing function (router, handler, engine)
- Find the gateway/provider layer (WhatsApp, Slack, etc.)
- Find existing test setup (cleanup, seed functions)
- Identify the DB client and tables

### 2. Create test data isolation
- Use a **phone prefix** or **unique marker** for test data (e.g., `97259900%`)
- Use a **category/area marker** for content data (e.g., `area='test'`)
- Create `cleanupTestData()` that deletes ALL test data across all tables
- Create `seedTestData()` that inserts minimal required test content

### 3. Build the test server
```typescript
import Fastify from 'fastify';

const app = Fastify({ logger: false });
const TEST_PORT = 3999;

// Capture responses via provider's EventEmitter
const captured: Response[] = [];
provider.onResponse((r) => captured.push(r));

// POST endpoint - simulate incoming messages
app.post('/api/send', async (req, reply) => {
  const msg = provider.parseWebhook(req.body);
  await engine.handleMessage(msg);
  return { success: true };
});

// GET endpoint - read state
app.get('/api/state/:phone', async (req) => {
  const user = await getUser(req.params.phone);
  const session = await getSession(user.id);
  return { user, session };
});

// DELETE endpoint - reset user
app.delete('/api/user/:phone', async (req) => {
  // cascade delete all user data
});

await app.listen({ port: TEST_PORT, host: '127.0.0.1' });
```

### 4. Write HTTP helper functions
```typescript
async function sendText(phone: string, text: string) {
  return fetch(`${BASE}/api/send`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ phone, type: 'text', text }),
  });
}

async function waitForResponses(n: number, timeout = 15000) {
  const start = Date.now();
  while (captured.length < n && Date.now() - start < timeout) {
    await new Promise(r => setTimeout(r, 100));
  }
  return captured;
}

async function getState(phone: string) {
  const res = await fetch(`${BASE}/api/state/${phone}`);
  return res.json();
}
```

### 5. Write test cases
Each test should:
- `clearCaptured()` before sending
- Send via HTTP (not direct function call)
- `waitForResponses(n)` for async processing
- Verify response content
- Verify state via HTTP GET
- Verify DB records via direct Supabase queries

### 6. Test categories to cover
- **Happy path flows** (full user journey start to finish)
- **Error handling** (invalid input, missing data)
- **State transitions** (verify session state after each step)
- **DB verification** (transactions, records created)
- **Multi-user concurrent** (parallel requests)
- **Chat history** (conversation recorded correctly)

## Key Principles

- **No mocks for core logic** - the whole point is testing the real stack
- **Mock only external services** (WhatsApp API, LLM) via provider pattern
- **Isolated test data** - unique prefixes, cleanup before AND after
- **Sequential execution** - use `singleFork: true` in vitest for DB consistency
- **Generous timeouts** - real DB operations take time (10-30s per test)
- **Wait, don't assume** - use `waitForResponses()` pattern for async processing

## Vitest Config

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    pool: 'forks',
    poolOptions: { forks: { singleFork: true } },
    env: { WHATSAPP_PROVIDER: 'mock' }, // or equivalent
  },
});
```

## Example Test

```typescript
it('should onboard new user via HTTP', async () => {
  clearCaptured();
  await sendText(PHONE, 'hello');
  const responses = await waitForResponses(1);

  expect(responses[0].content).toContain('Welcome');

  const state = await getState(PHONE);
  expect(state.session.state).toBe('onboarding');
  expect(state.user).not.toBeNull();
});
```
