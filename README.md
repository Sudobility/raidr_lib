# raidr_lib

Business logic for raidr front ends: catalog hooks with session-persistent
search and paging, MCP/skill/site detail hooks, connection snippets for MCP
clients, skill install instructions, and the API playground: a domain
browser, an API inspector with an endpoint flow map, and an endpoint
playground with typed parameter validation and per-host credentials kept in
the browser. When the raidr browser extension is installed, the playground's
sign-in goes through it (`createExtensionBridge`) and the site token is filled
in automatically; otherwise it opens the site's sign-in page in a popup for the
user to copy the token by hand. No UI.

Published to **GitHub Packages** under the `@sudobility` scope. A consumer
needs `@sudobility:registry=https://npm.pkg.github.com` in its `.npmrc` (and a
token with `read:packages` for private installs), then:

```bash
bun add @sudobility/raidr_lib
```

## Development

Bun only. `bun install`, then:

```bash
bun run verify        # lint → typecheck → test:unit → build (the full gate)
bun run check-all     # lint → typecheck → test:unit
bun run test:unit     # Vitest once (= test:run); test:watch for watch mode
bun run typecheck     # tsc --noEmit
bun run lint          # ESLint on src (lint:fix to autofix)
bun run build         # tsc → dist/, then scripts/fix-esm-imports.mjs
bun run dev           # build:watch
```

This is a library: there is no app to run. Peer deps
(`@sudobility/raidr_client`, `@sudobility/raidr_types`, `@sudobility/di`,
`@sudobility/types`, `@tanstack/react-query`, `react`, `zustand`) are supplied
by the consuming app.

```tsx
const { items, search, setSearch, page, setPage } = useMcpCatalog({ networkClient, baseUrl });
const { manifest, tools, connect } = useMcp({ networkClient, baseUrl, apiHost, isAuthenticated, apiKey, siteToken });
connect.claudeCode.cli; // claude mcp add --transport http … --header "X-Raidr-Token: …"
const play = useEndpointPlayground({ networkClient, baseUrl, endpointRef, isAuthenticated });
play.setValue('clip_id', '…'); play.execute(); // play.result: status, headers, body
play.openLogin(); // play.extension, play.loginWindow ('open' → 'captured' | 'closed'), play.tokenVerified
```

Layer: `raidr_types` → `raidr_client` → **`raidr_lib`** → `raidr_app`.
