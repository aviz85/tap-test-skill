# Tap Test - Real API Integration Tests for AI-Era Development

## The Problem: AI-Generated Tests Are Often Useless

When you ask an AI coding agent to "write tests", you typically get:

- **Mock-heavy tests** that test nothing real - they verify your mocks work, not your code
- **Unit tests for internal functions** that pass even when the actual API is broken
- **Snapshot tests** that just freeze current behavior without understanding it
- **Tests that check implementation details** instead of actual user-facing behavior

**The result?** A green test suite that gives you false confidence. Your tests pass, but your API is broken in production because no test ever sent a real HTTP request to a real server with a real database.

> "100% test coverage, 0% confidence." - Every team that relied on mocked integration tests

## The Solution: Tap Tests

A **Tap Test** "taps" into your API exactly like a real client would:

| Traditional AI Tests | Tap Tests |
|---------------------|-----------|
| Mock HTTP layer | **Real HTTP requests** |
| Mock database | **Real database** |
| Call functions directly | **Real server instance** |
| Assert on mock returns | **Assert on actual responses + DB state** |
| Fast but meaningless | **Slightly slower but actually useful** |

### What Makes This Different

```
Traditional test:                    Tap test:

mock(db.query) → return fake data    Client → HTTP POST → Real Server → Real DB
call handler(fakeReq)                         ← HTTP Response ←
assert handler returned something              + verify DB state
                                               + verify side effects
```

## Architecture

```
Test File
  |
  ├── Fastify test server (port 3999)
  │   ├── POST /api/send          → simulate incoming message
  │   ├── GET  /api/state/:id     → read user/session state
  │   ├── GET  /api/chat/:id      → read conversation history
  │   └── DELETE /api/user/:id    → reset user for next test
  │
  ├── EventEmitter capture        → collect outgoing responses
  ├── Direct DB queries           → verify transactions, records
  └── Cleanup helpers             → isolated test data (prefix/marker)
```

## Who Is This For?

Any application with a **server that has an API layer** - which is the most popular architecture in modern development:

- REST APIs (Express, Fastify, Hono, etc.)
- WhatsApp/Telegram bots with webhook handlers
- SaaS backends with CRUD operations
- Microservices with HTTP communication
- Any system where clients talk to your server via HTTP

If your app has routes that handle requests and interact with a database, tap tests are for you.

## Quick Start

### 1. Install the Skill

Copy the `SKILL.md` to your Claude Code skills directory:

```bash
# Personal (available in all projects)
mkdir -p ~/.claude/skills/tap-test
cp SKILL.md ~/.claude/skills/tap-test/SKILL.md

# Or project-level (available only in this project)
mkdir -p .claude/skills/tap-test
cp SKILL.md .claude/skills/tap-test/SKILL.md
```

### 2. Use It

In Claude Code, just say:

```
/tap-test
```

Or describe what you want naturally:

```
"Write real integration tests for my API"
"Create tap tests for the user registration flow"
"Test my WhatsApp bot with real HTTP requests"
```

Claude will automatically:
1. Explore your project structure
2. Find your routes, handlers, and DB client
3. Create a test server that mirrors your API
4. Write tests that send real HTTP requests
5. Verify both responses AND database state
6. Set up proper cleanup for test isolation

## The Pattern

```
1. beforeAll:  cleanup DB → seed test data → start Fastify server
2. beforeEach: reset test users → clear captured responses
3. test:       POST to API → wait for responses → verify HTTP state + DB
4. afterAll:   cleanup DB → stop server
```

## Example Test

```typescript
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import Fastify from 'fastify';

const TEST_PORT = 3999;
const BASE = `http://127.0.0.1:${TEST_PORT}`;
const captured: Response[] = [];

// Helper: send a message like a real client
async function sendText(phone: string, text: string) {
  return fetch(`${BASE}/api/send`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ phone, type: 'text', text }),
  });
}

// Helper: wait for async responses
async function waitForResponses(n: number, timeout = 15000) {
  const start = Date.now();
  while (captured.length < n && Date.now() - start < timeout) {
    await new Promise(r => setTimeout(r, 100));
  }
  return captured;
}

// Helper: check state via HTTP (like a real client would)
async function getState(phone: string) {
  const res = await fetch(`${BASE}/api/state/${phone}`);
  return res.json();
}

