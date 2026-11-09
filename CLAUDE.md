# CLAUDE.md — raidr_lib

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.

Platform-neutral business logic between `raidr_client` and the UIs. Published as
`@sudobility/raidr_lib` (public, GitHub Packages). Bun only. Mirrors `sudojo_lib`.

> Versions below 0.1.0 of this npm name were an unrelated package, now
> `@sudobility/raidr_processor`. Never publish a 0.0.x here.

## Purpose and layer position

```
raidr_types → raidr_client → raidr_lib (this repo) → raidr_app
```

Hooks that turn `raidr_client` queries into what a page renders (paged catalogs,
detail records classified as loading / found / missing / failed), plus pure
helpers: MCP-client connection snippets, skill install commands, tool
formatting, and the API playground's form model, remembered credentials and
flow-map layout. **No UI components, no styling.**

| Item | Value |
| --- | --- |
| npm name | `@sudobility/raidr_lib`, `publishConfig.access: public`, BUSL-1.1 |
| Version | see `package.json` (`0.1.x`) |
| Entry point | `.` only (`dist/index.js`); `src/index.ts` is the whole public API — nothing else is exported |
| Peer deps | `@sudobility/raidr_client`, `@sudobility/raidr_types`, `@sudobility/di`, `@sudobility/types` (ranges in `package.json`), `@tanstack/react-query` >=5, `react` >=18, `zustand` >=5 |
| Runtime dep | `@dagrejs/dagre` only (flow-map layout); everything else is a peer |
| Consumer | `raidr_app` |

## Commands

All verified passing on 2026-11-08 after `bun install`.

| Command | What it does | Result |
| --- | --- | --- |
| `bun run verify` | lint → typecheck → test:unit → build | exit 0 |
| `bun run check-all` | lint → typecheck → test:unit | exit 0 |
| `bun run quick-check` | lint → typecheck | exit 0 |
| `bun run lint` | ESLint 10 on `src`; `prettier/prettier` is an error | exit 0 |
| `bun run typecheck` | `tsc --noEmit` (TS 6; excludes `*.test.ts`) | exit 0 |
| `bun run test:unit` (= `test:run`) | Vitest 4 once, happy-dom, `src/test/setup.ts` | 10 files, 48 tests pass |
| `bun run test:coverage` | v8 → `coverage/` (gitignored) | ~40% lines, no threshold |
| `bun run build` | `tsc -p tsconfig.build.json` → `dist/` + `fix-esm-imports.mjs` | exit 0 |
| `bun run format:check` / `format` | Prettier on `src/**/*.ts` | check: exit 0 |

`dev` (= `build:watch`), `test:watch`, `lint:fix`, `format` are watch/mutate modes.

## File map

```
src/
├── index.ts                     public API; nothing else is exported
├── hooks/
│   ├── catalog.ts               CATALOG_PAGE_SIZE (20), CatalogResult; internal useCatalogFilter, toCatalogResult
│   ├── useMcpCatalog / useSkillCatalog / useSiteCatalog   useRaidr* + filter ('mcps'|'skills'|'sites'); sites take optional apiHost scope
│   ├── useMcp.ts                public summary; full MCP + tools + connect snippets when signed in
│   ├── useSkill.ts              skill + install commands + hasMcp
│   ├── useSite.ts               site + apiHosts
│   ├── useSkillBySlug.ts        slug → api_host (useRaidrSkillByName) → useSkill
│   ├── useDomains.ts            useSiteCatalog → DomainEntry[] (domain browser)
│   ├── useApiInspector.ts       API summary; doc + grouped endpoints + flow graph when signed in
│   └── useEndpointPlayground.ts one endpoint from an endpointRef: form values/errors, credentials, execute, sign-in
├── stores/catalogFilterStore.ts zustand persist, sessionStorage key 'raidr-catalog-filters'
├── utils/
│   ├── errors.ts                isNotFoundError, detailState
│   ├── connectConfigs.ts        buildConnectConfigs, mcpServerName, shellQuote, *_PLACEHOLDER
│   ├── skillInstall.ts          skillInstallInstructions (one curl), skillMarkdownUrl, skillDirectoryName
│   ├── tools.ts                 toolInputFields, formatToolSignature/Request, isMutatingTool
│   ├── params.ts                paramControl, validateParam, coerceParam, paramPlaceholder, buildExecute
│   ├── schemaForm.ts            formOf, schemaToParam — builds playground form from a v1 or v2 endpoint schema
│   ├── credentials.ts           createCredentialStore, browserStorage, openLoginWindow, watchWindowClosed
│   ├── extensionBridge.ts       createExtensionBridge (detect, requestToken), TokenRequestError
│   ├── endpoints.ts             groupEndpoints (by tag, 'other' last; each with its endpointRef)
│   ├── domains.ts               toDomainEntry (hostname w/o www., sorted apiHosts)
│   ├── flow.ts                  buildFlowGraph, flowPath, findBackEdges + consts (FLOW_TILE, MAX_FLOW_NODES=60, …)
│   └── *.test.ts                tests sit beside the code
└── test/setup.ts                Vitest setup: stubs localStorage, window listeners, matchMedia
```

## Conventions

