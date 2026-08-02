# Backend Readiness Implementation Reference

When asked to implement Backend Readiness, focus on structured logging and OpenAPI auto-generation. Both blueprints are **ESM** by default — modern versions of `pino`, `pino-pretty`, `zod`, and `@asteasolutions/zod-to-openapi` are ESM-first and crash with `ERR_REQUIRE_ESM` if `require()`d. If the backend `package.json` is deliberately CJS, use the CJS variants noted beside each file.

## 0. Module-system decision

Ensure `backend/package.json` has `"type": "module"` (or use `.mjs` extensions for the files below). If the user's stack mandates CJS, swap `import`/`export` for `require`/`module.exports` per the noted fallbacks.

## 1. Structured Logging (Pino)

Install Pino:

```bash
cd backend && npm install pino pino-pretty
```

Create `backend/src/common/lib/logger.js`:

```js
import pino from 'pino';

const isProd = process.env.NODE_ENV === 'production';
const level = process.env.LOG_LEVEL || 'info';

const logger = isProd
  ? pino({ level, formatters: { level: (label) => ({ level: label }) } })
  : pino({
      level,
      transport: {
        target: 'pino-pretty',
        options: { colorize: true, translateTime: 'SYS:HH:MM:ss', ignore: 'pid,hostname' },
      },
    });

export default logger;
```

CJS fallback — name the file `logger.cjs` or keep `logger.js` with the CJS body:

```js
const pino = require('pino');
// ... same body, replacing import/exports with require/module.exports
module.exports = isProd ? pino({...}) : pino({...});
```

Graceful hardening: if `pino` is not installed at the time the file is imported (e.g. a partial install), fall back to `console` so the app still boots — then warn the user to install pino.

```js
let pino;
try { ({ default: pino } = await import('pino')); } catch { pino = null; }

const logger = pino
  ? (isProd ? pino({ level }) : pino({ level, transport: { target: 'pino-pretty', options: {...} } }))
  : {
      info: (...a) => console.log('[INFO]', ...a),
      warn: (...a) => console.warn('[WARN]', ...a),
      error: (...a) => console.error('[ERROR]', ...a),
      debug: (...a) => console.debug('[DEBUG]', ...a),
      child: () => logger,
    };

export default logger;
```

Append to `backend/.env.example` (idempotent — only append if `LOG_LEVEL` isn't already there):

```env
# Logging
LOG_LEVEL=info
```

## 2. OpenAPI Auto-generation

Install dependencies:

```bash
cd backend && npm install @asteasolutions/zod-to-openapi js-yaml
```

Create `backend/scripts/generate-openapi.js` (ESM). It converts existing Zod schemas into an OpenAPI YAML file. **Register every application Zod schema in the marked block** — an empty registry produces an empty spec, which is a silent failure mode.

```js
#!/usr/bin/env node
import { z } from 'zod';
import { extendZodWithOpenApi, OpenApiGeneratorV31, OpenAPIRegistry } from '@asteasolutions/zod-to-openapi';
import yaml from 'js-yaml';
import { writeFileSync, mkdirSync } from 'node:fs';
import { resolve, dirname } from 'node:path';
import { fileURLToPath } from 'node:url';

const __dirname = dirname(fileURLToPath(import.meta.url));

extendZodWithOpenApi(z);
const registry = new OpenAPIRegistry();

// --- Register your application's Zod schemas here ---------------------------
// Example: registry.register('User', userSchema.openapi({ description: 'User schema' }));
// Scan backend/src for exported schemas and register each by name.
// ---------------------------------------------------------------------------

const generator = new OpenApiGeneratorV31(registry.definitions);
const document = generator.generateDocument({
  openapi: '3.1.0',
  info: { title: 'API Documentation', version: '1.0.0' },
  servers: [{ url: process.env.API_URL || 'http://localhost:9000', description: 'Local' }],
});

const outPath = resolve(__dirname, '../../docs/openapi.yaml');
mkdirSync(dirname(outPath), { recursive: true });
writeFileSync(outPath, yaml.dump(document, { indent: 2, lineWidth: 120 }));
console.log(`✅  docs/openapi.yaml written (${registry.definitions.length} definitions)`);
```

CJS fallback: rename to `generate-openapi.cjs` and swap `import`→`require`, `import.meta.url`→`__dirname` (the built-in CJS global). Body logic identical.

Registration discipline — to find unregistered schemas, after the script runs:

```bash
rg -l "^export const .*Schema = z\." backend/src | sort > /tmp/declared.txt
rg -o "registry\.register\('(\w+)'" backend/scripts/generate-openapi.js | sort -u > /tmp/registered.txt
diff <(cut -d: -f2 /tmp/declared.txt) /tmp/registered.txt
```

Anything in declared but not in registered is missing from the spec — add it and re-run. **Completion criterion:** `diff` returns empty (declared == registered).

Add the script to `backend/package.json`:

```json
{
  "scripts": {
    "openapi": "node scripts/generate-openapi.js"
  }
}
```

Run it and confirm the output file exists and has at least one path:

```bash
npm run openapi && test -s docs/openapi.yaml && grep -q "paths:" docs/openapi.yaml
```

If `paths:` is missing or empty, the registry was empty — the script succeeded but produced a useless spec. Fail the verify step and flag to the user.