describe('User Onboarding Flow', () => {
  beforeAll(async () => {
    await cleanupTestData();
    await seedTestData();
    await startTestServer();
  });

  beforeEach(() => {
    captured.length = 0;
  });

  afterAll(async () => {
    await cleanupTestData();
    await stopTestServer();
  });

  it('should onboard new user via HTTP', async () => {
    await sendText('972599001234', 'hello');
    const responses = await waitForResponses(1);

    // Verify response content
    expect(responses[0].content).toContain('Welcome');

    // Verify state via HTTP
    const state = await getState('972599001234');
    expect(state.session.state).toBe('onboarding');

    // Verify database directly
    const { data } = await supabase
      .from('users')
      .select()
      .eq('phone', '972599001234')
      .single();
    expect(data).not.toBeNull();
    expect(data.created_at).toBeDefined();
  });
});
```

## Key Principles

| Principle | Why |
|-----------|-----|
| **No mocks for core logic** | The whole point is testing the real stack |
| **Mock only external services** | WhatsApp API, LLM providers - things you can't control |
| **Isolated test data** | Use unique prefixes (`97259900%`), cleanup before AND after |
| **Sequential execution** | DB consistency requires `singleFork: true` in Vitest |
| **Generous timeouts** | Real DB operations take time (10-30s per test) |
| **Wait, don't assume** | Use `waitForResponses()` for async processing |

## Vitest Configuration

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    pool: 'forks',
    poolOptions: { forks: { singleFork: true } },
    testTimeout: 30000,
    env: { WHATSAPP_PROVIDER: 'mock' },
  },
});
```

## Test Data Isolation

```typescript
const TEST_PHONE_PREFIX = '97259900';

async function cleanupTestData() {
  // Delete ALL test data across all tables
  await supabase.from('messages').delete().like('phone', `${TEST_PHONE_PREFIX}%`);
  await supabase.from('sessions').delete().like('phone', `${TEST_PHONE_PREFIX}%`);
  await supabase.from('users').delete().like('phone', `${TEST_PHONE_PREFIX}%`);
  // Also clean content test data
  await supabase.from('content').delete().eq('area', 'test');
}

async function seedTestData() {
  await supabase.from('content').insert([
    { area: 'test', title: 'Test Item 1', body: 'Test content' },
    { area: 'test', title: 'Test Item 2', body: 'More test content' },
  ]);
}
```

## What Tests Should Cover

1. **Happy path flows** - Full user journey from start to finish
2. **Error handling** - Invalid input, missing data, edge cases
3. **State transitions** - Verify session state after each step
4. **DB verification** - Transactions created, records updated
5. **Multi-user concurrent** - Parallel requests don't interfere
6. **Chat history** - Conversations recorded correctly

---

## Using Skills in Claude Code & Coding Agents

### What Are Skills?

Skills are modular, filesystem-based capabilities that extend Claude Code's functionality. They package instructions, metadata, and optional resources that Claude uses automatically when relevant to your task.

### How Skills Work

Skills operate on a **progressive disclosure model**:

| Layer | When Loaded | Content |
|-------|------------|---------|
| **Metadata** | Always at startup | Skill name + description (~100 tokens) |
| **Instructions** | When triggered | Full SKILL.md content |
| **Resources** | On-demand | Supporting files, scripts, templates |

This means you can bundle comprehensive documentation and examples without paying context cost for unused content.

### Skill File Structure

```
my-skill/
├── SKILL.md              # Required: instructions + metadata
├── reference.md          # Optional: detailed guidance
├── examples.md           # Optional: sample outputs
├── templates/            # Optional: templates
└── scripts/              # Optional: executable code
```

### SKILL.md Anatomy

```yaml
---
name: my-skill
description: "What it does. When to use it."
disable-model-invocation: false   # true = only /command triggers it
user-invocable: true              # true = user can type /my-skill
allowed-tools: Read, Grep, Bash   # tools the skill can use
---

# Instructions for Claude to follow when skill is active
```

### Installing Skills

```bash
# Personal (all projects)
cp -r my-skill ~/.claude/skills/

# Project-level (one project)
cp -r my-skill .claude/skills/

# As a plugin
claude plugin install https://github.com/user/my-skills
```

### Implementing Skills in Your Coding Agent

If you're building a coding agent (with Claude Code, Agent SDK, or API), skills let you:

1. **Encapsulate domain knowledge** - Package testing strategies, deployment procedures, code review checklists
2. **Standardize workflows** - Every team member's agent follows the same patterns
3. **Share across projects** - Personal skills work everywhere, project skills travel with the repo
4. **Progressive loading** - Only load what's needed, keeping context lean

### Best Practices for Skill Authors

1. **Single Responsibility** - One skill = one focused capability
2. **Clear Descriptions** - Include keywords users would naturally say
3. **Progressive Disclosure** - Core in SKILL.md, details in supporting files
4. **Include Examples** - Show concrete inputs and expected outputs
5. **Test Isolation** - Skills shouldn't depend on global state

---

## Why Tap Tests Matter for AI Development

As AI coding agents write more of our tests, the quality of test instructions matters more than ever. Without clear guidance, AI agents default to:

- Mocking everything (path of least resistance)
- Testing implementation details (fragile, breaks on refactor)
- Generating high coverage with low value (vanity metrics)

**Tap Test as a skill solves this** by giving the AI agent a clear, opinionated framework for writing tests that actually verify your system works.

When you say `/tap-test`, the agent doesn't guess - it follows a proven pattern that produces tests you can trust.

---

## License

MIT

## Author

Created by [Aviz](https://linktr.ee/aviz85) - Building tools for the AI-native developer workflow.