**Hook shape.** One options object in, one flat result out:
`useX({ networkClient, baseUrl, ...ids }): UseXResult`, with `UseXOptions` /
`UseXResult` exported next to the hook and from `src/index.ts`. Hooks **wrap**
`raidr_client` hooks and add derived data — never fetch directly. Pure
computation lives in `src/utils/<area>.ts` with a `*.test.ts` beside it.

**Catalog hooks** return `CatalogResult<T>` (`items, totalCount, page` 0-based,
`pageCount` >=1, `hasNextPage/hasPreviousPage, search, setSearch, setPage,
isLoading, error, refetch`). They send `q` only when search is non-empty,
`limit: CATALOG_PAGE_SIZE`, `offset: page * CATALOG_PAGE_SIZE`. `setSearch`
resets page to 0. Filters persist per catalog for the browser session (Back
from a detail restores the list).

**Detail hooks** (`useMcp`, `useSkill`, `useSite`) return
`{ ...data, isLoading, notFound, error }` from `detailState` (`utils/errors.ts`):

| Query state | `notFound` | `error` |
| --- | --- | --- |
| disabled (empty id) | true | null |
| loading | false | null |
| error, `status === 404` (`isNotFoundError`) | true | null |
| any other error (status 0 = unreachable, 5xx, …) | false | the Error |
| settled, `data.success !== true` | true | null |
| settled, `data.success === true` | false | null |

`isNotFoundError` duck-types `.status === 404` (what `NetworkError` from
`@sudobility/types` carries). Detail hooks pass `retry: false` so a 404 is
reported immediately. Caching/keys/stale times come from raidr_client.

**Auth.** raidr_lib never handles raidr's own credentials: the caller's
`networkClient` carries the Firebase token. `useMcp`/`useApiInspector`/
`useEndpointPlayground` take `isAuthenticated` and request the gated full
manifest/doc/flow only when signed in; signed out they return the public
summary (`requiresSignIn: true`, `mcp: null`).

**Playground is the exception** — it keeps the *upstream* site's credentials
(user token + application key), not raidr's. `credentials.ts` stores them in
`localStorage` under `raidr:credential:{kind}:{apiHost}` (`kind` = `user` |
`api_key`) while `remember` is on; with no usable storage it remembers nothing
and never throws. Sign-in: if the raidr browser extension is present
(`createExtensionBridge`) it captures the token automatically; otherwise
`openLoginWindow` opens a `popup,width=520,height=760` named `raidr-login` and
`watchWindowClosed` polls `win.closed` (500 ms). Credentials leave the browser
only inside an execute request.

**Playground internals** (detail in code/JSDoc):
- `isMutatingTool`: non-GET, or description starting `mutates:` (case-insensitive).
- `buildConnectConfigs`: snippets for Claude Code CLI + `.mcp.json`, Claude
  Desktop (`npx -y mcp-remote`), Cursor. Headers `Authorization: Bearer <key>`
  and, when `needsSiteToken`, `X-Raidr-Token`. Empty inputs → `*_PLACEHOLDER`.
- `params.ts`/`schemaForm.ts`: form values are always strings, `''` = not sent.
  `buildExecute` validates the whole form, parses the `additionalBody` JSON,
  and attaches only the credential the endpoint's `auth` needs.
- `buildFlowGraph`: dagre LR layout; fan-out of `MIN_GROUP_SIZE` (3)+ leaf
  endpoints collapses to one `group` node; over `MAX_FLOW_NODES` (60) keeps the
  most-connected and reports `hidden`; `findBackEdges` marks cycle edges.
  Sorted inputs → deterministic layout.

## Release

- Family release via `raidr_app/scripts/push_all.sh` in dependency order
  (`raidr_types → raidr_processor → raidr_client → raidr_lib → … → raidr_app`);
  it polls the registry for each new version. **Never run it unasked.**
- Push to `main`/`develop` runs `.github/workflows/ci-cd.yml` → johnqh/workflows
  `unified-cicd.yml` (typecheck, lint, test:unit, build), then **publishes to
  GitHub Packages** (`publish-target: github`) only when this `package.json`
  version is not already published.
- A `raidr_client` change used here must be published before CI can install it:
  bump the peer/dev range only to a version the registry serves.

## Gotchas

- `tsconfig.build.json` sets `removeComments: false` (base sets `true`) so JSDoc
  reaches `dist/*.d.ts`. Keep it.
- Strict base tsconfig (`exactOptionalPropertyTypes`, `noUncheckedIndexedAccess`):
  hooks spread optional fields as `...(token ? { token } : {})`.
- Only one zustand store instance may exist: the app must not bundle two copies
  of raidr_lib (raidr_app's `vite.config.ts` dedupes it).
- `src/test/setup.ts` stubs `localStorage` and is compiled into `dist/test/`
  (build excludes only `*.test.ts`/`*.spec.ts`).
- While raidr_lib is synced into raidr_app by hand, copy
  `node_modules/@dagrejs/{dagre,graphlib}` across too.
- Prettier: single quotes, trailing commas `es5`, width 80, `arrowParens: avoid`.
  ESLint also enforces `prefer-template`, `object-shorthand`. `typecheck` skips
  test files; ESLint parses them via `tsconfig.eslint.json`.
- Installing needs `.npmrc` registry `@sudobility:registry=https://npm.pkg.github.com`.
