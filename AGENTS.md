# AGENTS.md

## Collaboration preferences

- Reply in Chinese; code comments in English, write *why* not *how*.
- Be concise and direct; no unnecessary summaries or explanations. Write code
  directly unless there's high risk or missing information.
- Favor functional style; avoid OOP in TS/JS. Prefer reusing or refactoring
  existing code over adding new abstractions.
- Follow KISS and DRY; break down problems from first principles; watch for
  XY problems.
- Immediately flag unreasonable requirements or wrong directions.

## Project overview

Sublink Worker is a multi-platform proxy subscription converter: it transforms
protocol links (ShadowSocks/VMess/VLESS/Hysteria2/Trojan/TUIC) into
client-specific configs (Sing-Box/Clash/Xray/Surge). Same codebase runs on
Cloudflare Workers / Node.js / Vercel / Docker.

Tech stack: Hono (Web + JSX SSR) + Vitest + Wrangler + esbuild + ioredis.

## Supported protocols & formats

| Input | Output |
|---|---|
| ss://, vmess://, vless://, trojan://, hysteria2://, tuic:// | Sing-Box JSON |
| Base64 subscription strings | Clash YAML |
| HTTP/HTTPS subscription URLs | Surge INI |
| Sing-Box JSON / Clash YAML / Surge INI configs | Xray/Base64 (passthrough) |
| Full configs with DNS, rules, etc. (merged on top of base) | Subconverter INI |

17 predefined rule presets (Ad Block, AI, Bilibili, YouTube, Google,
Telegram, GitHub, Microsoft, Apple, Social Media, Streaming, Gaming, etc.)
with min/balanced/comprehensive profiles.

## Multi-runtime architecture

Entry points: `src/worker.jsx` (CF Workers), `src/platforms/node-server.js`
(Node/Docker), `api/index.js` (Vercel). All call `createApp(runtime)` from
`src/app/createApp.jsx` to build the same Hono app.

- `src/runtime/{cloudflare,node,vercel}.js` — platform adapters
- `src/runtime/runtimeConfig.js` — normalizes KV, asset fetcher, logger, env defaults
- KV adapters in `src/adapters/kv/`: CloudflareKV, RedisKV (ioredis), UpstashKV (REST), MemoryKV
- Node/Vercel KV priority: Redis > Upstash > Memory (`DISABLE_MEMORY_KV=true` to disable fallback)
- ShortLinkService and ConfigStorageService (30-day TTL)

Env vars: `REDIS_URL`, `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`,
`REDIS_TLS`, `REDIS_KEY_PREFIX`, `KV_REST_API_URL`, `KV_REST_API_TOKEN`,
`CONFIG_TTL_SECONDS`, `SHORT_LINK_TTL_SECONDS`, `STATIC_DIR`, `PORT`.

## Config generation pipeline

```
Input (URL + query params)
  → BaseConfigBuilder.build()
    → parseCustomItems()
      → Try direct parse (JSON/YAML/INI)
      → Try Base64 decode + parse
      → Per line:
        → HTTP URL → fetchSubscriptionWithFormat()
                     → detectFormat() → parseSubscriptionContent()
                     → extract proxies + config overrides
        → Protocol URI → ProxyParser.parse()
                         → scheme dispatcher → protocol parser
                         → returns { tag, type, server, server_port, ... }
    → addCustomItems() — convert proxies to target format
    → addSelectors() — auto-select, node-select, country groups,
                       outbound groups, custom rule groups, fallback
    → formatConfig() — output YAML/JSON/INI/Base64
```

### Adding a new protocol

1. Add parser in `src/parsers/protocols/<name>Parser.js`
2. Register in `src/parsers/ProxyParser.js` protocolParsers map
3. Add converter logic in each builder (Clash/Singbox/Surge)
4. Add tests in `test/`

## Builders (Template Method pattern)

All extend `BaseConfigBuilder`. Must implement:

- `getProxies()`, `getProxyName(proxy)`, `convertProxy(proxy)`
- `addProxyToConfig(proxy)`, `addAutoSelectGroup(list)`
- `addNodeSelectGroup(list)`, `addOutboundGroups(outbounds, list)`
- `addCustomRuleGroups(list)`, `addFallBackGroup(list)`
- `addCountryGroups()`, `formatConfig()`

Country grouping via `src/utils.js#groupProxiesByCountry` (31 countries,
regex matching with word boundaries).

## Deployment

| Platform | Entry | Command |
|---|---|---|
| Cloudflare Workers | `src/worker.jsx` | `npm run deploy` (runs setup-kv + wrangler deploy) |
| Vercel | `api/index.js` | `npm run build` (esbuild) |
| Node.js | `node-server.js` | `npm run build:node && node dist/node-server.cjs` |
| Docker | `Dockerfile` (GHCR) | `docker compose up` (includes Redis) |

## Testing

Vitest + `@cloudflare/vitest-pool-workers`:

```bash
npm test                    # All tests
npx vitest test/<file>.test.js   # Single file
npx vitest -t "<pattern>"        # Match test name
```

Tests use `MemoryKVAdapter` and `createApp(runtime)` for the Hono app.

## i18n

zh-CN / en-US / fa-IR / ru in `src/i18n/index.js`. Language detection from
`lang` query param or `Accept-Language` header.

## Key conventions

- `.jsx` files use Hono JSX runtime, **not React**
- Base64 input: use `tryDecodeSubscriptionLines()`
- Errors: `ServiceError` subclasses (`InvalidPayloadError` etc.)
- No ESLint/Prettier/Biome — no auto-formatting
