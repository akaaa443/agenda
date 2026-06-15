# AgendaTS – Copilot Instructions

## Commands

```bash
npm run build          # Compile TypeScript to dist/
npm run test           # Run all tests (mocha)
npm run lint           # ESLint (src only)
npm run lint-fix       # ESLint with auto-fix
npm run mocha-coverage # Run tests with nyc coverage

# Run a single test file
npx mocha --reporter spec -b test/job.test.ts

# Run tests matching a description
npx mocha --reporter spec -b --grep "Agenda.define"

# Debug logging
DEBUG=agenda:**,-agenda:internal:** npm run mocha   # external logs
DEBUG=agenda:**                      npm run mocha   # all logs
```

## Architecture

The library is a MongoDB-backed job scheduler. Data flows through five main classes:

- **`Agenda`** (`src/index.ts`) — Public API entry point. Extends `EventEmitter`. Owns `definitions` (registered job processors) and delegates persistence to `JobDbRepository` and execution to `JobProcessor`.
- **`Job`** (`src/Job.ts`) — Represents one job instance. All state lives in `job.attrs` (`IJobParameters`). Supports forked child-process execution via `job.forkMode(true)`.
- **`JobDbRepository`** (`src/JobDbRepository.ts`) — MongoDB abstraction. Accepts either a connection string (`db.address`) or an existing `Db` instance (`mongo`). Default collection is `agendaJobs`.
- **`JobProcessor`** (`src/JobProcessor.ts`) — Polls MongoDB on an interval (`processEvery`), locks jobs, enforces global and per-type concurrency limits, and dispatches `processJob` events.
- **`JobProcessingQueue`** (`src/JobProcessingQueue.ts`) — In-memory priority queue. Jobs are sorted right-to-left by ascending `nextRunAt`, then descending `priority`; `pop()` returns the next job to execute.

### Job types
- `'normal'` — standard one-off or scheduled job; each save inserts a new document.
- `'single'` — idempotent; used by `.every()`. If a document with the same name already exists, it is updated rather than inserted.

### define() signature (breaking change from agenda.js)
Options are the **third** parameter, not the second:
```ts
agenda.define('jobName', processorFn, { priority: 'high', concurrency: 2 });
```

## Key Conventions

### TypeScript
- Strict mode is on, but `noImplicitAny` is currently disabled (marked for removal in comments).
- Target: CommonJS / ES2019. Output to `dist/`. Source in `src/`.
- All public types exported from `src/index.ts`.

### Formatting (Prettier)
- Single quotes, tabs (not spaces), print width 100, no trailing commas, no arrow-function parens when avoidable.

### Logging
Every file creates its own namespaced logger via the `debug` package:
```ts
const log = debug('agenda:mymodule');
```

### Testing
- Framework: Mocha + Chai + Sinon.
- In-memory MongoDB via `mongodb-memory-server` — no external DB needed.
- Test helper at `test/helpers/mock-mongodb.ts` exports `mockMongo()`.
- Timeout is 25 s per test (configured in `.mocharc.jsonc`).

### Priorities
Named priorities map to numbers (`lowest: -20`, `low: -10`, `normal: 0`, `high: 10`, `highest: 20`). Numeric values are also accepted directly.

### MongoDB index
`ensureIndex` is **off by default**. Enable it explicitly or create the index manually:
```js
db.agendaJobs.ensureIndex(
  { name: 1, nextRunAt: 1, priority: -1, lockedAt: 1, disabled: 1 },
  "findAndLockNextJobIndex"
);
```
